# Network Error Handling

Every network call returns a typed result. This skill uses a sealed `NetworkResult<T>` produced by a `safeRequest` wrapper, paired with `expectSuccess = false` from [client-setup.md](client-setup.md). Kotlin's stdlib `Result<T>` is a lighter alternative when the UI never branches on error kind; everything below assumes `NetworkResult`.

## NetworkResult

```kotlin
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()

    sealed class Failure : NetworkResult<Nothing>() {
        data class HttpError(val code: Int, val message: String?, val serverMessage: String? = null) : Failure()
        data class NetworkError(val message: String? = null) : Failure()
        data class Timeout(val message: String? = null) : Failure()
        data class Unauthorized(val serverMessage: String? = null) : Failure()
        data class SerializationError(val message: String? = null) : Failure()
        data class Unknown(val throwable: Throwable) : Failure()
    }
}

inline fun <T, R> NetworkResult<T>.map(transform: (T) -> R): NetworkResult<R> = when (this) {
    is NetworkResult.Success -> NetworkResult.Success(transform(data))
    is NetworkResult.Failure -> this
}

inline fun <T, R> NetworkResult<T>.fold(
    onSuccess: (T) -> R,
    onFailure: (NetworkResult.Failure) -> R,
): R = when (this) {
    is NetworkResult.Success -> onSuccess(data)
    is NetworkResult.Failure -> onFailure(this)
}

fun <T> NetworkResult<T>.getOrNull(): T? = (this as? NetworkResult.Success)?.data
```

## safeRequest

A `safeRequest` extension centralizes error handling so repositories stay focused on data mapping (see [data-layer.md](data-layer.md)). It relies on `expectSuccess = false` so it inspects `response.status` instead of catching Ktor's response exceptions:

```kotlin
suspend inline fun <reified T> HttpClient.safeRequest(
    block: HttpRequestBuilder.() -> Unit,
): NetworkResult<T> = try {
    val response = request { block() }
    when (response.status.value) {
        in 200..299 -> NetworkResult.Success(response.body<T>())
        else -> classifyStatus(response.status.value, tryParseError(response))
    }
} catch (e: CancellationException) {
    throw e                       // never swallow — breaks structured concurrency
} catch (e: Exception) {
    classifyException(e)
}
```

Note: for `204 No Content`, call `safeRequest<Unit> { … }`.

## safeCall

Not every failure path goes through `HttpClient` directly. A repository that
calls into an API service (see [data-layer.md](data-layer.md)) instead of
building a request itself needs the same classification, but `safeRequest`
can't wrap it — `safeRequest` is an `HttpClient` extension and the service
call already returns a decoded DTO, not a `HttpResponse`. `safeCall` is the
block-form sibling: it runs any suspending lambda through the same
`classifyException` path, with no `HttpClient` receiver.

```kotlin
suspend inline fun <T> safeCall(block: suspend () -> T): NetworkResult<T> = try {
    NetworkResult.Success(block())
} catch (e: CancellationException) {
    throw e
} catch (e: Exception) {
    classifyException(e)
}
```

`safeCall` only classifies *thrown* exceptions — it never inspects a status
code, because it never sees an `HttpResponse`. Status-code classification
(`classifyStatus`, with its per-code messages for 403/404/429) only happens
inside `safeRequest`. For `safeCall` to turn a failed call into a typed
`Failure` — instead of quietly decoding an error body as if it were the
success DTO — the wrapped call must actually throw on failure. Ktor lets a
specific request opt into `expectSuccess = true` even when the shared
client's default is `false` (per-request override, see
[Ktor's response-validation guide](https://ktor.io/docs/client-response-validation.html)),
so a non-2xx response raises `ClientRequestException` /
`ServerResponseException` for `classifyException` to catch; alternatively the
service can check `response.status` itself and throw before returning.
Without one of those two, a non-2xx response that happens to deserialize
into `T` reaches `safeCall` as an ordinary — and wrong — success.

Prefer `safeRequest` when a repository builds requests directly against
`HttpClient` and needs status-code granularity. Prefer `safeCall` only when
the service layer it wraps already throws on failure.

Server error envelope parsing, which must never itself throw:

```kotlin
@Serializable
data class ErrorDto(
    val message: String? = null,
    val error: String? = null,
    val detail: String? = null,
) {
    val displayMessage: String? get() = message ?: error ?: detail
}

suspend fun tryParseError(response: HttpResponse): String? = try {
    response.body<ErrorDto>().displayMessage
} catch (e: CancellationException) {
    throw e                       // never swallow — breaks structured concurrency
} catch (e: Exception) {
    null
}
```

## Exception Classification

`classifyException` maps thrown exceptions (transport failures, timeouts, serialization errors) to a `Failure`; `classifyStatus` maps an HTTP status code from a response `safeRequest` already received into a `Failure`. Because this code runs in `commonMain`, every matched exception type must exist on every target: `kotlinx.io.IOException` is multiplatform (JVM typealiases it to `java.io.IOException`; Native, JS, and Wasm get their own actuals), whereas JVM-only types such as `java.nio.channels.UnresolvedAddressException` cannot appear here — the generic `IOException` branch below already covers DNS and connection failures on every platform.

```kotlin
import kotlinx.io.IOException // multiplatform — see the portability rule above

fun classifyException(e: Exception): NetworkResult.Failure = when (e) {
    is HttpRequestTimeoutException,
    is ConnectTimeoutException,
    is SocketTimeoutException,
    -> NetworkResult.Failure.Timeout("Request timed out")

    is IOException,
    -> NetworkResult.Failure.NetworkError("No internet connection")

    is SerializationException,   // covers MissingFieldException too — it extends SerializationException
    is JsonConvertException,
    -> NetworkResult.Failure.SerializationError("Invalid response format")

    // Defensive: with expectSuccess = false (this skill's default), safeRequest never
    // throws these — a non-2xx response comes back as a normal HttpResponse and is
    // classified by classifyStatus below instead. These two branches only fire when a
    // specific request opts into expectSuccess = true and is wrapped with safeCall
    // instead of safeRequest (see the safeCall section above) — kept here so that path
    // is still classified instead of falling through to Unknown.
    is ClientRequestException -> when (e.response.status.value) {
        401 -> NetworkResult.Failure.Unauthorized()
        else -> NetworkResult.Failure.HttpError(e.response.status.value, "Request failed")
    }

    is ServerResponseException -> NetworkResult.Failure.HttpError(
        e.response.status.value, "Server error",
    )

    else -> NetworkResult.Failure.Unknown(e)
}

fun classifyStatus(code: Int, serverMessage: String? = null): NetworkResult.Failure = when (code) {
    401 -> NetworkResult.Failure.Unauthorized(serverMessage)
    403 -> NetworkResult.Failure.HttpError(code, "Access denied", serverMessage)
    404 -> NetworkResult.Failure.HttpError(code, "Not found", serverMessage)
    429 -> NetworkResult.Failure.HttpError(code, "Too many requests", serverMessage)
    in 400..599 -> NetworkResult.Failure.HttpError(code, if (code < 500) "Request failed" else "Server error", serverMessage)
    else -> NetworkResult.Failure.HttpError(code, "Unexpected error", serverMessage)
}
```

`CancellationException` must always be re-thrown — never swallow it. It breaks structured concurrency. Ktor exception types stop here — nothing above the data layer sees `ClientRequestException`.

## Plugin Composition

### What goes where

| Concern | Where | Why |
|---|---|---|
| Base URL, content type, static headers | `defaultRequest {}` | Runs per-request, reads live state |
| JSON parsing | `ContentNegotiation` | Core plugin |
| Timeouts | `HttpTimeout` | Default for every project |
| Logging | `Logging` | Debug aid — sanitize `Authorization` in production |
| Token load and refresh | `Auth` plugin | Built-in retry cycle — see [auth-realtime.md](auth-realtime.md) |
| Retry on server errors | `HttpRequestRetry` | Add when the API has transient failures worth retrying |
| Compression | `ContentEncoding` | Add for bandwidth-sensitive APIs |

### Plugin install order

Install order matters — plugins execute in installation order for requests, reverse order for responses.

```
ContentNegotiation → Auth → HttpRequestRetry → HttpTimeout → ContentEncoding
```

Install `HttpRequestRetry` before `HttpTimeout` so timeouts are retryable. Keep `Auth`'s 401 handling separate from `HttpRequestRetry` — the two concerns should not be conflated.

### Custom Client Plugins

*Advanced — use when built-in plugins don't cover the need.*

Build reusable interceptors with `createClientPlugin` for analytics, header injection, or response logging:

```kotlin
val ApiKeyPlugin = createClientPlugin("ApiKeyPlugin", ::ApiKeyConfig) {
    val apiKey = pluginConfig.apiKey

    onRequest { request, _ ->
        request.headers.append("X-Api-Key", apiKey)
    }
}

class ApiKeyConfig {
    var apiKey: String = ""
}

val client = HttpClient(engine) {
    install(ApiKeyPlugin) {
        apiKey = "my-secret-key"
    }
}
```

For global response observation (analytics, session expiry), use `onResponse` in a similar plugin without changing error handling.

## Logging

| Concern | Debug | Production |
|---|---|---|
| Ktor `Logging` plugin | `LogLevel.BODY` | `LogLevel.HEADERS` or not installed |
| `sanitizeHeader` | Optional | Required for `Authorization` |

## Anti-Patterns

| Anti-pattern | Why it hurts | Better approach |
|---|---|---|
| `HttpClient` per request | Connection pool waste, resource leaks | Shared singleton via DI |
| Swallowing `CancellationException` | Breaks structured concurrency, coroutine never cancels | Re-throw explicitly |
| Logging request bodies in production | Leaks sensitive data (tokens, PII) | `LogLevel.HEADERS` or off; `sanitizeHeader` for auth |
| Mixing `expectSuccess = true` with manual status inspection | `ClientRequestException` thrown before you inspect status | Pick one: `expectSuccess = true` + catch exceptions, or `false` + check `response.status` |
| Random plugin install order | Retries fire before timeout, auth conflicts with retry | Follow documented composition order |

# Auth, WebSockets, and SSE

Bearer tokens with automatic refresh, plus the two realtime transports. Client construction is in [client-setup.md](client-setup.md).

Official docs: https://ktor.io/docs/client-bearer-auth.html , https://ktor.io/docs/client-websockets.html , https://ktor.io/docs/client-server-sent-events.html

## Bearer Token Auth

Use Ktor's `Auth` plugin with `bearer` for token management. The plugin loads cached tokens, attaches them to requests, and refreshes on 401 automatically.

`createAuthenticatedClient` below takes the app's single client — `createHttpClient(engine, baseUrl)` from [client-setup.md](client-setup.md), which already installs `ContentNegotiation`, `HttpTimeout`, `Logging` (with `sanitizeHeader`), `defaultRequest`, and `expectSuccess = false` — and derives a second handle from it via Ktor's `HttpClient.config {}`, which copies the receiver's existing configuration and layers on whatever the block installs. This function only adds the `Auth` plugin; it does not build a second, independently-configured client, so the ONE-client HARD-GATE still holds — there is one underlying engine and one base configuration, `Auth` is just an additional plugin on top of it.

```kotlin
fun createAuthenticatedClient(
    baseClient: HttpClient,
    baseUrl: String,
    tokenStorage: TokenStorage,
    onSessionExpired: () -> Unit,
): HttpClient {
    val apiHost = Url(baseUrl).host

    return baseClient.config {
        install(Auth) {
            bearer {
                loadTokens {
                    val tokens = tokenStorage.getTokens()
                    if (tokens.accessToken.isBlank()) return@loadTokens null
                    BearerTokens(tokens.accessToken, tokens.refreshToken)
                }

                refreshTokens {
                    val refreshToken = oldTokens?.refreshToken
                        ?: return@refreshTokens null

                    try {
                        val response = client.post("auth/refresh") {
                            markAsRefreshTokenRequest()
                            contentType(ContentType.Application.Json)
                            setBody(RefreshRequest(refreshToken))
                        }.body<TokenResponse>()

                        tokenStorage.saveTokens(response.accessToken, response.refreshToken)
                        BearerTokens(response.accessToken, response.refreshToken)
                    } catch (e: CancellationException) {
                        throw e
                    } catch (e: Exception) {
                        onSessionExpired()
                        null
                    }
                }

                sendWithoutRequest { request ->
                    request.url.host == apiHost
                }
            }
        }
    }
}
```

**Key points:**
- `markAsRefreshTokenRequest()` — the default; called inside the refresh request's own builder lambda to tag that specific outgoing request, which stops the `Auth` plugin from intercepting it and prevents an infinite refresh loop.
- `oldTokens` — Ktor's `RefreshTokensParams` receiver exposes the expired tokens.
- `loadTokens` returns `null` when there is no stored access token (fresh install, logged-out state) instead of `BearerTokens` wrapping a blank string — otherwise the plugin would attach `Authorization: Bearer ` with an empty value to every request.
- `sendWithoutRequest` does not mean "skip auth for this request" — it controls whether the bearer provider attaches the token *preemptively*, before waiting for a `401` challenge, or only *reactively*, after one is received (Ktor's own example gates this the same way: `request.url.host == "www.googleapis.com"`, from the [bearer auth docs](https://ktor.io/docs/client-bearer-auth.html)). Gate it on `request.url.host`, not on the path — a path-only check (e.g. excluding `"login"`/`"register"` segments) does nothing to stop this app's token from being attached to a request bound for an unrelated host that happens to share those path segments.
- Returning `null` from `refreshTokens` signals refresh failure; Ktor will not retry the original request.

### Alternative: isolated refresh client

Some teams prefer a dedicated `HttpClient` for the refresh call — one with no `Auth` plugin installed — to guarantee the refresh request cannot trigger another auth cycle. `markAsRefreshTokenRequest()` is the default and achieves the same goal with less ceremony; treat this as an alternative only for teams that want explicit separation.

Build that refresh client once — the same way as every other client in this skill, via `createHttpClient(engine, baseUrl)` from [client-setup.md](client-setup.md), just without installing `Auth` on top — and inject it as a long-lived singleton. Constructing an engine-less `HttpClient { }` inside the refresh function itself would violate the "never construct one per request or per repository" HARD-GATE just as much as a per-request authenticated client would, even though it has no `Auth` plugin, and it would also skip the shared `ContentNegotiation`/`HttpTimeout`/`Logging` configuration by using Ktor's engine-autodiscovery constructor instead of an injected engine.

```kotlin
private suspend fun refreshBearerToken(
    refreshClient: HttpClient, // injected singleton: createHttpClient(engine, baseUrl), no Auth installed
    tokenStorage: TokenStorage,
    onSessionExpired: () -> Unit,
): BearerTokens? {
    val tokens = tokenStorage.getTokens()
    val refreshToken = tokens.refreshToken.ifBlank { null } ?: return null
    return try {
        val response = refreshClient.post("auth/refresh") {
            contentType(ContentType.Application.Json)
            setBody(RefreshRequest(refreshToken))
        }.body<TokenResponse>()
        tokenStorage.saveTokens(response.accessToken, response.refreshToken)
        BearerTokens(response.accessToken, response.refreshToken)
    } catch (e: CancellationException) {
        throw e
    } catch (e: Exception) {
        onSessionExpired()
        null
    }
}
```

If using this pattern, build `refreshClient` once at startup (or wire it through Koin, see [testing-di.md](testing-di.md)) and call `refreshBearerToken` from inside `refreshTokens` instead of using `client` directly — do not build or close an `HttpClient` per refresh call.

## Token Storage

Implement with DataStore, encrypted SharedPreferences, or Keychain depending on platform. The interface uses app-owned types — convert to `BearerTokens` only at the plugin boundary. Never log tokens — see the logging rules in [error-handling.md](error-handling.md).

```kotlin
interface TokenStorage {
    suspend fun getTokens(): AuthTokens
    suspend fun saveTokens(accessToken: String, refreshToken: String)
    suspend fun clearTokens()
}

data class AuthTokens(val accessToken: String, val refreshToken: String)
```

## WebSockets

### Dependencies

Add `ktor-client-websockets` to your version catalog and `commonMain` dependencies.

### Connection and messaging

The snippets below show only the `WebSockets`-specific block. `sharedClient` is the app's single client from [client-setup.md](client-setup.md) (`createHttpClient(engine, baseUrl)`, already carrying `ContentNegotiation`, `HttpTimeout`, `Logging`, and `expectSuccess = false`); `install(WebSockets)` layers the extra capability onto it via `HttpClient.config {}` rather than constructing a second, differently-configured client.

```kotlin
val client = sharedClient.config {
    install(WebSockets) {
        pingIntervalMillis = 30_000   // no effect on OkHttp — see the caveat below
    }
}

client.webSocket("wss://api.example.com/ws") {
    send(Frame.Text(Json.encodeToString(SubscribeMessage("items"))))

    for (frame in incoming) {
        when (frame) {
            is Frame.Text -> {
                val message = Json.decodeFromString<ServerMessage>(frame.readText())
                // handle message
            }
            is Frame.Close -> break
            else -> Unit
        }
    }
}
```

`pingIntervalMillis` is valid in Ktor 3.5.1, but per [Ktor's WebSockets docs](https://ktor.io/docs/client-websockets.html) it "is not applicable for the OkHttp engine" — which is this skill's Android default (see [client-setup.md](client-setup.md)). On Android, configure the ping interval on the engine itself instead: `OkHttp.create { config { pingInterval(30, TimeUnit.SECONDS) } }`, passed as the injected `HttpClientEngine`.

### Session reference for external control

```kotlin
val session = client.webSocketSession("wss://api.example.com/ws")
session.send(Frame.Text("hello"))
val response = session.incoming.receive() as Frame.Text
session.close()
```

### Serialization converter

Type-safe WebSocket messaging using kotlinx.serialization. `contentConverter` is one more line inside the same `install(WebSockets) { }` block shown above, not a second, separate install:

```kotlin
install(WebSockets) {
    pingIntervalMillis = 30_000
    contentConverter = KotlinxWebsocketSerializationConverter(Json)
}

client.webSocket("wss://api.example.com/ws") {
    sendSerialized(SubscribeMessage("items"))
    val message = receiveDeserialized<ServerMessage>()
}
```

## Server-Sent Events

SSE provides server-push updates over HTTP. Unlike WebSockets, SSE is unidirectional (server to client) and works over standard HTTP. SSE support is built into `ktor-client-core` — no extra artifact needed.

### Basic usage

As with WebSockets above, this shows only the `SSE`-specific addition: `sharedClient` is the same single client from [client-setup.md](client-setup.md), and `install(SSE)` extends it via `HttpClient.config {}` rather than building a separate client.

```kotlin
val client = sharedClient.config {
    install(SSE)
}

client.sse("https://api.example.com/events") {
    incoming.collect { event ->
        println("Event: ${event.event}")
        println("Data: ${event.data}")
        println("ID: ${event.id}")
    }
}
```

### When to use SSE vs WebSocket

| Criterion | SSE | WebSocket |
|---|---|---|
| Direction | Server -> Client only | Bidirectional |
| Protocol | HTTP (standard) | WebSocket (protocol upgrade) |
| Auto-reconnect | Built-in | Manual |
| Binary data | No (text only) | Yes |
| Use case | Live feeds, notifications, progress, streaming AI | Chat, gaming, real-time collaboration |

Prefer SSE for server-push scenarios. Use WebSockets when the client also needs to send frequent messages.

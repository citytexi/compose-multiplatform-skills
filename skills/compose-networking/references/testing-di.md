# Testing and DI

Deterministic network tests with `MockEngine`, and how the client gets injected. The factory under test is `createHttpClient` from [client-setup.md](client-setup.md). Official docs: https://ktor.io/docs/client-testing.html

## MockEngine

Add the mock engine to `commonTest` only — it never ships in a production source set:

```toml
[libraries]
ktor-client-mock = { module = "io.ktor:ktor-client-mock", version.ref = "ktor" } # same catalog as client-setup.md — verify latest: https://ktor.io/docs/releases.html
```

```kotlin
commonTest.dependencies {
    implementation(libs.ktor.client.mock)
}
```

A success case asserts the request the client sent, then the domain model the repository mapped back:

```kotlin
@Test
fun `getItem returns mapped domain model`() = runTest {
    val mockEngine = MockEngine { request ->
        assertEquals("/items/123", request.url.encodedPath)
        respond(
            content = """{"id":"123","name":"Test","status":"active","created_at":1700000000}""",
            status = HttpStatusCode.OK,
            headers = headersOf(HttpHeaders.ContentType, "application/json"),
        )
    }
    val repo = ItemRepositoryImpl(ItemApi(createHttpClient(mockEngine, "https://api.example.com/")))

    val result = repo.getItem("123")

    assertTrue(result is NetworkResult.Success)
    assertEquals("Test", (result as NetworkResult.Success).data.name)
}
```

An error case asserts the typed failure the repository returns — never a thrown exception. `ItemApi.getItem` opts its request into `expectSuccess = true` (see [data-layer.md](data-layer.md)), so a 404 raises `ClientRequestException`; `ItemRepositoryImpl` wraps the call in `safeCall`, which classifies that exception into `NetworkResult.Failure.HttpError` (see [error-handling.md](error-handling.md)) instead of letting it escape:

```kotlin
@Test
fun `getItem returns HttpError on 404`() = runTest {
    val mockEngine = MockEngine {
        respond(
            content = """{"error":"not found"}""",
            status = HttpStatusCode.NotFound,
            headers = headersOf(HttpHeaders.ContentType, "application/json"),
        )
    }
    val repo = ItemRepositoryImpl(ItemApi(createHttpClient(mockEngine, "https://api.example.com/")))

    val result = repo.getItem("999")

    assertTrue(result is NetworkResult.Failure.HttpError)
    assertEquals(404, (result as NetworkResult.Failure.HttpError).code)
}
```

`MockEngine` can also branch on the request path to serve several endpoints from one lambda, falling back to an error response for anything unmapped:

```kotlin
val mockEngine = MockEngine { request ->
    when (request.url.encodedPath) {
        "/items" -> respond(
            content = """{"items":[],"total":0}""",
            headers = headersOf(HttpHeaders.ContentType, "application/json"),
        )
        "/items/1" -> respond(
            content = """{"id":"1","name":"Test","status":"active","created_at":1700000000}""",
            headers = headersOf(HttpHeaders.ContentType, "application/json"),
        )
        else -> respondError(HttpStatusCode.NotFound)
    }
}
```

## Request Assertions

Verify method, content type, and serialized body before responding — this catches a service method sending the wrong verb or a malformed payload before it ever reaches a real server. Assert on the media type rather than the full header string: Ktor's `ContentType.withCharsetIfNeeded`, which the kotlinx-serialization converter uses to build the outgoing `TextContent`, only appends a `charset` parameter when the top-level type is `text` — `application/json` is left bare — but checking the prefix keeps the test valid even if that composition rule ever changes:

```kotlin
@Test
fun `createItem sends correct request`() = runTest {
    val mockEngine = MockEngine { request ->
        assertEquals(HttpMethod.Post, request.method)
        assertTrue(request.body.contentType?.toString()?.startsWith("application/json") == true)

        val body = (request.body as TextContent).text
        assertTrue(body.contains("\"name\":\"Widget\""))

        respond(
            content = """{"id":"1","name":"Widget","status":"active","created_at":1700000000}""",
            status = HttpStatusCode.Created,
            headers = headersOf(HttpHeaders.ContentType, "application/json"),
        )
    }

    val client = createHttpClient(mockEngine, "https://api.example.com/")
    val api = ItemApi(client)
    val result = api.createItem(CreateItemRequest(name = "Widget"))
    assertEquals("Widget", result.name)
}
```

## Engine Injection

`createHttpClient` takes `HttpClientEngine` as a parameter precisely so tests can inject `MockEngine` while production passes the platform engine (`OkHttp`, `Darwin`, `CIO`, `Js`) — nothing else about the client differs between the two. Always build both through the same `createHttpClient` factory; constructing a bespoke `HttpClient` for tests (skipping `ContentNegotiation`, timeouts, or `expectSuccess = false`) lets test behavior drift from what production actually sends and lets bugs in the shared plugin configuration go unnoticed.

## DI Wiring

Koin wires the same `createHttpClient` factory as a singleton, with the engine itself supplied per platform:

```kotlin
val networkModule = module {
    single { createHttpClient(engine = get(), baseUrl = "https://api.example.com/") }
    single { ItemApi(get()) }
    single<ItemRepository> { ItemRepositoryImpl(get()) }
}

// platform modules supply the engine
// androidMain: single<HttpClientEngine> { OkHttp.create() }
// iosMain:     single<HttpClientEngine> { Darwin.create() }
```

`ItemRepositoryImpl(get())`'s `get()` resolves to the `ItemApi` singleton registered directly above it — Koin infers the type from `ItemRepositoryImpl`'s constructor parameter (see [data-layer.md](data-layer.md)), so the module never constructs `ItemRepositoryImpl` with the raw `HttpClient`.

Full Koin module organization — grouping modules by feature, wiring `expect`/`actual` platform modules for the engine, and scoping instances to a screen or session — is covered by the compose-di skill; nothing beyond the singletons above belongs in this file.

## Anti-Patterns

| Anti-pattern | Why it hurts | Better replacement |
|---|---|---|
| DTOs used directly in UI state | UI coupled to API contract, breaks on API changes | Map to domain models at repository boundary |
| Network calls in composables | Violates UDF, untestable, reruns on recomposition | Call from ViewModel, expose via StateFlow |
| No timeout configuration | Requests hang indefinitely on bad networks | Set `connectTimeoutMillis`, `requestTimeoutMillis`, `socketTimeoutMillis` |
| Hardcoded base URLs | Can't switch environments (dev/staging/prod) | Inject base URL via config or DI |
| Parsing/mapping inside the API service | Mixes concerns, harder to test | API service returns DTOs; repository maps to domain |
| Creating a new `HttpClient` per test | Tests miss plugin-config mismatches | Use the same `createHttpClient` factory with `MockEngine` |
| No compression | Wastes bandwidth on text-heavy APIs | `install(ContentEncoding) { gzip() }` |

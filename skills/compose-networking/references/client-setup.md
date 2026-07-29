# Ktor Client Setup

Dependencies, per-platform engines, and the single shared `HttpClient`. For DTOs and the repository layer see [data-layer.md](data-layer.md); for the `safeRequest` wrapper see [error-handling.md](error-handling.md). Official docs: https://ktor.io/docs/client-engines.html

## Dependencies

Version catalog pinned to Ktor 3.5.1:

```toml
[versions]
ktor = "3.5.1"   # verify latest: https://ktor.io/docs/releases.html

[libraries]
ktor-client-core = { module = "io.ktor:ktor-client-core", version.ref = "ktor" }
ktor-client-content-negotiation = { module = "io.ktor:ktor-client-content-negotiation", version.ref = "ktor" }
ktor-serialization-kotlinx-json = { module = "io.ktor:ktor-serialization-kotlinx-json", version.ref = "ktor" }
ktor-client-logging = { module = "io.ktor:ktor-client-logging", version.ref = "ktor" }
ktor-client-auth = { module = "io.ktor:ktor-client-auth", version.ref = "ktor" }
ktor-client-websockets = { module = "io.ktor:ktor-client-websockets", version.ref = "ktor" }
ktor-client-okhttp = { module = "io.ktor:ktor-client-okhttp", version.ref = "ktor" }
ktor-client-darwin = { module = "io.ktor:ktor-client-darwin", version.ref = "ktor" }
ktor-client-cio = { module = "io.ktor:ktor-client-cio", version.ref = "ktor" }
ktor-client-js = { module = "io.ktor:ktor-client-js", version.ref = "ktor" }
ktor-client-mock = { module = "io.ktor:ktor-client-mock", version.ref = "ktor" }
```

Note: use these base module ids in KMP — Gradle resolves the per-target variant. Do not write platform-suffixed ids such as `ktor-client-okhttp-jvm`.

Then the source-set wiring:

```kotlin
commonMain.dependencies {
    implementation(libs.ktor.client.core)
    implementation(libs.ktor.client.content.negotiation)
    implementation(libs.ktor.serialization.kotlinx.json)
    implementation(libs.ktor.client.logging)
}

androidMain.dependencies { implementation(libs.ktor.client.okhttp) }
iosMain.dependencies { implementation(libs.ktor.client.darwin) }
jvmMain.dependencies { implementation(libs.ktor.client.cio) }
jsMain.dependencies { implementation(libs.ktor.client.js) }

commonTest.dependencies { implementation(libs.ktor.client.mock) }
```

## Platform Engines

| Target | Engine class | Artifact |
|---|---|---|
| Android | `OkHttp` (or `Android`) | `ktor-client-okhttp` / `ktor-client-android` |
| JVM / desktop | `CIO` (or `OkHttp`, `Java`, `Apache5`) | `ktor-client-cio` / `ktor-client-okhttp` |
| iOS / macOS | `Darwin` | `ktor-client-darwin` |
| JS / Wasm-JS | `Js` | `ktor-client-js` |
| Any (tests) | `MockEngine` | `ktor-client-mock` |

Select the engine in the platform source set and pass it into `createHttpClient` — `commonMain` code never names an engine. Verify engine availability for your exact targets at https://ktor.io/docs/client-engines.html

## HttpClient Configuration

The single factory (created once and injected; see `testing-di.md` for the DI wiring):

```kotlin
fun createHttpClient(engine: HttpClientEngine, baseUrl: String): HttpClient =
    HttpClient(engine) {
        expectSuccess = false   // safeRequest inspects status itself — see error-handling.md

        install(ContentNegotiation) {
            json(Json {
                ignoreUnknownKeys = true    // tolerate new server fields
                coerceInputValues = true    // null -> declared defaults
                encodeDefaults = true
            })
        }

        defaultRequest {
            url(baseUrl)
            headers.append("Accept", "application/json")
        }

        install(HttpTimeout) {
            connectTimeoutMillis = 15_000
            requestTimeoutMillis = 30_000
            socketTimeoutMillis = 15_000
        }

        install(Logging) {
            logger = Logger.DEFAULT
            level = LogLevel.HEADERS
            sanitizeHeader { it == "Authorization" }
        }
    }
```

One `HttpClient` per app, created once and injected — never per request or per repository. `ignoreUnknownKeys = true` is mandatory; set `isLenient = true` only for non-standard APIs, since it accepts malformed JSON and hides data problems.

`defaultRequest { url(baseUrl) }` resolves each call's final URL with standard base/relative combination rules, so the two sides of the contract must match: `baseUrl` must end with `/` (`"https://api.example.com/"`, not `"https://api.example.com"`), and every per-call path must be relative, without a leading `/` (`client.get("items")`, not `client.get("/items")`). Mixing the two — a `baseUrl` without a trailing slash, or a path with a leading slash — silently resolves to the wrong URL instead of throwing, dropping part of the path. The API service methods in [data-layer.md](data-layer.md) (`client.get("items")`, `client.get("items/$id")`) and the `MockEngine` tests in [testing-di.md](testing-di.md) (`assertEquals("/items/123", request.url.encodedPath)`) both depend on this convention.

## expectSuccess

This skill sets `expectSuccess = false` on the shared client so `safeRequest` inspects `response.status` and returns a typed `NetworkResult`. Never mix `expectSuccess = true` with manual status inspection in the same call — that's the banned pattern, not per-request overrides in general. A specific request may still opt into `expectSuccess = true` on top of the shared default when the caller classifies the resulting exception instead of inspecting status — see `safeCall` and the API service in [data-layer.md](data-layer.md) and [error-handling.md](error-handling.md). Plugin install order lives in [error-handling.md](error-handling.md).

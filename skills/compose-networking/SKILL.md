---
name: compose-networking
description: Use when adding or reviewing HTTP networking in a Compose Multiplatform app — configuring the Ktor client and platform engines, modeling DTOs and mappers, writing API service or repository code, handling network errors, wiring bearer-token auth, WebSockets or SSE, or testing with MockEngine.
---

# Compose Multiplatform Networking (Ktor)

HTTP networking for Compose Multiplatform with the Ktor client. One shared
`HttpClient`, DTOs mapped to domain models at the data boundary, and every call
returning a typed `NetworkResult<T>`.

<HARD-GATE>
- Build ONE `HttpClient` and inject it. NEVER construct one per request or per repository.
- NEVER let a DTO or a Ktor type (`ClientRequestException`, `HttpResponse`) escape the data layer.
- Every request goes through `safeRequest` or `safeCall` and returns `NetworkResult<T>` — never a raw throwing call.
- ALWAYS rethrow `CancellationException` before handling other exceptions.
- NEVER log request bodies or unsanitized `Authorization` headers in production.
</HARD-GATE>

## Checklist

Create a todo for each item and complete them in order:

1. Read the existing networking code — where the client is built, which engine each source set uses, how errors surface today.
2. Identify the concern — client setup, data layer, error handling, auth/realtime, or testing.
3. Apply the Core Rules below.
4. Load the relevant reference for depth.
5. Check the work against the Red Flags table.
6. Cover the change with a `MockEngine` test that uses the same client factory as production.

## Core Rules

### One client, engine per platform
`createHttpClient(engine, baseUrl)` installs ContentNegotiation (with
`ignoreUnknownKeys = true`), `HttpTimeout`, `Logging`, and `defaultRequest`, and
sets `expectSuccess = false`. `commonMain` never names an engine — the platform
source set supplies `OkHttp` / `Darwin` / `CIO` / `Js`.
→ dependencies, engines, client configuration: [references/client-setup.md](references/client-setup.md)

### DTOs stop at the repository
`@Serializable` DTOs mirror the API contract; domain models carry no
serialization annotations. The API service returns DTOs, the repository maps
them with `toDomain()` and returns domain types.
→ DTOs, mappers, service and repository layers: [references/data-layer.md](references/data-layer.md)

### Typed results, not exceptions
`safeRequest` inspects the status, parses the error envelope, and classifies
failures into `NetworkResult.Failure` variants; `safeCall` runs the same
classification around any suspending call — including a repository calling
into an API service — from the thrown-exception side. Ktor exception types
never travel upward.
→ NetworkResult, safeRequest, safeCall, classification, plugin order: [references/error-handling.md](references/error-handling.md)

### Auth and realtime
Bearer tokens via the `Auth` plugin, with `markAsRefreshTokenRequest()` inside
`refreshTokens` so the refresh call is not intercepted. SSE for server-push,
WebSockets when the client also sends.
→ bearer auth, token storage, WebSockets, SSE: [references/auth-realtime.md](references/auth-realtime.md)

### Test with MockEngine
Inject `HttpClientEngine`; tests pass `MockEngine` into the same
`createHttpClient` factory production uses, so plugin config cannot drift.
→ MockEngine patterns, request assertions, Koin wiring: [references/testing-di.md](references/testing-di.md)

## Red Flags — STOP

| Smell | Fix |
|---|---|
| `HttpClient` built per request or per repository | One instance, created once and injected |
| DTO used in UI state or a domain signature | Map with `toDomain()` at the repository boundary |
| `ClientRequestException` caught in a ViewModel | Classify in `safeRequest`; return `NetworkResult.Failure` |
| `catch (e: Exception)` around a request with no rethrow | Rethrow `CancellationException` first |
| `expectSuccess = true` mixed with manual `response.status` checks in the same call | Pick one pairing: `expectSuccess = false` + `safeRequest` (status inspected by the wrapper), or per-request `expectSuccess = true` + `safeCall` (exceptions classified by the wrapper) |
| Missing `ignoreUnknownKeys = true` | Add it — a new server field must not crash the client |
| `LogLevel.BODY` or unsanitized `Authorization` in production | `LogLevel.HEADERS` + `sanitizeHeader { it == "Authorization" }` |
| Refresh call made by the same authenticated client | `markAsRefreshTokenRequest()`, or an isolated refresh client |
| Base URL hardcoded at call sites | `defaultRequest { url(baseUrl) }`, base URL injected |
| No timeouts configured | `install(HttpTimeout)` with connect/request/socket values |
| Mapping or parsing inside the API service | Service returns DTOs; the repository maps |
| Tests hitting the live network | `MockEngine` through the shared client factory |

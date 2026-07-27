---
name: compose-async
description: Use when writing or reviewing Kotlin Coroutines and Flow in a Compose Multiplatform app — choosing StateFlow/SharedFlow/Channel, applying Flow operators, picking dispatchers, structuring concurrency and exception handling, converting cold flows to hot with stateIn/shareIn, or testing flows.
---

# Compose Multiplatform Coroutines & Flow

Asynchronous programming for Compose Multiplatform with Kotlin Coroutines and
Flow. Works across all CMP targets — mind the platform caveats (notably
`Dispatchers.IO`).

<HARD-GATE>
- NEVER launch from `GlobalScope` — use `viewModelScope` or a structured scope tied to a lifecycle.
- NEVER swallow `CancellationException` — rethrow it before handling other exceptions.
- NEVER hardcode `Dispatchers.IO` in `commonMain` that targets JS/Wasm — inject the dispatcher or use `expect/actual`.
- Expose cold flows as hot with `stateIn`/`shareIn` declared ONCE as a `val`, never per function call.
</HARD-GATE>

## Checklist

Create a todo for each item and complete them in order:

1. Read the existing async code — scopes in use, dispatcher choices, how flows are collected.
2. Identify the concern — hot-stream choice, operator, dispatcher/scope, cold→hot, or advanced/testing.
3. Apply the Core Rules below.
4. Load the relevant reference for depth.
5. Check the work against the Red Flags table.
6. Ensure structured concurrency: every coroutine has a lifecycle-bound scope; cancellation is respected.

## Core Rules

### Hot-stream selection
`StateFlow` for UI state (holds latest, conflated), `Channel(BUFFERED)` for
one-off effects (single consumer), `SharedFlow` for fan-out broadcasts.
→ comparison, MVI mapping, common mistakes: [references/hot-streams.md](references/hot-streams.md)

### Flow operators
Prefer declarative operators over manual collection. `flatMapLatest` for
restartable work (search); `combine` for multiple state sources (seed with
`onStart` so it emits before all upstreams do).
→ transform / flatten / combine / timing / terminal: [references/flow-operators.md](references/flow-operators.md)

### Dispatchers, scopes, exceptions
Inject dispatchers; the callee switches with `withContext`, not the caller.
`Dispatchers.IO` is JVM + Native only (not JS/Wasm). Structured concurrency
always; rethrow `CancellationException`.
→ dispatchers (+ CMP caveats), scopes, exception handling, flowOn: [references/dispatchers-scopes.md](references/dispatchers-scopes.md)

### Cold to hot
Expose repository flows to the UI with
`stateIn(scope, SharingStarted.WhileSubscribed(5_000), initial)` — declared once.
→ stateIn/shareIn, SharingStarted policies: [references/statein-sharein.md](references/statein-sharein.md)

### Advanced + testing
Backpressure (`buffer`/`conflate`/`collectLatest`), `callbackFlow`/`channelFlow`,
`Mutex`/`Semaphore`, and Turbine for Flow tests.
→ advanced patterns and testing: [references/advanced.md](references/advanced.md)

## Red Flags — STOP

| Smell | Fix |
|---|---|
| `GlobalScope.launch { }` | `viewModelScope` or a structured, lifecycle-bound scope |
| `catch (e: Exception)` swallowing `CancellationException` | Rethrow `CancellationException` first |
| Hardcoded `Dispatchers.IO` in JS/Wasm `commonMain` | Inject dispatcher / `expect/actual`; `Dispatchers.Default` fallback |
| `StateFlow` used for one-off events | `Channel(BUFFERED)` — StateFlow replays latest to new collectors |
| `Channel()` default (RENDEZVOUS) for effects | `Channel(Channel.BUFFERED)` |
| `SharingStarted.Eagerly`/`Lazily` where UI state belongs | `WhileSubscribed(5_000)` |
| `stateIn`/`shareIn` created inside a per-call function | Declare once as a `val` |
| `runBlocking` on the main thread | `launch`/`async` from a coroutine scope |
| Blocking I/O on `Dispatchers.Default` | `Dispatchers.IO` (JVM/Native) or an injected IO dispatcher |
| `synchronized` in coroutine code | `Mutex.withLock` |
| `callbackFlow` without `awaitClose { }` | Add `awaitClose` cleanup (mandatory) |

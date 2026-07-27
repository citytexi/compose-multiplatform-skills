# Dispatchers, Scopes, and Exception Handling

Where coroutines run, how their lifecycles nest, and how failures propagate.

## Dispatchers

| Dispatcher | Use for | CMP support |
|---|---|---|
| `Dispatchers.Main` | UI state updates, composable callbacks | All targets (needs `kotlinx-coroutines-android` on Android; included on Darwin/iOS; desktop JVM needs `kotlinx-coroutines-swing` or `kotlinx-coroutines-javafx`) |
| `Dispatchers.Default` | CPU-heavy work — sorting, parsing | All targets |
| `Dispatchers.IO` | Network, database, file I/O | **JVM + Kotlin/Native (since 1.7.0); NOT Kotlin/JS or Wasm** |

In `commonMain` that must compile for JS/Wasm, do not reference `Dispatchers.IO` directly — inject the dispatcher or provide it via `expect/actual`; fall back to `Dispatchers.Default` or `Dispatchers.Default.limitedParallelism(n)`.

## Main-Safe Rule

The callee switches dispatchers, not the caller. A repository (or other data-layer class) should hop to an I/O dispatcher internally so callers can invoke it safely from `Dispatchers.Main`:

```kotlin
class ProductRepository(
    private val api: ProductApi,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO,
) {
    suspend fun getProducts(): List<Product> = withContext(ioDispatcher) {
        api.getProducts().toDomain()
    }
}
// Caller: viewModelScope.launch { repository.getProducts() } — safe from Main
```

Inject dispatchers as constructor params (`ioDispatcher: CoroutineDispatcher = Dispatchers.IO`) for testability — tests substitute a test dispatcher — and for platform-safety, since a JS/Wasm-targeting `commonMain` can supply an `expect/actual` dispatcher instead of hardcoding `Dispatchers.IO`.

## flowOn

`flowOn` changes the dispatcher for upstream operators and buffers at the switch point. It affects everything above it in the chain only — operators below `flowOn` still run on the collector's dispatcher:

```kotlin
repository.observeProducts()        // runs on IO
    .map { it.toDomain() }           // runs on IO
    .flowOn(Dispatchers.IO)           // everything above runs on IO
    .collect { updateUi(it) }         // runs on caller's dispatcher (Main)
```

## Structured Concurrency and Scopes

| Scope | Lifecycle | Use for |
|---|---|---|
| `viewModelScope` | ViewModel cleared | ViewModel coroutines (CMP `commonMain` since lifecycle 2.8+) |
| `lifecycleScope` | Lifecycle destroyed | Android Activity/Fragment only |
| `rememberCoroutineScope()` | Leaves composition | Compose event handlers |
| `coroutineScope { }` | All children complete | Parallel decomposition (one fails → all cancel) |
| `supervisorScope { }` | Child failure independent | Independent parallel tasks |

Use `supervisorScope` when tasks are independent (e.g., dashboard sections that should not cancel each other). Use `coroutineScope` when all children must succeed together. Never use `GlobalScope` — no lifecycle, memory leak. Never create an unbound `CoroutineScope(Job())` without lifecycle management.

## Exception Handling

### launch vs async

`launch` propagates exceptions immediately (to the scope's handler). `async` defers exceptions until `await()` is called on the resulting `Deferred`. This applies to a top-level `async` started directly in a `CoroutineScope`/`supervisorScope`; a structured child `async` (one with a parent `Job` in an ordinary `coroutineScope`) still propagates its exception to the parent immediately, before `await()` is ever called.

```kotlin
viewModelScope.launch {
    try {
        val data = repository.fetchData()
        _state.update { it.copy(data = data, isLoading = false) }
    } catch (e: IOException) {
        _state.update { it.copy(error = "Network error", isLoading = false) }
    }
}
```

### CancellationException — never swallow

```kotlin
// BAD: catch(e: Exception) catches CancellationException — zombie coroutine
try { suspendingWork() }
catch (e: Exception) { handleError(e) }

// GOOD: rethrow CancellationException before the general catch
try { suspendingWork() }
catch (e: CancellationException) { throw e }
catch (e: Exception) { handleError(e) }
```

For scope-level handling of independent children, pair `SupervisorJob()` with a `CoroutineExceptionHandler` so one child's failure doesn't cancel its siblings and uncaught exceptions are routed to a single handler instead of crashing the app.

# Advanced Coroutines & Flow

Backpressure, bridging callback APIs, concurrency primitives, and testing. For core patterns see the other references in this skill: [hot-streams.md](hot-streams.md), [flow-operators.md](flow-operators.md), [dispatchers-scopes.md](dispatchers-scopes.md), [statein-sharein.md](statein-sharein.md).

## Backpressure

When producer emits faster than consumer processes:

| Strategy | Behavior | Use when |
|---|---|---|
| Default (no buffer) | Producer suspends until consumer processes | Simple sequential work |
| `buffer(capacity)` | Queue between producer and consumer | Smooth speed spikes, process every item |
| `conflate()` | Drop old values, keep only latest | UI updates, progress bars — stale data unnecessary |
| `collectLatest { }` | Cancel previous processing when new value arrives | Search — only final result matters |

```kotlin
// Search with collectLatest: only the last query completes
queryFlow
    .debounce(300)
    .distinctUntilChanged()
    .collectLatest { query ->
        val results = repository.search(query) // cancelled if new query arrives
        _state.update { it.copy(results = results) }
    }
```

`flowOn` is a related but separate concern — it changes the dispatcher for upstream operators and buffers at the context switch. See [dispatchers-scopes.md](dispatchers-scopes.md) for details.

## callbackFlow and channelFlow

### callbackFlow — bridge listener APIs to Flow

Use `callbackFlow` to convert callback-based platform APIs into a `Flow`. In CMP, place these wrappers in `expect/actual` declarations or platform source sets.

```kotlin
// Android example — LocationManager (place in androidMain for CMP)
fun LocationManager.locationUpdates(): Flow<Location> = callbackFlow {
    val listener = LocationListener { location ->
        trySend(location) // non-blocking, thread-safe
    }
    requestLocationUpdates(GPS_PROVIDER, 1000L, 0f, listener)
    awaitClose { removeUpdates(listener) } // mandatory cleanup
}
```

**Rules:**
- Use `trySend()` (non-blocking) not `send()` (suspending) from callbacks
- `awaitClose { }` is mandatory — omitting it throws `IllegalStateException`
- The cleanup block in `awaitClose` unregisters the listener
- Place platform-specific wrappers like this in `expect/actual` declarations or platform source sets, not in `commonMain`

### channelFlow — concurrent production

```kotlin
fun loadDashboard(): Flow<DashboardSection> = channelFlow {
    launch { send(DashboardSection.Profile(fetchProfile())) }
    launch { send(DashboardSection.Stats(fetchStats())) }
    launch { send(DashboardSection.Feed(fetchFeed())) }
}
```

Use `channelFlow` when producing values from multiple concurrent coroutines. Use `callbackFlow` specifically for wrapping external callback APIs.

## Concurrency Primitives

### Mutex — mutual exclusion

```kotlin
private val mutex = Mutex()
private var tokenCache: String? = null

suspend fun getToken(): String = mutex.withLock {
    tokenCache ?: refreshToken().also { tokenCache = it }
}
```

Use Mutex for: token refresh synchronization, shared mutable state protection, sequential access to resources.

### Semaphore — limited concurrency

```kotlin
private val semaphore = Semaphore(permits = 3)

suspend fun downloadFile(url: String): ByteArray = semaphore.withPermit {
    httpClient.get(url).body()
}
```

Use Semaphore for: rate-limiting concurrent network calls, limiting parallel file operations.

### Why not synchronized?

`synchronized` blocks the thread. Coroutines suspend — blocking a thread holding a coroutine defeats the purpose. Use `Mutex.withLock` instead of `synchronized` in coroutine code.

## Testing with Turbine

### Turbine API quick reference

| Function | Purpose |
|---|---|
| `flow.test { }` | Start collecting and asserting |
| `awaitItem()` | Wait for next emission, fail if timeout |
| `awaitComplete()` | Assert flow completes |
| `awaitError()` | Assert flow throws |
| `expectNoEvents()` | Assert no emissions pending |
| `cancelAndIgnoreRemainingEvents()` | Clean up after assertions |
| `cancelAndConsumeRemainingEvents()` | Cancel and return remaining events |

`runTest` from `kotlinx-coroutines-test:1.11.0` provides deterministic coroutine execution — delays are skipped automatically. Use `advanceUntilIdle()` to process all pending coroutines.

```kotlin
// commonTest
implementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.11.0")  // verify latest: https://repo1.maven.org/maven2/org/jetbrains/kotlinx/kotlinx-coroutines-test/maven-metadata.xml
implementation("app.cash.turbine:turbine:1.2.0")                         // verify latest: https://repo1.maven.org/maven2/app/cash/turbine/turbine/maven-metadata.xml
```

For full ViewModel event→state→effect testing, see the compose-architecture skill.

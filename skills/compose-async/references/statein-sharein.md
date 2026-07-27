# stateIn and shareIn

Convert a cold `Flow` to a hot `StateFlow`/`SharedFlow`. Always declare the result as a `val` created once — never inside a function that runs per call (that leaks hot flows).

```kotlin
val products: StateFlow<List<Product>> = repository.observeProducts()
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
```

## SharingStarted Strategies

| Strategy | Starts | Stops | Use for |
|---|---|---|---|
| `WhileSubscribed(5000)` | First collector | 5s after last gone | ViewModel state — the default |
| `Lazily` | First collector | Never (scope cancel) | Expensive-to-restart shared resources |
| `Eagerly` | Immediately | Never (scope cancel) | Data needed before first collector |

The `5_000` ms grace window survives configuration changes and brief unsubscription (Android rotation, screen-off) without restarting the upstream flow. For hot data feeding a screen, prefer `WhileSubscribed`; reserve `Eagerly`/`Lazily` for the specific cases above.

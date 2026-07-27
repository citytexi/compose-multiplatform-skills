# Hot Streams: StateFlow, SharedFlow, Channel

Choosing the right hot stream. For how these map onto an MVI screen (state owner, effect delivery), see the compose-architecture skill — this file covers the coroutine mechanics.

## StateFlow vs SharedFlow vs Channel

| | StateFlow | SharedFlow | Channel |
|---|---|---|---|
| Holds current value | Yes (replay=1, conflated) | No (configurable replay) | No |
| New collector gets | Latest value immediately | Replayed values (if configured) | Nothing (consumed) |
| Delivery | All collectors | All collectors | One receiver |
| Duplicate filtering | `distinctUntilChanged` built-in | None | None |
| Use for | UI state | Broadcasting events | One-off effects |

### MVI mapping

```kotlin
class ProductViewModel : ViewModel() {
    private val _state = MutableStateFlow(ProductState())
    val state: StateFlow<ProductState> = _state.asStateFlow()

    private val _effects = Channel<ProductEffect>(Channel.BUFFERED)
    val effects: Flow<ProductEffect> = _effects.receiveAsFlow()
}
```

## When to Use Which

- **Screen state** (loading, data, errors, form input) → `StateFlow`
- **One-off UI effects** (navigate, snackbar, haptic) → `Channel(BUFFERED)` collected via `CollectEffect`
- **Broadcasting to multiple collectors** (analytics, logging) → `SharedFlow` with appropriate replay
- **Hot data streams** (search results reacting to query) → cold `Flow` converted via `stateIn` — see [statein-sharein.md](statein-sharein.md)

## Common Mistakes

- StateFlow for one-off events → shows twice on config change (new collector gets latest)
- `SharedFlow(replay=0)` for mandatory effects → lost when UI detached
- `Channel()` default (RENDEZVOUS) → suspends sender if no receiver; use `Channel.BUFFERED`

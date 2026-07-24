# Architecture & State Management (MVI)

Shared architecture concepts for MVI screens. See [mvi.md](mvi.md) for the Event/State/Effect pattern and [state-modeling.md](state-modeling.md) for form/derived state.

Preservation rule: if the project already has a coherent MVI (or variant) screen architecture pattern, preserve it unless the user explicitly asks to migrate or the current pattern cannot satisfy a required constraint.

## Source of Truth

Per screen:

- **Screen behavior:** `StateFlow<ScreenState>` owned by the screen state holder, often a ViewModel
- **Persisted data:** repository / database / remote service
- **Local visual-only concerns:** local Compose state in the route or leaf composable

Do not mix them.

## Choosing a State Owner

| Situation | Default owner | Why |
|---|---|---|
| Visual state for one composable subtree | Local Compose state | Smallest scope, easiest reuse |
| Complex UI logic, no business/data responsibilities | Plain state holder class | Testable without ViewModel |
| Screen-level business rules, async, persistence, effects | ViewModel | Lifecycle integration, screen state ownership |

A ViewModel is one implementation of a screen state holder, not a requirement for every composable.

## When to Use Lighter Patterns

- Purely presentational leaf composables
- Small screens with trivial local state and no async/persistence
- Prototypes unless the user asks to formalize
- Do not invent reducers, result types, or global frameworks unless they earn their keep

## Domain Layer

Pure business logic. Zero platform dependencies — runs in `commonTest` without emulators.

| Rule | Rationale |
|---|---|
| Zero platform imports | Testable anywhere, shareable |
| Domain models ≠ DTOs or entities | Decouples from API/DB schema |
| Repository interfaces in domain, impls in data | Dependency inversion |
| Mappers at data boundary | Domain ignores serialization (mappers live at the data boundary) |
| Use cases only for multi-step orchestration | Don't wrap single repo calls |

```kotlin
data class Item(val id: String, val name: String, val status: ItemStatus)

interface ItemRepository {
    suspend fun getById(id: String): Item?
    suspend fun save(item: Item)
}

class CreateItemUseCase(private val repository: ItemRepository, private val validator: ItemValidator) {
    suspend operator fun invoke(name: String, status: ItemStatus): Result<Item> {
        val errors = validator.validate(name)
        if (errors.isNotEmpty()) return Result.failure(ValidationException(errors))
        val item = Item(id = uuid(), name = name.trim(), status = status)
        repository.save(item)
        return Result.success(item)
    }
}
```

## Inter-Feature Communication

| Need | Pattern | Why |
|---|---|---|
| React to event from another feature | Event bus (`SharedFlow`) | Fire-and-forget, many listeners |
| Navigate to another feature | Feature API contract (`:api` module) | Type-safe, no impl dependency |
| Pass data back | Feature API + callback | Structured return, testable |
| Shared data stream (current user) | Shared repository in `core` | Persistent state, not one-shot |

**Anti-patterns:** importing another feature's ViewModel, global "god event bus" with 50 events, cross-feature data via `CompositionLocal`.

For the full api/impl split pattern, split features into `:api` and `:impl` modules so the contract stays a leaf dependency.

## Module Dependency Rules

```text
app -> feature:*:impl, feature:*:api, core:*
feature:*:impl -> feature:*:api (any feature), core:*
feature:*:api -> core:designsystem (route types only)
core:data -> core:network, core:database, core:datastore
```

| Forbidden | Why |
|---|---|
| `feature:impl` → another `feature:impl` | Circular risk |
| `feature:api` → any `feature` | API contracts must be leaf dependencies |
| `core:*` → `feature:*` or `app` | Core cannot depend on consumers |
| Domain → Data layer | Domain declares interfaces, data implements |

## Where Logic Belongs

| Logic | Where |
|---|---|
| Validation | ViewModel/domain — never in composable body |
| Calculations | Pure calculator/domain service called by ViewModel |
| Async orchestration | ViewModel — launch/cancel, debounce, ignore stale |
| Side effects | ViewModel via `Effect` or `viewModelScope.launch` |
| Local UI state | Composable — `LazyListState`, focus, animation, expansion, tooltip |

Not acceptable in composables: validation, derived totals, data loading, submit enablement, business decisions.

## Effect Delivery

`Channel<Effect>(Channel.BUFFERED)` with `receiveAsFlow()` — default for single-consumer effects. Buffers for reliable delivery, single consumer, no replay. `SharedFlow(replay=0)` acceptable for truly fire-and-forget signals. Preserve an existing `SharedFlow` effect mechanism when it is used consistently across the project.

## Reactive Data Collection

```kotlin
private fun collectData() {
    viewModelScope.launch {
        repository.observe()
            .catch { sendEffect(ShowError(it.message ?: "Load failed")) }
            .collect { data -> updateState { copy(items = data, isLoading = false) } }
    }
}
```

Room and DataStore `Flow` queries auto-re-emit on changes. Map data-layer types to domain models at the repository boundary.

## State Collection and Slicing

**Default:** collect whole screen state once at the route boundary, slice downward.

- `Route` collects `StateFlow<ScreenState>`
- `Screen` receives `ScreenState`
- Leaves receive **only what they need**
- Do **not** make leaves observe the ViewModel directly

### Callbacks at Boundaries

- `onEvent(Event)` at the route/screen boundary; leaves prefer specific callbacks
- Reusable components must not know your event contract or ViewModel type

## Adapting to Existing Projects

| Project has | Action |
|---|---|
| MVI with base class (`MviHost`, `BaseViewModel`) | Use it. Don't introduce a competing base. See [mvi.md](mvi.md) |
| Plain state holder classes | Valid. Only move to ViewModel when the screen needs async/persistence/lifecycle |
| 4-type MVI (Event, Result, State, Effect) | Use `Result` as the project expects. Don't strip it out |
| No architecture | Choose MVI. Trivial screens: local state is fine |

## Scaling Notes

- Small screens: one file for contract + ViewModel
- Medium: split contract, ViewModel, screen, route
- Large: extract calculation, validation, formatting into dedicated collaborators
- Do **not** create nested state holders for every card/section by default — only when independent lifecycle, async, tests, and real reuse justify it

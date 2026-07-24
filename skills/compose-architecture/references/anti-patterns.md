# Anti-Patterns

Cross-cutting patterns that hurt MVI Compose Multiplatform codebases. For overengineering patterns (bloated base classes, unnecessary use cases, 4-type MVI), see [clean-code.md](clean-code.md).

## Cross-Cutting Anti-Patterns

| Anti-pattern | Why it is harmful | Better replacement | Detailed in |
|---|---|---|---|
| Business logic inside composables | forks source of truth, hurts testability, reruns during composition | move logic into ViewModel/domain services | [architecture.md](architecture.md) |
| Giant god-ViewModel | blast radius too large, slow reasoning, hard ownership | one ViewModel per screen or independent flow | [architecture.md](architecture.md) |
| Scattered `updateState`/`sendEffect` with no structure | state transitions hard to trace, mutations across callbacks | disciplined `onEvent()` as single entry point | [clean-code.md](clean-code.md) |
| Unstable state models (mutable collections, lambdas in state) | defeats Compose skipping, more recomposition | immutable data classes, immutable collections | [state-modeling.md](state-modeling.md) |
| Duplicated derived data (`total`, `formattedTotal`, `hasTotal` all stored) | bugs from drift, harder transitions | keep canonical value + derive via computed property | [architecture.md](architecture.md) |
| Broad state reads in parent composables | recomposition cascades to all children | slice state, pass only required props to each child | [architecture.md](architecture.md#state-collection-and-slicing) |
| Mutable state passed deep into tree | hidden writes, unpredictable data flow | explicit props + callbacks | [architecture.md](architecture.md#state-collection-and-slicing) |
| One-off events stored as consumable state (`showSnackbarOnce = true`) | event replay on config change, stale effects | separate `Effect` via `Channel` | [architecture.md](architecture.md#effect-delivery) |
| No-op state emissions (copy state when nothing changed) | wasted recomposition cycles | guard unchanged values before updating | performance (guard unchanged state) |
| ViewModel doing platform work directly (share, analytics, navigation) | breaks testability, platform coupling | emit effects, handle in Route composable | [architecture.md](architecture.md) |
| Display strings stored too early (ViewModel emits pre-baked formatted text) | locale inflexibility, state duplication, harder reuse | keep canonical values until presentation boundary | [architecture.md](architecture.md) |
| Too many trivial composables (wrappers around single `Text`/`Spacer`) | fragmentation, harder reading | extract only meaningful boundaries | [clean-code.md](clean-code.md) |
| Forcing MVI migration on existing codebase | churn without value, team friction | respect existing patterns, introduce MVI for new features only | [clean-code.md](clean-code.md) |
| Inline fully qualified package paths | hurts readability, clutters business logic, hides intent behind package noise | import at file top; use `import ... as ...` for name clashes | [clean-code.md](clean-code.md) |

## Examples

### Business logic inside composables

```kotlin
// BAD — logic in composable; untestable, reruns on every recomposition
@Composable
fun CheckoutScreen(viewModel: CheckoutViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val total = state.items.sumOf { it.price * it.qty } // business logic here
    val tax = total * 0.08
    Text("Total: $${"$"}total  Tax: $${"$"}tax")
}

// GOOD — derive in ViewModel/state, composable only renders
data class CheckoutState(
    val items: List<LineItem> = emptyList(),
    val total: Double = 0.0,
    val tax: Double = 0.0,
)

@Composable
fun CheckoutScreen(state: CheckoutState, onEvent: (CheckoutEvent) -> Unit) {
    Text("Total: ${state.total}  Tax: ${state.tax}")
}
```

### One-off events as consumable state booleans

```kotlin
// BAD — event replays on config change, race between read and reset
data class UiState(val showSnackbar: Boolean = false)

LaunchedEffect(state.showSnackbar) {
    if (state.showSnackbar) {
        snackbarHostState.showSnackbar("Saved")
        viewModel.onEvent(DismissSnackbar)   // consumer must remember to reset
    }
}

// GOOD — Channel delivers exactly once, survives config change
sealed interface Effect { data class ShowSnackbar(val msg: String) : Effect }

CollectEffect(viewModel.effects) { effect ->
    when (effect) {
        is Effect.ShowSnackbar -> snackbarHostState.showSnackbar(effect.msg)
    }
}
```

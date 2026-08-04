# Orbit MVI Library

[Orbit](https://orbit-mvi.org/) is a Kotlin Multiplatform MVI framework. Use it
when you want the Event/State/Effect discipline from [mvi.md](mvi.md) without
hand-writing `MutableStateFlow` + `Channel` + `onEvent()` boilerplate on every
screen. Orbit supplies the container, the state stream, and the side-effect
stream; you write intents that `reduce` state and `postSideEffect`.

The plain 3-type pattern in [mvi.md](mvi.md) and Orbit are not competitors —
Orbit is one implementation of that pattern. State stays an immutable `data
class`, side effects stay a sealed hierarchy, and the same rules from
[architecture.md](architecture.md) (state owner, effect delivery, slicing)
still apply.

## Dependencies

Verify the latest version on Maven Central before adding — coordinates below are
Orbit 12.x.

```kotlin
// group: org.orbit-mvi
implementation("org.orbit-mvi:orbit-core:12.0.0")        // multiplatform core; verify latest: https://repo1.maven.org/maven2/org/orbit-mvi/orbit-core/maven-metadata.xml
implementation("org.orbit-mvi:orbit-viewmodel:12.0.0")   // AndroidX ViewModel integration; verify latest: https://repo1.maven.org/maven2/org/orbit-mvi/orbit-viewmodel/maven-metadata.xml
implementation("org.orbit-mvi:orbit-compose:12.0.0")     // Compose / CMP collectors; verify latest: https://repo1.maven.org/maven2/org/orbit-mvi/orbit-compose/maven-metadata.xml
testImplementation("org.orbit-mvi:orbit-test:12.0.0")    // state/effect test DSL; verify latest: https://repo1.maven.org/maven2/org/orbit-mvi/orbit-test/maven-metadata.xml
```

- `orbit-core` is the only required module and is fully multiplatform (`commonMain`).
- Add `orbit-viewmodel` only where you use AndroidX `ViewModel`.
- Add `orbit-compose` for the `collectAsState` / `collectSideEffect` collectors.

## Core API

| Concept | Orbit | Plain MVI equivalent ([mvi.md](mvi.md)) |
|---|---|---|
| State + effect owner | `OrbitContainerHost<State, SideEffect>` | screen state holder |
| Container factory | `orbitContainer(initialState)` | `MutableStateFlow(State())` + `Channel<Effect>` |
| Handle a user action | `intent { ... }` | a branch of `onEvent()` |
| Update state | `reduce { state.copy(...) }` | `_state.update { it.copy(...) }` |
| One-off command | `postSideEffect(effect)` | `_effect.trySend(effect)` |

State is immutable; `reduce` receives the current `state` and returns the next
one. Side effects are a sealed interface, posted (not stored) — same reasoning
as [mvi.md](mvi.md): a "show snackbar" boolean in state needs consume logic and
replays on config change.

## ViewModel Example

```kotlin
data class CreateItemState(
    val title: String = "",
    val amount: String = "",
    val isSaving: Boolean = false,
    val errors: Map<String, String> = emptyMap(),
) {
    val canSave: Boolean get() = title.isNotBlank() && amount.isNotBlank()
}

sealed interface CreateItemSideEffect {
    data object NavigateBack : CreateItemSideEffect
    data class ShowMessage(val text: String) : CreateItemSideEffect
}

class CreateItemViewModel(
    private val repository: ItemRepository,
) : ViewModel(), OrbitContainerHost<CreateItemState, CreateItemSideEffect> {

    override val container = orbitContainer<CreateItemState, CreateItemSideEffect>(CreateItemState())

    fun onTitleChanged(title: String) = intent {
        reduce { state.copy(title = title, errors = state.errors - "title") }
    }

    fun onAmountChanged(amount: String) = intent {
        reduce { state.copy(amount = amount, errors = state.errors - "amount") }
    }

    fun onBackClick() = intent {
        postSideEffect(CreateItemSideEffect.NavigateBack)
    }

    fun onSaveClick() = intent {
        val errors = buildMap {
            if (state.title.isBlank()) put("title", "Required")
            if (state.amount.isBlank()) put("amount", "Required")
        }
        if (errors.isNotEmpty()) {
            reduce { state.copy(errors = errors) }
            return@intent
        }
        reduce { state.copy(isSaving = true) }
        repository.runCatching { save(state.toItem()) }
            .also { reduce { state.copy(isSaving = false) } }
            .onSuccess { postSideEffect(CreateItemSideEffect.NavigateBack) }
            .onFailure { postSideEffect(CreateItemSideEffect.ShowMessage(it.message ?: "Save failed")) }
    }
}
```

`intent` runs on Orbit's background scope and serializes state updates, so
`reduce` blocks never race — the ordering guarantee that plain MVI gets from a
single `onEvent()` entry point.

## Compose Usage

`orbit-compose` provides collectors on the `ContainerHost`. Verify the exact
names against the orbit-compose docs for your version.

```kotlin
@Composable
fun CreateItemRoute(
    viewModel: CreateItemViewModel,
    snackbar: SnackbarHostState,
    onBack: () -> Unit,
) {
    val state by viewModel.collectAsState()
    viewModel.collectSideEffect { effect ->
        when (effect) {
            CreateItemSideEffect.NavigateBack -> onBack()
            is CreateItemSideEffect.ShowMessage -> snackbar.showSnackbar(effect.text)
        }
    }
    CreateItemScreen(
        state = state,
        onTitleChange = viewModel::onTitleChanged,
        onAmountChange = viewModel::onAmountChanged,
        onSave = viewModel::onSaveClick,
        onBack = viewModel::onBackClick,
    )
}
```

Keep the Route/Screen/leaf split from [mvi.md](mvi.md#ui-rendering-boundary):
the Route collects state and side effects; the Screen is stateless and receives
`state` plus narrow callbacks. Orbit exposes public intent functions, so the
Screen calls named callbacks (`onSaveClick`) rather than dispatching a sealed
`Event` — either boundary style is fine as long as reusable leaves stay unaware
of the container.

## Testing

`orbit-test` runs the container in a controlled scope and asserts the resulting
state sequence and posted side effects:

```kotlin
@Test
fun save_with_blank_fields_sets_errors() = runTest {
    CreateItemViewModel(FakeItemRepository()).test(this) {
        expectInitialState()
        containerHost.onSaveClick()
        expectState { copy(errors = mapOf("title" to "Required", "amount" to "Required")) }
    }
}
```

## When to Reach for Orbit

- You want the MVI contract but not the per-screen `StateFlow` + `Channel` +
  `onEvent()` scaffolding.
- The project targets multiple platforms and needs the same container in
  `commonMain`.
- You value the built-in `orbit-test` DSL for deterministic state/effect assertions.

When a project already has a working plain-MVI base class or convention, do not
introduce Orbit just to standardize — follow the preservation rule in
[architecture.md](architecture.md).

# MVI (Event/State/Effect)

MVI pattern: a sealed `Event` contract processed by a single `onEvent()` entry point. For shared architecture concepts (state owner selection, domain layer, module rules), see [architecture.md](architecture.md).

## The 3 MVI Types

A non-trivial screen using MVI defines 3 types: `Event`, `State`, `Effect`.

### Event

User actions from UI: button clicks, field changes, lifecycle-start, retry, refresh, back press. Events are the **only** input from the UI into the screen state holder, processed by a single `onEvent()` function.

### State

Immutable `data class` that fully describes what the screen should render. Given the same state, the screen always looks the same. One state per screen, owned by the screen state holder via `StateFlow<State>`.

State should be **equality-friendly** — use `data class` with immutable collections. Computed properties (`val hasRequiredFields get() = name.isNotBlank()`) are acceptable for trivial derivations. Store canonical values; derive display values at the UI boundary.

### Effect

One-off UI commands that don't belong in state: navigate, show snackbar, trigger haptic, copy/share, open browser.

**Why effects are not state:** if you model "show snackbar" as a boolean in state, you need "consume" logic to flip it back — a classic source of bugs. Effects fire once and are gone.

## Event Naming

Events should be named from the **user's perspective** — what happened, not what should happen.

| Good | Bad |
|---|---|
| `OnSaveClick` | `SaveCategory` |
| `OnTitleChanged` | `UpdateTitle` |
| `OnRetryClick` | `RetryRequest` |
| `OnBackClick` | `NavigateBack` |

The event describes a user action; the ViewModel decides how to handle it.

## State Modeling

Use an immutable `data class` with computed properties for derivations. For forms, calculators, and avoiding duplicated state, see [state-modeling.md](state-modeling.md).

## Effect Delivery

Default: `Channel<Effect>(Channel.BUFFERED)` with `receiveAsFlow()`. For Channel vs SharedFlow guidance, see [architecture.md](architecture.md#effect-delivery).

## Event Processing Flow

```text
UI gesture / lifecycle signal
    → Event dispatched via onEvent()
    → ViewModel processes the event in a when() block
    → Synchronous events: updateState { copy(...) }
    → Side effects: sendEffect(effect)
    → Async work: viewModelScope.launch { ... }
    → On async completion: updateState { copy(...) } + sendEffect(...)
```

**Key insight:** `onEvent()` is the single decision point. It decides what happens for each event — update state, send an effect, launch async work, or some combination. This keeps all event→reaction logic in one place.

## Screen State Holder Anatomy

A screen state holder using MVI has three responsibilities:

1. **State ownership** — holds `MutableStateFlow<State>`, exposes `StateFlow<State>`
2. **Effect delivery** — holds `Channel<Effect>` or the project's equivalent, exposes `Flow<Effect>`
3. **Event processing** — implements `onEvent()` to handle all events

State is updated via a thread-safe `update` function (e.g., `MutableStateFlow.update { it.copy(...) }` or a wrapper like `updateState { copy(...) }`). Effects are sent via `channel.trySend(effect)`.

## UI Rendering Boundary

### Route composable

Obtains the screen state holder (via `koinViewModel()`, `hiltViewModel()`, manual construction), collects state once via a lifecycle-aware collector, collects effects via `CollectEffect` or equivalent, binds navigation/snackbar/sheet/platform APIs.

### Screen composable

Stateless render function receiving state plus an `onEvent: (Event) -> Unit` callback.

### Leaf composables

Render sub-state, emit specific callbacks, keep only tiny visual-local state. Do not pass `onEvent` to reusable leaves — adapt to specific callbacks.

For how state is collected and sliced across these layers, see [architecture.md](architecture.md#state-collection-and-slicing). For domain and data-layer boundaries, see [architecture.md](architecture.md#domain-layer) and [architecture.md](architecture.md#where-logic-belongs).

## When MVI Is Appropriate

- Project already uses MVI with a base class or convention
- Screen has many user actions and you want them enumerated in one sealed type
- Team values explicit event contracts for debugging, analytics, or time-travel debugging
- You need exhaustive `when` handling for all UI actions
- Complex screens with interrelated state transitions

## MVI Libraries

To adopt the pattern via a library instead of hand-writing the `StateFlow` +
`Channel` + `onEvent()` scaffolding, see [orbit-mvi.md](orbit-mvi.md) — the
Orbit Multiplatform framework (`OrbitContainerHost`, `intent`, `reduce`,
`postSideEffect`).

## Code Examples

### BAD: business logic inside composables

```kotlin
@Composable
fun LoanCalculatorScreen() {
    var amountText by rememberSaveable { mutableStateOf("") }
    var rateText by rememberSaveable { mutableStateOf("") }
    var yearsText by rememberSaveable { mutableStateOf("") }
    // Calculation/validation omitted — belongs in state holder, not here.
    Column {
        OutlinedTextField(value = amountText, onValueChange = { amountText = it })
        OutlinedTextField(value = rateText, onValueChange = { rateText = it })
        OutlinedTextField(value = yearsText, onValueChange = { yearsText = it })
        Text("Monthly payment: …")
        Button(onClick = { /* … */ }) { Text("Calculate") }
    }
}
```

Problems: logic and validation live in the composable, hard to test, and recomposition becomes the execution model.

### GOOD: MVI contract — Event, State, Effect

```kotlin
sealed interface CreateItemEvent {
    data class OnTitleChanged(val title: String) : CreateItemEvent
    data class OnAmountChanged(val amount: String) : CreateItemEvent
    data object OnSaveClick : CreateItemEvent
    data object OnBackClick : CreateItemEvent
}

data class CreateItemState(
    val title: String = "",
    val amount: String = "",
    val isSaving: Boolean = false,
    val errors: Map<String, String> = emptyMap()
) {
    val canSave: Boolean get() = title.isNotBlank() && amount.isNotBlank()
}

sealed interface CreateItemEffect {
    data object NavigateBack : CreateItemEffect
    data class ShowMessage(val text: String) : CreateItemEffect
}
```

### GOOD: ViewModel with onEvent

```kotlin
class CreateItemViewModel(
    private val repository: ItemRepository,
) : ViewModel() {
    private val _state = MutableStateFlow(CreateItemState())
    val state: StateFlow<CreateItemState> = _state.asStateFlow()

    private val _effect = Channel<CreateItemEffect>(Channel.BUFFERED)
    val effect: Flow<CreateItemEffect> = _effect.receiveAsFlow()

    fun onEvent(event: CreateItemEvent) {
        when (event) {
            is CreateItemEvent.OnTitleChanged -> _state.update { it.copy(title = event.title, errors = it.errors - "title") }
            is CreateItemEvent.OnAmountChanged -> _state.update { it.copy(amount = event.amount, errors = it.errors - "amount") }
            CreateItemEvent.OnSaveClick -> save()
            CreateItemEvent.OnBackClick -> _effect.trySend(CreateItemEffect.NavigateBack)
        }
    }

    private fun save() {
        val errors = buildMap {
            if (_state.value.title.isBlank()) put("title", "Required")
            if (_state.value.amount.isBlank()) put("amount", "Required")
        }
        if (errors.isNotEmpty()) { _state.update { it.copy(errors = errors) }; return }
        viewModelScope.launch {
            _state.update { it.copy(isSaving = true) }
            val result = repository.runCatching { save(_state.value.toItem()) }
            _state.update { it.copy(isSaving = false) }
            result
                .onSuccess { _effect.trySend(CreateItemEffect.NavigateBack) }
                .onFailure { _effect.trySend(CreateItemEffect.ShowMessage(it.message ?: "Save failed")) }
        }
    }
}
```

### GOOD: same pattern with a base class or interface

`class CreateItemViewModel(...) : ViewModel(), MviHost<CreateItemEvent, CreateItemState, CreateItemEffect>` keeps the same `onEvent` / `save()` shape, with `updateState` / `sendEffect` provided by the host. For when a base class earns its keep, see [clean-code.md](clean-code.md).

### GOOD: Route/Screen split

```kotlin
@Composable
fun CreateItemRoute(vm: CreateItemViewModel = koinViewModel(), snackbar: SnackbarHostState, onBack: () -> Unit) {
    val state by vm.state.collectAsStateWithLifecycle()
    CollectEffect(vm.effect) { e -> when (e) {
        CreateItemEffect.NavigateBack -> onBack()
        is CreateItemEffect.ShowMessage -> snackbar.showSnackbar(e.text)
    }}
    CreateItemScreen(state, vm::onEvent)
}

@Composable
fun CreateItemScreen(state: CreateItemState, onEvent: (CreateItemEvent) -> Unit) {
    Column {
        OutlinedTextField(state.title, { onEvent(CreateItemEvent.OnTitleChanged(it)) })
        OutlinedTextField(state.amount, { onEvent(CreateItemEvent.OnAmountChanged(it)) })
        Button(onClick = { onEvent(CreateItemEvent.OnSaveClick) }, enabled = !state.isSaving && state.canSave) {
            Text(if (state.isSaving) "Saving..." else "Save")
        }
    }
}
```

### GOOD: Event model for form-heavy screens

```kotlin
enum class FormField { Area, MaterialRate, LaborRate, TaxPercent, Notes }

sealed interface FormEvent {
    data class FieldChanged(val field: FormField, val raw: String) : FormEvent
    data class IncludeWasteChanged(val enabled: Boolean) : FormEvent
    data object SubmitClicked : FormEvent
    data object RetryClicked : FormEvent
    data object ScreenShown : FormEvent
    data object ClearClicked : FormEvent
}
```

Pragmatic default for large forms: specific intent names for screen-level actions, generic `FieldChanged(field, raw)` only when many fields are structurally similar.

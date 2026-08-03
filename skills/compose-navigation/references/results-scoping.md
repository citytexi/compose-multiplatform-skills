# Results and ViewModel Scoping

A `NavKey` from [nav3-core.md](nav3-core.md) is an address, not a payload — the moment a
value needs to travel backward, or a ViewModel needs to outlive the entry
that created it, the back stack stops being the right tool. This reference
covers both: result channels, and how a ViewModel's lifetime is scoped,
shared, or extended around `entry<...>`.

## The Rule

A result is not a route argument. `Details(id)` is fine to carry forward —
the callee needs `id` to render itself. Carrying a value backward through a
key is not fine: to get back to `Home` with a payload you would either push
a *new* `Home` (a second copy of the caller, now duplicated on the stack) or
mutate the existing entry's key in place, which nothing in Navigation 3
does — keys are immutable once an entry exists. A route key is an address:
it tells `NavDisplay` what to render and what that screen needs to render
itself. A result is a message: it tells an already-existing screen that
something happened while it was gone. Navigation 3 ships no result API — no
"previous back stack entry" to write a value into — so the app supplies its
own channel. The three below differ only in what they survive; picking
wrong loses a result silently, or spends a `@Serializable` annotation that
nothing needed.

## Choosing a Result Channel

| Channel | Survives recomposition | Survives configuration change | Survives process death | Delivery |
|---|---|---|---|---|
| Event bus | yes | no | no | Once, to whoever is listening |
| Composition-local store | yes | yes | no | Read as state, repeatedly |
| Saved-state store | yes | yes | yes | Read as state, repeatedly |

Selection rule: a one-shot action ("item deleted, show a snackbar") wants
the event channel; a rendered value ("selected filter") wants a state
channel; a rendered value that must survive a killed app wants the
saved-state channel. If unsure, take saved state — it costs a
`@Serializable` annotation, nothing more.

## Results as Events

A keyed channel map behind a composition local:

```kotlin
class ResultEventBus {
    val channelMap: MutableMap<String, Channel<Any?>> = mutableMapOf()
    inline fun <reified T> getResultFlow(resultKey: String = T::class.toString()) =
        channelMap[resultKey]?.receiveAsFlow()
    inline fun <reified T> sendResult(resultKey: String = T::class.toString(), result: T) {
        channelMap.getOrPut(resultKey) { Channel(capacity = Channel.BUFFERED) }.trySend(result)
    }
}

object LocalResultEventBus {
    private val local: ProvidableCompositionLocal<ResultEventBus?> = compositionLocalOf { null }
    val current: ResultEventBus
        @Composable get() = local.current ?: error("No ResultEventBus has been provided")
    infix fun provides(bus: ResultEventBus): ProvidedValue<ResultEventBus?> = local.provides(bus)
}

@Composable
inline fun <reified T> ResultEffect(
    resultEventBus: ResultEventBus = LocalResultEventBus.current,
    resultKey: String = T::class.toString(),
    crossinline onResult: suspend (T) -> Unit,
) {
    LaunchedEffect(resultKey, resultEventBus.channelMap[resultKey]) {
        resultEventBus.getResultFlow<T>(resultKey)?.collect { onResult(it as T) }
    }
}
```

Usage: the caller declares `ResultEffect<SelectedFilter> { … }`; the callee
calls `sendResult(result = SelectedFilter(...))` then `backStack.removeLastOrNull()`
— named, because `resultKey` is the defaulted parameter that comes first and
a positional call would bind to it instead of `result`.
The channel is `BUFFERED`, not the rendezvous default, so a result sent
before the caller resumes collecting is queued instead of dropped — a
rendezvous channel loses it whenever the sender runs first, which
navigating back always makes possible (the queuing reasoning belongs to the
compose-async skill). A newer, unreleased Navigation 3 build (the Android
recipe repo pins an `androidx.dev` snapshot ahead of any release) ships
this exact shape as a library type —
`androidx.navigation3.runtime.result.{ResultEventBus, LocalResultEventBus,
ResultEffect}` plus `rememberResultEventBusNavEntryDecorator()`, not yet on
the pinned `org.jetbrains.androidx.navigation3` coordinate. Until it ships,
the class above is what you write yourself, provided once above both
screens with `CompositionLocalProvider(LocalResultEventBus provides remember { ResultEventBus() })`.

## Results as State

The same shape, holding state instead of a channel, so a value can be read
repeatedly instead of consumed once:

```kotlin
class ResultStore {
    val resultStateMap: MutableMap<String, MutableState<Any?>> = mutableMapOf()
    inline fun <reified T> getResultState(resultKey: String = T::class.toString()): T? =
        resultStateMap[resultKey]?.value as T?
    inline fun <reified T> setResult(resultKey: String = T::class.toString(), result: T) {
        resultStateMap[resultKey] = mutableStateOf(result)
    }
    inline fun <reified T> removeResult(resultKey: String = T::class.toString()) {
        resultStateMap.remove(resultKey)
    }
}

object LocalResultStore {
    private val local: ProvidableCompositionLocal<ResultStore?> = compositionLocalOf { null }
    val current: ResultStore
        @Composable get() = local.current ?: error("No ResultStore has been provided")
    infix fun provides(store: ResultStore): ProvidedValue<ResultStore?> = local.provides(store)
}
```

`ResultStore` and `LocalResultStore` alone only earn the table's "yes" for
recomposition — a `remember`ed instance dies with whatever composable
created it, same as the event bus. To earn "survives configuration change,"
hold it inside a ViewModel scoped above `NavDisplay`, provided through
`LocalResultStore` instead of a bare `remember`:

```kotlin
class ResultsHolderViewModel : ViewModel() { val resultStore = ResultStore() }
@Composable
fun AppNavDisplay() {
    val resultStore = koinViewModel<ResultsHolderViewModel>().resultStore
    val backStack = rememberNavBackStack(navConfig, Home)
    CompositionLocalProvider(LocalResultStore provides resultStore) {
        NavDisplay(
            backStack = backStack,
            onBack = { backStack.removeLastOrNull() },
            entryProvider = entryProvider {
                entry<Home> { HomeRoute(onOpenDetails = { id -> backStack.add(Details(id)) }, onBack = { backStack.removeLastOrNull() }) }
            },
        )
    }
}
```

The read belongs inside [nav3-core.md](nav3-core.md)'s `HomeRoute`, as one
added line next to `val state by viewModel.state...` — not a second,
contradicting definition of the same function:

```kotlin
val selectedFilter = LocalResultStore.current.getResultState<SelectedFilter?>()
```

`koinViewModel<ResultsHolderViewModel>()` resolves against whatever
`ViewModelStoreOwner` sits above `AppNavDisplay` — the hosting Activity,
which outlives a configuration change independent of Compose (compose-di
covers the module wiring).

State the leak: a result left in the store is still there next time the
caller opens, since `resultStateMap` is keyed by type, not visit — a stale
`SelectedFilter` looks identical to a fresh one. Clear it once consumed, in
the same effect that reads it: `removeResult<SelectedFilter>()`.

## Results as Saved State

The same store, made durable with a `Saver` and `rememberSaveable`:

```kotlin
@Composable
fun rememberResultStore(): ResultStore =
    rememberSaveable(saver = ResultStoreSaver()) { ResultStore() }

private fun ResultStoreSaver(): Saver<ResultStore, *> = Saver(
    save = { it.resultStateMap },
    restore = { ResultStore().apply { resultStateMap.putAll(it) } },
)
```

This swaps the ViewModel-held instance for a `rememberSaveable` one, so it
now survives process death too — through a different, more restrictive
mechanism than [nav3-core.md](nav3-core.md)'s `SavedStateConfiguration`, though. The back
stack's serialization runs every key through an explicit `KSerializer`;
`rememberSaveable`'s default registry instead checks each value with
`canBeSaved()`, whose default implementation — per the SDK's own error
message — "only supports types which can be stored inside the Bundle":
primitives, strings, and nested collections of those. So the `Saver` above,
handing the registry a raw `Map<String, MutableState<Any?>>`, only
round-trips through process death when every value in it is already one of
those primitive-shaped types. `@Serializable` does not clear that bar by
itself — it only satisfies [nav3-core.md](nav3-core.md)'s back-stack machinery, a separate
encoder path this `Saver` does not use. A richer result needs encoding to a
primitive-safe form first (a JSON string via kotlinx.serialization, decoded
back out on read) — the same problem the `serializable` recipe below solves
with an explicit `KSerializer` per call site, which this `Saver` skips.

`android/nav3-recipes`'s `app/src/main/java/com/example/nav3recipes/results/`
has all three worked recipes on the Android side — `event`, `state`, and a
third, `serializable`, using an explicit `KSerializer` instead of trusting
the map; it depends on the unreleased snapshot API above and does not build
against the pinned CMP release today. The `state` recipe — mirrored,
CMP-verified, in `terrakok/nav3-recipes` (branch `master`) as
`sharedUI/src/commonMain/kotlin/com/example/nav3recipes/results/state/ResultStore.kt`
— backs `rememberResultStore()` above, durable through process death only
for the primitive-shaped payloads just described.

## Entry-Scoped ViewModels

With `rememberViewModelStoreNavEntryDecorator()` installed ([nav3-core.md](nav3-core.md)'s
`## Entry Decorators`), a ViewModel obtained inside an entry belongs to that
entry — created the first time the entry composes, cleared the moment that
key is popped. That is the default, and it is almost always what you want:

```kotlin
entry<Details> { key ->
    val viewModel: DetailsViewModel = koinViewModel { parametersOf(key.id) }
    val state by viewModel.state.collectAsStateWithLifecycle()
    DetailsScreen(state = state, onEvent = viewModel::onEvent)
}
```

`parametersOf(key.id)` is how the route's own argument reaches the
ViewModel's constructor through Koin (compose-di covers the module wiring).
The anti-pattern sits one scope up: an app-scoped ViewModel — obtained
above `NavDisplay`, or resolved as a Koin singleton — holding data that is
really per-screen. Open `Details("1")`, then `Details("2")`: entry scoping
gives each its own `DetailsViewModel`; an app-scoped one gives both the
same instance, still showing item 1's data the instant item 2's screen
composes, because nothing ever told it to reset.

## Sharing a ViewModel

Entry scoping breaks down when two *adjacent* entries are really one flow
wearing two screens — a shipping-address step and a payment step that both
need the same `CheckoutViewModel`. Nothing above lets two entries share an
owner; each gets its own by design. `SharedViewModelStoreNavEntryDecorator`
is app-owned code, not a library type — copy it from `terrakok/nav3-recipes`
(branch `master`)'s `sharedUI/src/commonMain/kotlin/com/example/nav3recipes/sharedviewmodel/SharedViewModelStoreNavEntryDecorator.kt`
(`commonMain`, CMP-verified), the same way `BottomSheetSceneStrategy` in [scenes-animations.md](scenes-animations.md) is copied.
Obtain it through its own `rememberSharedViewModelStoreNavEntryDecorator()` factory and install that in place of, not beside,
`rememberViewModelStoreNavEntryDecorator()`: it already provides each entry's own `ViewModelStoreOwner`, plus a parent's when metadata names one:

```kotlin
@Serializable data object CheckoutAddress : AppRoute
@Serializable data object CheckoutPayment : AppRoute
// each new key needs its own subclass(...) line in navConfig:
// subclass(CheckoutAddress::class, CheckoutAddress.serializer())
// subclass(CheckoutPayment::class, CheckoutPayment.serializer())
entry<CheckoutAddress> {
    val viewModel: CheckoutViewModel = koinViewModel()
    CheckoutAddressScreen(state = viewModel.state.value)
}
entry<CheckoutPayment>(
    metadata = SharedViewModelStoreNavEntryDecorator.parent(CheckoutAddress.toString())
) {
    val shared: CheckoutViewModel = koinViewModel(viewModelStoreOwner = LocalSharedViewModelStoreOwner.current)
    CheckoutPaymentScreen(state = shared.state.value)
}
```

`.parent(...)` takes the parent entry's content key, which defaults to
`key.toString()` — why a plain `data object` route needs no extra wiring.
Install it on `NavDisplay` as `entryDecorators = listOf(rememberSaveableStateHolderNavEntryDecorator(),
rememberSharedViewModelStoreNavEntryDecorator())`, in place of the
ViewModel-store decorator per the paragraph above. The caution: prefer hoisting shared state into the flow's parent
composable, or a repository, before reaching for a shared owner — it makes
the two screens' lifetimes implicit, `CheckoutPayment` silently depending on
`CheckoutAddress` staying on the stack.

## Retaining Values

An entry that's on the back stack but not currently composed — behind
another pane in a two-pane scene, or (on Android) a host Activity destroyed
and recreated for a configuration change — loses any plain `remember` value
inside it, since `remember`'s contract is scoped to composition, not back
stack presence. `retain { … }`, from `androidx.compose.runtime:runtime-retain`,
is a `remember` replacement whose value survives exactly that gap. JetBrains
does not publish this artifact under `org.jetbrains.compose.runtime` — pin
the androidx coordinate directly, the same situation [nav3-core.md](nav3-core.md)'s
`## Dependencies` describes for `navigation3-runtime`. Stable since 1.10.0,
it publishes real Android, desktop, iOS, `wasmJs`, and Kotlin/Native
targets, not the `jvmstubs`/`linuxx64stubs` placeholders [nav3-core.md](nav3-core.md) warns
about for `navigation3-ui`:

```toml
[libraries]
compose-runtime-retain = { module = "androidx.compose.runtime:runtime-retain", version = "1.11.4" } # verify latest
```

Usage inside an entry:
```kotlin
entry<Details> { key ->
    val viewModel: DetailsViewModel = koinViewModel { parametersOf(key.id) }
    val thumbnail = retain { renderThumbnail(key.id) }
    DetailsScreen(state = viewModel.state.value, thumbnail = thumbnail, onEvent = viewModel::onEvent)
}
```

`retain` needs a store to write into, the same way
`rememberViewModelStoreNavEntryDecorator()` needs a `ViewModelStoreOwner`: an
entry decorator scoping one `RetainedValuesStore` per entry, backed by
`RetainedValuesStoreRegistry`. That decorator is app-owned — both recipe
repos hand-roll `rememberRetainedValuesStoreNavEntryDecorator()`;
`commonMain`-verified, it's `terrakok/nav3-recipes`'s (branch `master`)
`sharedUI/src/commonMain/kotlin/com/example/nav3recipes/retain/RetainActivity.kt`,
wrapping `registry.LocalRetainedValuesStoreProvider(entry.contentKey) { entry.Content() }`
and clearing the child store from `onPop`. Add it alongside the pair from
[nav3-core.md](nav3-core.md)'s `## Entry Decorators`; never retain a platform UI handle or
anything referencing one, since a retained value outlives its composable.

`## Entry-Scoped ViewModels` already gives you a value that survives this
gap for free, whenever the value belongs in a ViewModel — its store is tied
to back stack presence, not composition. Reach for `retain` when the value
doesn't belong there: an expensive-to-recompute `remember`ed side value
that isn't state a `*Route` reads or a repository would own. If a `*Route`
already reads it through a ViewModel, keep it there — a second retention
mechanism for the same value is redundant.

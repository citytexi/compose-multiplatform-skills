# Modularizing Navigation

A single `entryProvider { }` in the app module — one `entry<...>` call per
screen — is the shape every earlier reference in this skill uses, and it does
not scale past a small app. This reference covers the fix: features
contribute their own entries, and the app module aggregates them without
naming a single screen. `navigationState`/`navigator` below are the
`NavigationState`/`Navigator` from [tabs-flows.md](tabs-flows.md); the
single-stack shape they replace is in [nav3-core.md](nav3-core.md).

## The Problem

Every `entry<Home> { }`, `entry<Details> { }`, `entry<Profile> { }` call in
one `entryProvider { }` block is a compile-time dependency on that screen's
composable and, transitively, its ViewModel and everything the ViewModel
depends on. Put them all in the app module and the app module depends on
every feature's implementation — `feature-details` cannot be built, tested,
or even compiled in isolation, because the app module that assembles the
graph won't build without it. Nothing about `entry<...>` requires this; the
coupling is where the call is written, not what it does.

The fix is inversion: each feature exposes a function that adds its own
entries to an `EntryProviderScope`, and the app module calls that function
without ever writing `entry<Details>` itself. The app module still knows
`feature-details` exists — it still has to call something — but it no longer
knows what `DetailsRoute` looks like or what `DetailsViewModel` needs.

## api / impl Split

```text
feature-details/
  api/    DetailsRoute.kt      -- @Serializable data class Details(val id: String) : AppRoute
  impl/   DetailsScreen.kt     -- the composable
          DetailsViewModel.kt
          DetailsEntry.kt      -- the entry builder
```

Rule: `api` holds route keys and nothing else — no composable, no
ViewModel, no repository. That is what lets another feature navigate to
`feature-details` by depending on `feature-details:api`, a module with no UI
in it to pull in. `impl` depends on its own `api` plus any other feature's
`api` — never another feature's `impl`. That second half is what actually
prevents the graph from re-tangling: if `feature-search:impl` depended on
`feature-details:impl` to reuse one composable, the two features would be
back to knowing about each other's implementations, just one hop removed
from the app module. This matches the module boundary rules in the
compose-architecture skill.

One coupling this split does not remove: `navConfig`'s
`subclass(...)` registration (nav3-core.md's `## Route Keys`) needs every
`AppRoute` subclass, so the app module still lists each feature's key type
there. That is a much smaller dependency than before — a `Details::class`
reference into `feature-details:api`, not a `DetailsRoute` composable call
into `feature-details:impl` — but it is real, and no pattern below removes
it.

## Entry Builder Functions

The framework-free pattern — an extension function on `EntryProviderScope`,
which works with any DI or none:

```kotlin
// feature-details/impl
fun EntryProviderScope<NavKey>.detailsEntry(navigator: Navigator) {
    entry<Details> { key ->
        DetailsRoute(id = key.id, onBack = { navigator.goBack() })
    }
}

// feature-home/impl
fun EntryProviderScope<NavKey>.homeEntry(navigator: Navigator) {
    entry<Home> {
        HomeRoute(
            onOpenDetails = { id -> navigator.navigate(Details(id)) },
            onBack = { navigator.goBack() },
        )
    }
}

// feature-profile/impl — ProfileRoute() takes no navigation callback
fun EntryProviderScope<NavKey>.profileEntry() {
    entry<Profile> { ProfileRoute() }
}

// app module — no feature's composable or ViewModel is named here
val entryProvider = entryProvider {
    homeEntry(navigator)
    detailsEntry(navigator)
    profileEntry()
}

NavDisplay(
    entries = navigationState.toEntries(entryProvider),
    onBack = { navigator.goBack() },
)
```

`EntryProviderScope<T : Any>` is the real receiver type `entry<...>` is
declared on, in `androidx.navigation3.runtime` — the same package
nav3-core.md and scenes-animations.md's section builders already extend, so
`detailsEntry` above is not a new shape, just one written in a feature
module instead of inline in the app module's `entryProvider { }`. (A
single-stack app passes `backStack = …` and `entryProvider = …` to
`NavDisplay` instead of `entries`/`toEntries` — see [nav3-core.md](nav3-core.md).)

The app module still lists the features — it calls `homeEntry`,
`detailsEntry`, `profileEntry` by name — but it never sees a screen
composable or a ViewModel, and that list is the only thing that changes
when a feature is added or removed. That is usually enough. The two
DI-aggregated variants below remove even that list, at the cost of a
framework dependency and, for Koin, an experimental API.

## Koin Navigation DSL

Koin is this library's default DI framework (see the compose-di skill for
Koin setup itself). Its Navigation 3 integration ships as its own artifact,
`io.insert-koin:koin-compose-navigation3`, pinned to the same `4.2.2` line
as compose-di's Koin reference:

```toml
[libraries]
koin-compose-navigation3 = { module = "io.insert-koin:koin-compose-navigation3", version.ref = "koin" } # verify latest
```

It genuinely is multiplatform: the module's own build config targets
Android, JVM, iOS (x64/arm64/simulatorArm64), macOS, JS, and wasmJs. Two
pieces matter:

- `Module.navigation<T> { }` / `ScopeDSL.navigation<T> { }`, in package
  `org.koin.dsl.navigation3` — registers one route's entry the same way
  `single { }` registers a dependency.
- `koinEntryProvider<T>()`, a `@Composable` function in `commonMain`, package
  `org.koin.compose.navigation3` — collects every `navigation<T>` registered
  in the current Koin scope and aggregates them into one entry provider.

Both are still marked `@KoinExperimentalAPI`.

`navigation<T> { }`'s definition lambda has no parameter slot for
`Navigator` the way the framework-free builders above take one — so the
common module also defines a composition-local bridge, the same shape
`LocalResultEventBus`/`LocalResultStore` already use in
[results-scoping.md](results-scoping.md). Read `LocalNavigator.current` in
the composable body and capture the result in a `val` before building
`onBack` — a composition local cannot be read inside a plain `() -> Unit`
callback, only inside a `@Composable` one, no matter how deeply that
callback is nested inside composable code:

```kotlin
// common module
object LocalNavigator {
    private val local: ProvidableCompositionLocal<Navigator?> = compositionLocalOf { null }
    val current: Navigator
        @Composable get() = local.current ?: error("No Navigator has been provided")
    infix fun provides(navigator: Navigator): ProvidedValue<Navigator?> = local.provides(navigator)
}

// feature-details/impl
@OptIn(KoinExperimentalAPI::class)
val detailsModule = module {
    navigation<Details> { key ->
        val navigator = LocalNavigator.current
        DetailsRoute(id = key.id, onBack = { navigator.goBack() })
    }
}

// app module — one include per feature module; no feature's composable is named here
val appModule = module { includes(detailsModule) }

@OptIn(KoinExperimentalAPI::class)
@Composable
fun AppNavDisplay() {
    val navigationState = rememberNavigationState(
        startRoute = Home,
        topLevelRoutes = setOf(Home, Search, Profile),
    )
    val navigator = remember { Navigator(navigationState) }
    val entryProvider = koinEntryProvider<NavKey>()
    CompositionLocalProvider(LocalNavigator provides navigator) {
        NavDisplay(
            entries = navigationState.toEntries(entryProvider),
            onBack = { navigator.goBack() },
        )
    }
}
```

`Navigator` is deliberately never registered in Koin. A root `single` has no
scope shorter than the whole process here, so it would cache the first
`NavigationState` forever — stale the moment `AppNavDisplay` recomposes from
scratch, exactly the churn tabs-flows.md's `## Conditional Flows` describes
for an auth swap. Koin does ship a composition-bound scope (`KoinScope`,
package `org.koin.compose.scope`), but wiring `Navigator` through it only
re-derives the lifetime `AppNavDisplay` already owns. This reference keeps
`Navigator` composition-owned instead, and lets Koin do only what
`koinEntryProvider()` is for: aggregating entries. `LocalNavigator` is the
bridge, not a DI registration.

The Android recipe this is adapted from —
`android/nav3-recipes`'s `app/src/main/java/com/example/nav3recipes/modular/koin/`
— does not call `koinEntryProvider()` at all — its `KoinModularActivity`
calls `ComponentCallbacks.getEntryProvider()`, an eager, Android-only
variant in `org.koin.androidx.compose.navigation3`, paired with
`AndroidScopeComponent` and `activityRetainedScope { }` so entries can pull
activity-scoped dependencies. `AndroidScopeComponent` and
`activityRetainedScope` are Android-only APIs and cannot appear in
`commonMain`; a CMP app uses the multiplatform `koinEntryProvider()`
composable above instead, backed by plain `Module.navigation<T>`
registrations rather than a scope tied to an Activity. Full reference:
`https://insert-koin.io/docs/reference/koin-compose/navigation3`.

## Metro

Metro, the compile-time alternative documented in the compose-di skill, has
no Navigation 3-specific artifact — there is no `koinEntryProvider()`
equivalent to import. Instead, the app defines an ordinary typealias and
lets Metro's generic set-multibinding aggregate it:

```kotlin
// common module — app-defined, not a Metro type; LocalNavigator is the same
// bridge object the Koin section above defines
typealias EntryProviderInstaller = EntryProviderScope<NavKey>.() -> Unit

// feature-details/impl
@ContributesTo(AppScope::class)
@BindingContainer
object DetailsEntryModule {
    @IntoSet
    @Provides
    fun provideDetailsEntry(): EntryProviderInstaller = {
        entry<Details> { key ->
            val navigator = LocalNavigator.current
            DetailsRoute(id = key.id, onBack = { navigator.goBack() })
        }
    }
}

// app module — the graph exposes the aggregated set, no feature named
@DependencyGraph(AppScope::class)
interface AppGraph {
    val entryProviderInstallers: Set<EntryProviderInstaller>
}

@Composable
fun AppNavDisplay(graph: AppGraph) {
    val navigationState = rememberNavigationState(
        startRoute = Home,
        topLevelRoutes = setOf(Home, Search, Profile),
    )
    val navigator = remember { Navigator(navigationState) }
    val entryProvider = entryProvider {
        graph.entryProviderInstallers.forEach { install -> this.install() }
    }
    CompositionLocalProvider(LocalNavigator provides navigator) {
        NavDisplay(
            entries = navigationState.toEntries(entryProvider),
            onBack = { navigator.goBack() },
        )
    }
}
```

`@IntoSet` is the annotation doing the aggregating here — distinct from
`@ContributesBinding` in compose-di's Metro reference, which binds one
implementation to one interface rather than accumulating contributions into
a collection. `@ContributesTo` + `@BindingContainer` puts the `@Provides`
function on the graph the same way any other binding gets there; `@IntoSet`
is what collects one instance per feature module into
`Set<EntryProviderInstaller>` instead of erroring on a duplicate binding.
`Navigator` is not an `AppGraph` accessor, for the same reason it is not a
Koin registration above — composition owns it. Adapted from
`android/nav3-recipes`'s
`metroapp/src/main/java/com/example/nav3recipes/modular/metro/`,
which also injects `MetroModularActivity` as an `Activity` via
`dev.zacsweers.metrox.android` and provides `Navigator` as a graph
`@SingleIn(ActivityScope::class)` singleton — both Android-only, without a
verified `commonMain` equivalent, so neither is in the shape above.

Hilt is Android-only and is out of scope for this library; use Koin or Metro.

## Scoping the Navigator

The back stack is created by and lives in composition — `rememberNavBackStack`
and `rememberNavigationState` are both `@Composable` functions. A `Navigator`
registered as a DI singleton must therefore receive that state rather than
create it itself, or its back stack outlives the composition that renders
it and every screen built against the singleton starts reading state that
no `NavDisplay` is observing anymore.

Two workable arrangements:

- Create `navigationState` in composition and pass it into a
  locally-`remember`ed `Navigator` — the simplest option, and what
  [tabs-flows.md](tabs-flows.md) shows. `## Entry Builder Functions` takes
  `navigator` as a parameter; the Koin and Metro sections above reach the
  same instance through `LocalNavigator` instead, since neither DSL has a
  parameter slot for it. Both are this one arrangement — composition owns
  the `Navigator`, DI never does.
- Hold `Navigator` itself in a retained DI scope, so `NavDisplay` reads
  `navigator.state` directly — what the upstream Android recipes do (Koin's
  `activityRetainedScope`, Metro's `@SingleIn(ActivityScope::class)`), both
  bound to an Activity's lifecycle with no verified `commonMain` equivalent
  (`## Koin Navigation DSL` above covers why a root `single` fails here).
  This reference does not show a DI-retained `Navigator` on a CMP target.

Prefer the first — every pattern in this file already uses it. Reach for a
retained scope only once a concrete platform trigger needs to navigate from
outside composition (a push notification, a platform deep-link listener) —
and verify whatever scope mechanism resolves that against the CMP artifact
you pin, the same way this reference had to for the DSLs above.

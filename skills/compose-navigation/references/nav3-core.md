# Navigation 3 Core

Navigation 3 inverts Nav 2 — the app owns the back stack as Compose state and
`NavDisplay` renders it. State ownership is explicit, which is why it fits the
MVI stance of this library. For scenes and transitions see
[scenes-animations.md](scenes-animations.md); for tabs see
[tabs-flows.md](tabs-flows.md); for deep links see
[deeplinks-platform.md](deeplinks-platform.md).

Official docs: https://developer.android.com/guide/navigation/navigation-3 and
https://kotlinlang.org/docs/multiplatform/compose-navigation-3.html. Recipes:
https://github.com/android/nav3-recipes (Android only) and
https://github.com/terrakok/nav3-recipes (Compose Multiplatform, including
iOS, desktop, and web targets).

## Building Blocks

| Type | Role |
|---|---|
| `NavKey` | Marker interface for a serializable destination key |
| `NavBackStack<NavKey>` | The `SnapshotStateList` your app owns and mutates |
| `NavEntry` | A key plus its composable content plus a metadata map |
| `NavDisplay` | Observes the back stack, resolves entries, picks a scene, renders |
| `Scene` / `SceneStrategy` | Decides the layout an entry set is rendered in |
| `NavEntryDecorator` | Wraps every entry with a cross-cutting concern |

```text
event callback / effect handler
  -> backStack.add(key) or backStack.removeLastOrNull()
  -> NavDisplay observes the change
  -> entryProvider resolves key -> NavEntry
  -> SceneStrategy picks a Scene
  -> Scene renders the entry content
```

## Dependencies

> `androidx.navigation3:navigation3-ui` publishes Android plus `jvmstubs` and
> `linuxx64stubs` variants only — the stubs do not function. In a Compose
> Multiplatform project take the UI layer from
> `org.jetbrains.androidx.navigation3:navigation3-ui` instead. Package names
> are identical (`androidx.navigation3.ui.*`), so only the Gradle coordinate
> differs and the mistake compiles. `androidx.navigation3:navigation3-runtime`
> genuinely is multiplatform on the androidx coordinates, which is exactly
> what hides the mistake — the runtime import resolves everywhere, and only
> the UI layer silently degrades to a non-functional stub.

All three versions below are read together off one anchor — Compose
Multiplatform `1.11.1`'s Dependencies section (see the compose-dependencies
skill's `version-lock.md`) — never chosen independently. A stable Lifecycle
`2.11.0` and a `1.3.0-beta02` Material3 Adaptive both exist on Maven Central,
and a later, still-prerelease Compose Multiplatform build
(`1.12.0-beta03`, Dependencies section checked 2026-08-04) does name that
exact Lifecycle/Adaptive pair — but that same build moves Navigation3 to
`1.2.0-alpha02`, not the `1.1.1` this file pins, and no release names
`1.1.1` / `2.11.0` / `1.3.0-beta02` together as one coordinated set. So the
pins below stay on the `1.11.1` anchor throughout, not the newer, still-beta
one that happens to name two of the three numbers.

```toml
[versions]
nav3 = "1.1.1"              # verify latest: https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.11.1
lifecycle = "2.11.0-beta01" # verify latest: https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.11.1
adaptive = "1.3.0-alpha07"  # verify latest: https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.11.1

[libraries]
nav3-ui = { module = "org.jetbrains.androidx.navigation3:navigation3-ui", version.ref = "nav3" }
lifecycle-viewmodel-nav3 = { module = "org.jetbrains.androidx.lifecycle:lifecycle-viewmodel-navigation3", version.ref = "lifecycle" }
lifecycle-viewmodel-compose = { module = "org.jetbrains.androidx.lifecycle:lifecycle-viewmodel-compose", version.ref = "lifecycle" }
lifecycle-runtime-compose = { module = "org.jetbrains.androidx.lifecycle:lifecycle-runtime-compose", version.ref = "lifecycle" }
adaptive-navigation3 = { module = "org.jetbrains.compose.material3.adaptive:adaptive-navigation3", version.ref = "adaptive" }
```

```kotlin
commonMain.dependencies {
    implementation(libs.nav3.ui)                    // brings navigation3-runtime transitively
    implementation(libs.lifecycle.viewmodel.compose)
    implementation(libs.lifecycle.viewmodel.nav3)   // rememberViewModelStoreNavEntryDecorator
    implementation(libs.lifecycle.runtime.compose)  // collectAsStateWithLifecycle(), used in The MVI Boundary below
    implementation(libs.adaptive.navigation3)       // only if you use Material adaptive scenes
}
```

The `kotlinx-serialization` Gradle plugin is required too — every route key
below is `@Serializable`. Every pinned version above carries a verify-latest
pointer to its release page — written as `// verify latest` in Kotlin DSL
elsewhere in this skill, `#` here since TOML has no `//` comment.

## Route Keys

```kotlin
@Serializable sealed interface AppRoute : NavKey
@Serializable data object Home : AppRoute
@Serializable data class Details(val id: String) : AppRoute
@Serializable data object Search : AppRoute
@Serializable data object Profile : AppRoute
@Serializable data object Settings : AppRoute
```

Group routes under one `@Serializable sealed interface` so an exhaustive
`when` over the whole route set is possible and so polymorphic registration
(below) is mechanical — you walk the sealed hierarchy once instead of hunting
for stray `NavKey` implementations. Keep key payloads small and primitive — a
key is an address, not a data transfer object. On CMP prefer `String`/`Int`
fields, and supply a `KSerializer` if a domain type must appear in a key.

## Creating the Back Stack

```kotlin
// 1. Prototyping only — lost on configuration change.
val backStack = remember { mutableStateListOf<NavKey>(Home) }

// 2. Android only, and only from an androidMain source file — survives
//    configuration change and process death via a reflection-based serializer.
val backStack = rememberNavBackStack(Home)

// 3. Every target, desktop JVM included — REQUIRED outside androidMain, and
//    what commonMain code should use unconditionally.
val backStack = rememberNavBackStack(navConfig, Home)
```

Tier 2's no-argument-config overload is declared only in `androidMain` — it
does not exist in `commonMain`, so shared code that calls it fails to
resolve no matter which platform ultimately runs it; only a file compiled
into `androidMain` may call it. Tier 3 needs a `SavedStateConfiguration`, and
it is not optional there: the reflective overload is published only in
`androidMain`, so every other target — including desktop JVM, which has
reflection but never gets the overload — must register each `NavKey`
subclass explicitly. Since this skill's code lives in `commonMain`, use tier
3 unconditionally rather than branching on platform.

```kotlin
val navConfig = SavedStateConfiguration {
    serializersModule = SerializersModule {
        polymorphic(NavKey::class) {
            subclass(Home::class, Home.serializer())
            subclass(Details::class, Details.serializer())
            subclass(Search::class, Search.serializer())
            subclass(Profile::class, Profile.serializer())
            subclass(Settings::class, Settings.serializer())
        }
    }
}
```

Forgetting one subclass fails at restore time, not at build time — a screen
deep in the stack silently disappears after the app is killed and reopened.
Register every key the moment you declare it.

## NavDisplay

```kotlin
@Composable
fun AppNavDisplay() {
    val backStack = rememberNavBackStack(navConfig, Home)

    NavDisplay(
        backStack = backStack,
        onBack = { backStack.removeLastOrNull() },
        entryDecorators = listOf(
            rememberSaveableStateHolderNavEntryDecorator(),
            rememberViewModelStoreNavEntryDecorator(),
        ),
        entryProvider = entryProvider {
            entry<Home> {
                HomeRoute(
                    onOpenDetails = { id -> backStack.add(Details(id)) },
                    onBack = { backStack.removeLastOrNull() },
                )
            }
            entry<Details> { key ->
                DetailsRoute(id = key.id, onBack = { backStack.removeLastOrNull() })
            }
        },
    )
}
```

`onBack` is not optional in practice — without it the system back gesture
(Android predictive back, desktop Esc handling you wire yourself) does
nothing, and users get stuck on screens they should be able to leave.

The `entryProvider { }` DSL above builds a lookup by key type. There is also a
manual form, which gives you a `when` and therefore an `else` branch — useful
when routes are contributed dynamically (a feature module registers its own
keys) and the compiler cannot see the full set at this call site:

```kotlin
entryProvider = { key ->
    when (key) {
        is Home -> NavEntry(key) {
            HomeRoute(
                onOpenDetails = { id -> backStack.add(Details(id)) },
                onBack = { backStack.removeLastOrNull() },
            )
        }
        is Details -> NavEntry(key) { DetailsRoute(id = key.id, onBack = { backStack.removeLastOrNull() }) }
        else -> error("Unknown route: $key")
    }
}
```

## Entry Decorators

| Decorator | Preserves | Symptom when missing |
|---|---|---|
| `rememberSaveableStateHolderNavEntryDecorator()` | `rememberSaveable` state per entry | Scroll position and form text reset when returning to a screen |
| `rememberViewModelStoreNavEntryDecorator()` | A `ViewModelStoreOwner` per entry | ViewModels are never cleared, or are shared across screens by accident |

Pass both, in this order, on every `NavDisplay` — as the `entryDecorators`
argument on the `backStack`-based overload shown above, or inside
`rememberDecoratedNavEntries` on the `entries`-based overload, which has no
`entryDecorators` parameter of its own (see [tabs-flows.md](tabs-flows.md)
and [di-modularization.md](di-modularization.md) for that path). They are
cheap and their absence is silent — nothing crashes, a screen just loses
state or leaks a ViewModel. One documented exception:
[results-scoping.md](results-scoping.md)'s `SharedViewModelStoreNavEntryDecorator`
installs in place of, not beside, `rememberViewModelStoreNavEntryDecorator()`.

## The MVI Boundary

`HomeRoute`, called from `entry<Home>` above, is where the back stack meets
the ViewModel — three layers, three jobs. The `entry<Home>` lambda in
`## NavDisplay` is the only place touching `backStack`; it decided what
`onOpenDetails` and `onBack` do. `HomeRoute` never sees `backStack` — it
obtains the ViewModel, collects state, and translates each `HomeEffect` into
a call to the matching callback it was handed. `HomeScreen` and every leaf
below it see neither the back stack nor a navigation callback; they render
`state` and emit `onEvent`.

```kotlin
sealed interface HomeEffect {
    data class NavigateToDetails(val id: String) : HomeEffect
    data object NavigateBack : HomeEffect
}

@Composable
fun HomeRoute(onOpenDetails: (String) -> Unit, onBack: () -> Unit) {
    val viewModel: HomeViewModel = koinViewModel()
    val state by viewModel.state.collectAsStateWithLifecycle()

    CollectEffect(viewModel.effect) { effect ->
        when (effect) {
            is HomeEffect.NavigateToDetails -> onOpenDetails(effect.id)
            HomeEffect.NavigateBack -> onBack()
        }
    }

    HomeScreen(state = state, onEvent = viewModel::onEvent)
}
```

`HomeEffect.NavigateToDetails` maps to `onOpenDetails: (String) -> Unit`;
`HomeEffect.NavigateBack` maps to `onBack: () -> Unit` the same way — one
effect, one callback call, nothing touches `backStack` here.
`collectAsStateWithLifecycle` needs `lifecycle-runtime-compose` (see
`## Dependencies`) or this function fails to resolve.

Navigation happens in an effect handler or event callback, never a composable
body — a body reruns on every recomposition, so a callback there fires
repeatedly. Guard the click where it happens, wrapping the event it sends,
with `dropUnlessResumed { }` so a double tap can't dispatch it twice:

```kotlin
// Inside HomeScreen — still only ever calls onEvent, never a nav callback.
Button(onClick = dropUnlessResumed { onEvent(HomeEvent.OnDetailsClick(id)) }) { Text("Open") }
```

`dropUnlessResumed` (from `androidx.lifecycle.compose`) only wraps a
no-argument `() -> Unit`, so it guards the click, not the eventual
`backStack.add`; `HomeEvent` is illustrative. `CollectEffect` and the
ViewModel contract (`state`, `effect`, `onEvent`) come from the
compose-architecture skill — assumed here, not redefined.

## Back Stack Cookbook

```kotlin
backStack.add(Details("42"))                                  // forward
backStack.removeLastOrNull()                                  // back
backStack.removeAll { it is Details }                         // drop every Details…
backStack.add(Details(newId))                                 // …then push a fresh one
backStack.clear(); backStack.addAll(listOf(Home, Details(id))) // synthetic stack, e.g. a deep link
while (backStack.size > 1) backStack.removeLast()             // pop to root
```

Multi-tab back stacks need a different structure, one stack per tab — see [tabs-flows.md](tabs-flows.md).

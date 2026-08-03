# Nav 2 and Migration to Nav 3

## When This File Applies

You are working in a codebase already built on `NavHost` and `NavController`.
Navigation 2 is **not deprecated** — Google ships it alongside Navigation 3,
not in place of it. Migrating is optional, and forcing a migration onto a
working codebase is out of scope for a feature request; do it only when the
app already needs what Nav 3 buys (state-owned back stacks, adaptive scenes,
multi-pane layouts — see [nav3-core.md](nav3-core.md)). For new Compose
Multiplatform code with no existing graph to preserve, start at
[nav3-core.md](nav3-core.md) instead of reading further here.

## Dependency

On Compose Multiplatform the artifact is
`org.jetbrains.androidx.navigation:navigation-compose`, not the
Google-published `androidx.navigation:navigation-compose` coordinate. Package
names match (`androidx.navigation.*`), so importing the wrong one still
compiles — worth checking the actual mechanism rather than assuming it
matches [nav3-core.md](nav3-core.md)'s `## Dependencies` split, because it
doesn't. The Google coordinate for Nav 2 is not one bundled artifact: its
`navigation-compose` module genuinely depends on separately published
`androidx.navigation:navigation-common` and `androidx.navigation:navigation-runtime`
artifacts, confirmed by reading all three modules' Gradle metadata — the
same runtime/UI-shaped split Nav 3 has. The difference from Nav 3 is what
each layer publishes: for Nav 3, `navigation3-runtime` is genuinely
multiplatform on the androidx coordinate and only `navigation3-ui` degrades
to a stub (see nav3-core.md's `## Dependencies`). For Nav 2, `navigation-common`,
`navigation-runtime`, *and* `navigation-compose` all carry the identical
Android-only pattern — `androidJvm` release variants plus non-functional
`jvm` and `linuxX64` stub variants, on every one of the three. Pulling
`navigation-runtime` alone does not sidestep the problem; there is no layer
of the Google-published stack that isn't Android-only. The JetBrains
coordinate is the one that is genuinely multiplatform: Android, desktop
JVM, JS, wasmJs, macOS, and iOS targets, confirmed against its Gradle module
metadata.

```toml
[versions]
nav2 = "2.9.2"   # verify latest: https://repo1.maven.org/maven2/org/jetbrains/androidx/navigation/navigation-compose/maven-metadata.xml

[libraries]
navigation-compose = { module = "org.jetbrains.androidx.navigation:navigation-compose", version.ref = "nav2" }
```

The `kotlinx-serialization` Gradle plugin is required too, same as Nav 3 —
every route below is `@Serializable`.

## The Nav 2 Working Set

Route types are `@Serializable`, same as Nav 3, but **do not implement
`NavKey`** — that interface belongs to Nav 3's back stack and does not exist
on this side; carrying it backward into Nav 2 code, or forgetting to add it
when you eventually convert a route, is the single most common slip between
the two files.

```kotlin
@Serializable data object Home
@Serializable data class Details(val id: String)

@Composable
fun AppNavHost() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = Home) {
        composable<Home> {
            HomeRoute(
                onOpenDetails = { id -> navController.navigate(Details(id)) },
                onBack = { navController.navigateUp() },
            )
        }
        composable<Details> { entry ->
            val details: Details = entry.toRoute()
            DetailsRoute(id = details.id, onBack = { navController.navigateUp() })
        }
    }
}
```

`composable<T>` and `entry.toRoute<T>()` are the type-safe pair — string
routes (`composable("details/{id}")` with a manual `NavType` bundle) are the
legacy shape from before type-safe routing shipped; do not introduce a new
one in an existing graph.

**Navigation options** — `popUpTo` clears entries back to (and, with
`inclusive`, including) the given route; `launchSingleTop` skips pushing a
duplicate on top of an identical entry:

```kotlin
navController.navigate(Details(id)) {
    popUpTo<Home> { inclusive = true }
    launchSingleTop = true
}
```

**`navigateUp` versus `popBackStack`** — `navigateUp` honors Up navigation,
which can differ from Back when the current entry has no real predecessor on
the stack (arriving via a deep link, for instance); `popBackStack` always
pops the actual back stack entry:

```kotlin
navController.navigateUp()    // Up: may synthesize a parent the stack never had
navController.popBackStack()  // Back: pops the real top entry, nothing more
```

**Nested graphs** group a subsection of routes under one entry point, so the
parent graph only ever sees the graph key, not every screen inside it:

```kotlin
navigation<SettingsGraph>(startDestination = Settings) {
    composable<Settings> { SettingsRoute() }
}
```

**Tab selection** reads the current destination as state and restores each
tab's saved state on return, instead of rebuilding it from scratch:

```kotlin
val currentDestination = navController.currentBackStackEntryAsState().value?.destination
navController.navigate(Search) {
    popUpTo(navController.graph.findStartDestination().id) { saveState = true }
    launchSingleTop = true
    restoreState = true
}
```

**Results** travel backward through the previous entry's `SavedStateHandle`,
set before popping and read by the entry that regains focus:

```kotlin
// Details, before popping:
navController.previousBackStackEntry?.savedStateHandle?.set("result", value)
navController.popBackStack()
// Home, reading it back as state:
val result = navController.currentBackStackEntry
    ?.savedStateHandle?.getStateFlow<String?>("result", null)?.collectAsState()?.value
```

**Deep links** attach to a destination declaratively, instead of the app
parsing a URI itself:

```kotlin
composable<Details>(deepLinks = listOf(navDeepLink<Details>(basePath = "https://example.com/item"))) { entry ->
    val details: Details = entry.toRoute()
    DetailsRoute(id = details.id, onBack = { navController.navigateUp() })
}
```

## Concept Mapping

| Nav 2 | Nav 3 |
|---|---|
| `NavController` owns the back stack | Your app owns a `NavBackStack` |
| `NavHost` renders destinations | `NavDisplay` renders entries from the back stack |
| `composable<T> { }` | `entry<T> { }` in an `entryProvider` |
| `navController.navigate(Details(id))` | `backStack.add(Details(id))` |
| `navigateUp()` / `popBackStack()` | `backStack.removeLastOrNull()` |
| `popUpTo(Home) { inclusive = true }` | List operations on the back stack |
| Nested `navigation<Graph>` | No graph type — entries resolve through the provider |
| `navDeepLink` attached per destination | You parse URIs and build the stack — see [deeplinks-platform.md](deeplinks-platform.md) |
| Per-destination ViewModel (the default `koinViewModel()` scope) | Entry-scoped ViewModel via the decorator — see [results-scoping.md](results-scoping.md) |
| Graph-scoped ViewModel via `getBackStackEntry()` — one instance shared across a nested graph | Shared-owner decorator, not plain entry scoping — see [results-scoping.md](results-scoping.md)'s `## Sharing a ViewModel` |
| `saveState`/`restoreState` for tabs | Per-tab back stacks — see [tabs-flows.md](tabs-flows.md) |
| `savedStateHandle` for results | A result channel — see [results-scoping.md](results-scoping.md) |

## Migration Steps

1. **Make route types implement `NavKey` and register them polymorphically.**
   Nothing breaks yet — Nav 2 has no idea the interface exists, so this step
   is purely additive and safe to land on its own.
2. **Convert leaf screens first.** They have the fewest navigation
   dependencies pointing at them, so a mistake in one screen's conversion
   stays contained to that screen.
3. **Replace `NavHost` with `NavDisplay` for the converted subtree**, and
   install both entry decorators from [nav3-core.md](nav3-core.md)'s
   `## Entry Decorators` — skipping them is silent, not a build failure.
4. **Convert effect handlers screen by screen.** Each
   `navController.navigate(...)` becomes a `backStack.add(...)`; each
   `navigateUp()`/`popBackStack()` becomes `backStack.removeLastOrNull()`.
5. **Move graph-scoped ViewModels last.** They need the most restructuring,
   since Nav 3 has no graph to scope a ViewModel to — entry scoping or the
   shared-owner decorator from [results-scoping.md](results-scoping.md)
   replaces it, and getting that wrong duplicates or drops state.
6. **Replace deep links only after the stack shape is settled.** A deep link
   builds a whole synthetic stack (see
   [deeplinks-platform.md](deeplinks-platform.md)); building it against a
   graph shape that is still moving means rewriting the parser twice.

## Coexistence

The two libraries can run in the same app for the length of a migration: keep
the existing graph on Nav 2, build new feature subtrees on Nav 3, and bridge
at the host boundary — a Nav 2 destination that hosts nothing but a Nav 3
`NavDisplay` with its own back stack, or the reverse. State the constraint
honestly: a single screen cannot be half-migrated, since a screen is either
rendered from a `NavHost`'s `composable<T>` block or from Nav 3's
`entryProvider`, never both. The boundary has to fall between screens, not
inside one, which is why leaf screens convert first in the steps above — they
are the cheapest place to draw that line.

Official guides: https://developer.android.com/guide/navigation/migrate-to-nav3
and https://github.com/android/nav3-recipes/blob/main/docs/migration-guide.md.

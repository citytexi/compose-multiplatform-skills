# Tabs and Conditional Flows

A tab is not a destination, it is a back stack. Deciding that up front
determines whether switching away from a tab and back returns the user where
they were, or drops them at the tab's root every time. See
[nav3-core.md](nav3-core.md) for the single-stack basics this section builds
on.

## Two Tab Models

| | Root swap | Per-tab back stacks |
|---|---|---|
| Structure | One back stack; switching a tab replaces the root | One `NavBackStack` per top-level route |
| Tab history | Lost on switch | Preserved |
| Back from a tab root | Exits the app | Returns to the start tab, then exits |
| Complexity | A few lines | A state holder plus a navigator |
| Use it when | Tabs are shallow — each is a single screen | Any tab can be navigated into |

Pick root swap only when a tab can never grow a second screen — the moment one
tab needs its own drill-down, root swap silently deletes that history on every
switch, and the fix is the per-tab structure below.

## Root Swap

The minimal version, with its limitation stated in the same breath:

```kotlin
fun switchTab(backStack: NavBackStack<NavKey>, target: NavKey) {
    while (backStack.size > 1) backStack.removeLast()
    backStack[0] = target
}
```

One stack, one entry per tab. Fine when every tab is a single flat screen;
wrong the moment a tab needs `Details` pushed on top of it, because the next
`switchTab` call throws that push away.

## Per-Tab Back Stacks

The full pattern needs a state holder that owns one `NavBackStack` per
top-level route, plus a navigator that knows which stack is currently active.
Start with the state holder:

```kotlin
@Composable
fun rememberNavigationState(
    startRoute: NavKey,
    topLevelRoutes: Set<NavKey>,
): NavigationState {
    val topLevelRoute = rememberSerializable(
        startRoute, topLevelRoutes,
        configuration = navConfig,
        serializer = MutableStateSerializer(PolymorphicSerializer(NavKey::class)),
    ) { mutableStateOf(startRoute) }

    val backStacks = topLevelRoutes.associateWith { key -> rememberNavBackStack(navConfig, key) }

    return remember(startRoute, topLevelRoutes) {
        NavigationState(startRoute, topLevelRoute, backStacks)
    }
}

class NavigationState(
    val startRoute: NavKey,
    topLevelRoute: MutableState<NavKey>,
    val backStacks: Map<NavKey, NavBackStack<NavKey>>,
) {
    var topLevelRoute: NavKey by topLevelRoute

    val stacksInUse: List<NavKey>
        get() = if (topLevelRoute == startRoute) listOf(startRoute)
                else listOf(startRoute, topLevelRoute)
}
```

`topLevelRoute` is wrapped with `rememberSerializable` and the same `navConfig`
from [nav3-core.md](nav3-core.md) — it is just as vulnerable to process death
as a back stack, and a plain `mutableStateOf` would forget which tab was
selected once the process is killed and restored.

`stacksInUse` is the load-bearing property: the start tab always stays
underneath, so Back from another tab's root lands on the start tab rather than
exiting the app — the behavior a bottom bar is expected to have, made an
explicit list computation instead of scattered `popUpTo` flags.

Then flatten the active stacks into entries `NavDisplay` can render:

```kotlin
@Composable
fun NavigationState.toEntries(
    entryProvider: (NavKey) -> NavEntry<NavKey>,
): SnapshotStateList<NavEntry<NavKey>> {
    val decoratedEntries = backStacks.mapValues { (_, stack) ->
        rememberDecoratedNavEntries(
            backStack = stack,
            entryDecorators = listOf(
                rememberSaveableStateHolderNavEntryDecorator(),
                rememberViewModelStoreNavEntryDecorator(),
            ),
            entryProvider = entryProvider,
        )
    }
    return stacksInUse
        .flatMap { decoratedEntries[it] ?: emptyList() }
        .toMutableStateList()
}
```

Each stack is decorated separately, so a tab's ViewModels and saveable state
live and die with that tab's stack rather than with one global stack. Only
`stacksInUse` is flattened into the result — an inactive, non-start tab
contributes nothing, which keeps its screens out of the composition entirely.

Then the navigator — the single place that mutates `NavigationState`:

```kotlin
class Navigator(val state: NavigationState) {
    fun navigate(route: NavKey) {
        if (route in state.backStacks.keys) {
            state.topLevelRoute = route          // top-level: switch tabs, keep that tab's stack
        } else {
            state.backStacks[state.topLevelRoute]?.add(route)
        }
    }

    fun goBack() {
        val currentStack = state.backStacks[state.topLevelRoute]
            ?: error("No back stack for ${state.topLevelRoute}")
        if (currentStack.last() == state.topLevelRoute) {
            state.topLevelRoute = state.startRoute   // at a tab root: fall back to the start tab
        } else {
            currentStack.removeLastOrNull()
        }
    }
}
```

In the single-stack world of [nav3-core.md](nav3-core.md), the `entry<...>`
lambda was the only place touching the back stack. With multiple stacks that
lambda no longer knows which one is active, so `Navigator` takes over that
role: `navigate` and `goBack` are the only functions here that touch
`state.backStacks` or `state.topLevelRoute` directly. Everything else — the
entry provider, the bottom bar, every `*Route` composable — calls through
`Navigator`, never the maps or stacks underneath it.

## Wiring the Navigation Bar

`NavDisplay` takes an `entries` parameter for this case, not `backStack` —
there is no longer a single stack to hand it, only the flattened list
`toEntries` produces:

```kotlin
@Composable
fun AppNavDisplay(initialUri: String? = null) {
    val navigationState = rememberNavigationState(
        startRoute = Home,
        topLevelRoutes = setOf(Home, Search, Profile),
    )
    val navigator = remember { Navigator(navigationState) }

    val entryProvider = entryProvider {
        entry<Home> {
            HomeRoute(
                onOpenDetails = { id -> navigator.navigate(Details(id)) },
                onBack = { navigator.goBack() },
            )
        }
        entry<Search> { SearchRoute(onOpenDetails = { id -> navigator.navigate(Details(id)) }) }
        entry<Profile> { ProfileRoute() }
        entry<Details> { key -> DetailsRoute(id = key.id, onBack = { navigator.goBack() }) }
    }

    Scaffold(
        bottomBar = {
            NavigationBar {
                setOf(Home, Search, Profile).forEach { route ->
                    NavigationBarItem(
                        selected = route == navigationState.topLevelRoute,
                        onClick = dropUnlessResumed { navigator.navigate(route) },
                        icon = { Icon(iconFor(route), contentDescription = labelFor(route)) },
                        label = { Text(labelFor(route)) },
                    )
                }
            }
        },
    ) { padding ->
        NavDisplay(
            entries = navigationState.toEntries(entryProvider),
            onBack = { navigator.goBack() },
            modifier = Modifier.padding(padding),
        )
    }
}
```

`HomeRoute` still takes both `onOpenDetails` and `onBack` — the signature does
not change just because navigation is routed through `Navigator` instead of a
raw `backStack`. `iconFor` and `labelFor` stand in for whatever mapping the
app keeps from a top-level route to its bar presentation; keep that mapping
out of the routes themselves, since a route is an address, not a view model.
A larger app builds `navigationState` and `navigator` once and keeps
`Navigator` composition-owned rather than `remember`ing it inline in every
call site — reached from feature modules through `LocalNavigator`, never a
Koin registration; see the di-modularization reference for why.

## Conditional Flows

The rule: a conditional flow is a different back stack, not a screen that
redirects. Checking a condition inside the destination composable means the
user can see that screen for one frame before the redirect fires, and the
back button can return to it afterward, because it was briefly a real entry
on the stack.

The clean version puts the condition above `NavDisplay`, so the restricted
screen's `NavKey` is never even constructed while the condition fails:

```kotlin
@Composable
fun AppRoot(isSignedIn: Boolean) {
    if (!isSignedIn) {
        AuthNavDisplay()      // its own back stack, its own entry provider
    } else {
        AppNavDisplay()
    }
}
```

That works for a condition known before the first composition. For a
condition that flips mid-flow — a session expires, a step suddenly requires
verification — swap the stack contents from the effect handler that observes
the change, and drop the screen the user must not return to:

```kotlin
CollectEffect(viewModel.effect) { effect ->
    when (effect) {
        AuthEffect.SignedIn -> {
            backStack.clear()
            backStack.add(Home)      // Login is gone; Back cannot return to it
        }
    }
}
```

A third shape guards the condition at the point of `navigate()` itself, so a
single restricted screen does not need a whole second `NavDisplay`. That is
the worked example in `android/nav3-recipes`'s
`app/src/main/java/com/example/nav3recipes/conditional/ConditionalActivity.kt`:
each key carries a `requiresLogin: Boolean`, its `Navigator.navigate()` checks
that flag before pushing and pushes `Login(redirectToKey = key)` instead when
it fails, so `Profile` is never pushed and never flashes on screen. On success
`Login` removes itself from the stack and pushes the stored target in its
place, so Back from the post-login screen skips over `Login` entirely. Adapt
the shape, not the file verbatim — it is Android-only (`ComponentActivity`,
`setContent`); only the `requiresLogin` guard inside `navigate()` is portable.

All three shapes answer the same question — where does the check happen — and
none let the restricted screen exist on the stack while the condition is
false. Handing a value back out of a completed flow (the ID a picker
returned, whether login succeeded) is covered in
[results-scoping.md](results-scoping.md).

## Deep Links Into a Tab

A deep link parsed by `parseDeepLink` (see
[deeplinks-platform.md](deeplinks-platform.md)) has to land inside the tab it
belongs to, not a lone top-level `backStack` — that stack no longer exists
once `NavigationState` replaces it. `parseDeepLink`'s returned list is
already rooted at the owning tab's key, so resolving it is `topLevelRoute`
plus a stack replacement, both on `NavigationState`. `AppNavDisplay` takes the
same `initialUri: String?` parameter as the single-stack version in
[deeplinks-platform.md](deeplinks-platform.md), so the platform entry points
there need no change to reach the tabbed app:

```kotlin
fun NavigationState.applyDeepLink(route: List<NavKey>) {
    val owningTab = route.first()
    topLevelRoute = owningTab
    backStacks.getValue(owningTab).apply { clear(); addAll(route) }
}

// in AppNavDisplay, alongside rememberNavigationState(...)
LaunchedEffect(initialUri) {
    val route = initialUri?.let(::parseDeepLink) ?: return@LaunchedEffect
    navigationState.applyDeepLink(route)
}
```

Setting `topLevelRoute` switches the bottom bar to the right tab; this
file's `## Per-Tab Back Stacks` `stacksInUse` then flattens that tab's
freshly-replaced stack with the start tab beneath it, so Back unwinds the
deep link before falling back to the start tab like any other navigation.

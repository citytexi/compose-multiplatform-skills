# Scenes, Layouts, and Transitions

A `Scene` decides how a set of entries is laid out — one pane, two panes side
by side, a dialog floating over the previous entry. A `SceneStrategy`
examines the current entry list and either claims the entries it can render
or declines by returning `null`. When every strategy declines, the implicit
`SinglePaneSceneStrategy` renders the last entry alone, so you never
register it yourself. For `NavDisplay` setup and the back stack itself, see
[nav3-core.md](nav3-core.md).

## What a Scene Decides

A strategy receives the current entry list and returns a `Scene<T>` or
`null`. The scene decides which entries are visible at once and how they are
arranged on screen; `NavDisplay` tries each registered strategy in order and
renders whatever the first non-null result produces. Entries opt in to a
scene through their `metadata` map — a value the entry's `entry<Key>` call
attaches, not something the screen composable ever sees. That indirection is
what lets the same `DetailsRoute` render as a full page under one strategy
and as the right-hand pane under another, with zero branching inside the
composable itself.

## Built-in Scenes

`DialogSceneStrategy` ships in `androidx.navigation3.scene` and is genuinely
built in — no recipe hand-rolls it. `ConfirmDialog` and `FilterSheet` below
are new keys, so each also needs a `subclass(...)` line in `navConfig` (see
[nav3-core.md](nav3-core.md)) or the entry vanishes silently on restore:

```kotlin
@Serializable data object ConfirmDialog : AppRoute

val dialogStrategy = remember { DialogSceneStrategy<NavKey>() }

NavDisplay(
    backStack = backStack,
    onBack = { backStack.removeLastOrNull() },
    sceneStrategies = listOf(dialogStrategy),
    entryProvider = entryProvider {
        entry<ConfirmDialog>(metadata = DialogSceneStrategy.dialog()) {
            AlertDialog(
                onDismissRequest = { backStack.removeLastOrNull() },
                confirmButton = { TextButton(onClick = { backStack.removeLastOrNull() }) { Text("OK") } },
                text = { Text("Delete this item?") },
            )
        }
    },
)
```

`BottomSheetSceneStrategy` is different: Navigation 3 1.1.1 does not ship one.
Both recipe repos hand-roll it as an app-owned class — an `OverlayScene`
wrapping `ModalBottomSheet`, paired with a `SceneStrategy` that checks entry
metadata for a `ModalBottomSheetProperties` value — copied from
`android/nav3-recipes`'s
`app/src/main/java/com/example/nav3recipes/bottomsheet/BottomSheetSceneStrategy.kt`
into your own source tree, not pulled from a library artifact:

```kotlin
@Serializable data object FilterSheet : AppRoute

val bottomSheetStrategy = remember { BottomSheetSceneStrategy<NavKey>() } // your class, not the library's
entry<FilterSheet>(metadata = BottomSheetSceneStrategy.bottomSheet()) {
    FilterContent(onApply = { backStack.removeLastOrNull() })
}
```

Register it in the `sceneStrategies` list, ahead of any strategy that would
otherwise claim the same entry — first match wins, see `## Chaining Strategies`.

## Material Adaptive Scenes

`rememberListDetailSceneStrategy` and its metadata builders come from
`org.jetbrains.compose.material3.adaptive:adaptive-navigation3`, declared
alongside `nav3-ui` in nav3-core.md's `## Dependencies` — package names are
the androidx ones, only the Gradle coordinate is JetBrains's, same
substitution as the Nav 3 UI artifact:

```kotlin
val listDetailStrategy = rememberListDetailSceneStrategy<NavKey>()

NavDisplay(
    backStack = backStack,
    onBack = { backStack.removeLastOrNull() },
    sceneStrategies = listOf(listDetailStrategy),
    entryProvider = entryProvider {
        entry<Home>(
            metadata = ListDetailSceneStrategy.listPane(
                detailPlaceholder = { Text("Select an item") }
            )
        ) {
            HomeRoute(onOpenDetails = { id -> backStack.add(Details(id)) }, onBack = { backStack.removeLastOrNull() })
        }
        entry<Details>(metadata = ListDetailSceneStrategy.detailPane()) { key ->
            DetailsRoute(id = key.id, onBack = { backStack.removeLastOrNull() })
        }
    },
)
```

The payoff: side by side on a wide window, single pane on a narrow one, with
no branching in `HomeRoute` or `DetailsRoute` — the strategy alone decides
the split. `SupportingPaneSceneStrategy` is the second Material scene, for a
persistent pane — a filter rail or a comment list — docked beside a main
pane rather than replacing it; its builders are `mainPane()`/`supportingPane()`.

## Custom Scenes

An implementer writes two pieces: a `Scene<T>` holding the entries it
renders and the composable that lays them out — the same `key` /
`previousEntries` / `entries` / `content` shape described above — plus a
`SceneStrategy<T>` that examines the entry list and either returns a scene
or declines. `calculateScene` is a `@Composable` extension function on
`SceneStrategyScope<T>`, not a plain method — the scope is what gives it
access to `onBack` for scenes (like an overlay) that need to dismiss
themselves:

```kotlin
class TwoPaneSceneStrategy<T : Any>(private val windowSizeClass: WindowSizeClass) : SceneStrategy<T> {
    override fun SceneStrategyScope<T>.calculateScene(entries: List<NavEntry<T>>): Scene<T>? {
        if (!windowSizeClass.isWidthAtLeastBreakpoint(WIDTH_DP_MEDIUM_LOWER_BOUND)) return null
        val lastTwo = entries.takeLast(2)
        // TwoPaneKey nests inside TwoPaneScene, so callers outside it must qualify the reference.
        if (lastTwo.size != 2 || lastTwo.any { !it.metadata.contains(TwoPaneScene.TwoPaneKey) }) return null
        return TwoPaneScene(  // a Scene<T> with two NavEntry<T> panes, laid out 50/50 in a Row
            key = lastTwo[0].contentKey to lastTwo[1].contentKey,
            previousEntries = entries.dropLast(1),
            firstEntry = lastTwo[0],
            secondEntry = lastTwo[1],
        )
    }
}
```

`android/nav3-recipes`'s
`app/src/main/java/com/example/nav3recipes/scenes/listdetail/ListDetailScene.kt`
and `.../scenes/twopane/TwoPaneScene.kt` are worked examples of this pair —
adapt the shape, verify each API, since both target Android-only surfaces in
places. A custom scene claims entries by reading metadata keys it defines
itself (a `NavMetadataKey<Boolean>` object, set through the entry's
`metadata = ` parameter), so a screen that knows nothing about the scene
still renders correctly under the single-pane fallback.

## Chaining Strategies

```kotlin
NavDisplay(
    backStack = backStack,
    onBack = { backStack.removeLastOrNull() },
    sceneStrategies = listOf(dialogStrategy, bottomSheetStrategy, listDetailStrategy),
    entryProvider = entryProvider { /* … */ },
)
```

`NavDisplay` tries each strategy in list order and renders the first
non-null `Scene`; single pane is always the implicit final fallback, so you
never add it. Order the list most specific to least — dialog or bottom sheet
first, adaptive layout last — so a narrow strategy gets first refusal before
a broader one claims the entry. Older Navigation 3 releases chained
strategies with an infix `then` into one `sceneStrategy` parameter; 1.1.0
replaced that with the `sceneStrategies` list above, and no current recipe
uses `then`.

## Scene Decorators

A top app bar or a navigation rail belongs around the scene, not inside
every screen, and it needs to react to window size the same way a scene
does. A `SceneDecoratorStrategy<T>` wraps whatever scene the strategy chain
produced, the same way an entry decorator wraps an entry:

```kotlin
class ResponsiveNavigationSceneDecoratorStrategy<T : Any>(
    private val windowSizeClass: WindowSizeClass,
    private val navBarContent: @Composable () -> Unit,
    private val navRailContent: @Composable () -> Unit,
) : SceneDecoratorStrategy<T> {
    override fun SceneDecoratorStrategyScope<T>.decorateScene(scene: Scene<T>): Scene<T> =
        ResponsiveNavigationScene(scene, windowSizeClass, navBarContent, navRailContent)
}

NavDisplay(
    backStack = backStack,
    onBack = { backStack.removeLastOrNull() },
    sceneStrategies = listOf(listDetailStrategy),
    sceneDecoratorStrategies = listOf(responsiveDecoratorStrategy),
    entryProvider = entryProvider { /* … */ },
)
```

`decorateScene` is a separate extension on `SceneDecoratorStrategyScope<T>`,
registered through its own `sceneDecoratorStrategies` list parameter on
`NavDisplay`, parallel to and independent from `sceneStrategies`. Adapt
`android/nav3-recipes`'s
`app/src/main/java/com/example/nav3recipes/navscenedecorator/ResponsiveNavigationSceneDecorator.kt`:
it renders a rail beside the scene on wide windows and a bar below it on
narrow ones, wrapping whatever scene is active without that scene knowing.

## Transitions

Global transitions on `NavDisplay`:

```kotlin
NavDisplay(
    backStack = backStack,
    onBack = { backStack.removeLastOrNull() },
    transitionSpec = {
        slideInHorizontally(initialOffsetX = { it }) togetherWith
            slideOutHorizontally(targetOffsetX = { -it })
    },
    popTransitionSpec = {
        slideInHorizontally(initialOffsetX = { -it }) togetherWith
            slideOutHorizontally(targetOffsetX = { it })
    },
    predictivePopTransitionSpec = {
        slideInHorizontally(initialOffsetX = { -it }) togetherWith
            slideOutHorizontally(targetOffsetX = { it })
    },
    entryProvider = entryProvider { /* … */ },
)
```

`transitionSpec` covers forward navigation, `popTransitionSpec` covers
programmatic back, and `predictivePopTransitionSpec` covers the
gesture-driven back preview. Omitting the third means the predictive back
preview falls back to the pop spec and can look wrong mid-gesture — the
frames the user drags through were never designed for that.

Per-entry override, composed as metadata rather than passed as an argument:

```kotlin
entry<Settings>(
    metadata = NavDisplay.transitionSpec {
        slideInVertically(initialOffsetY = { it }) togetherWith ExitTransition.KeepUntilTransitionsFinished
    } + NavDisplay.popTransitionSpec {
        EnterTransition.None togetherWith slideOutVertically(targetOffsetY = { it })
    }
) { SettingsRoute(onBack = { backStack.removeLastOrNull() }) }
```

Conditional transitions pick the direction from the entry pair instead of
hardcoding one, for lateral movement between sibling screens where a single
fixed direction reads as wrong for at least one of the pairs:

```kotlin
transitionSpec = {
    val fromKey = initialState.entries.lastOrNull()?.contentKey
    val toKey = targetState.entries.lastOrNull()?.contentKey
    when (fromKey to toKey) {
        Step1Key to Step2Key -> slideLeft()
        Step2Key to Step3Key -> slideUp()
        Step3Key to Step4Key -> slideRight()
        else -> slideRight()
    }
}
```

adapted from `android/nav3-recipes`'s
`app/src/main/java/com/example/nav3recipes/conditionaltransitions/ConditionalTransitionsActivity.kt` —
strip its `ComponentActivity` and DI-framework wiring, which are Android-only;
keep only the `initialState`/`targetState` comparison, which is portable.

Keep animation out of state: a transition is a property of the display, not
of screen state. Do not add an animation-direction flag to your MVI state —
`transitionSpec` already has everything it needs in the entry pair.

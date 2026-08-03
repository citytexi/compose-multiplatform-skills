---
name: compose-navigation
description: Use when adding or reviewing navigation in a Compose Multiplatform app — defining Nav 3 route keys and back stack, configuring NavDisplay, scenes and adaptive layouts, tabs with multiple back stacks, returning results between screens, deep links per platform, wiring navigation with Koin or Metro, or working in an existing Nav 2 codebase.
---

# Compose Multiplatform Navigation (Nav 3)

Navigation 3 is the default for new Compose Multiplatform code: the back
stack is Compose state that the route layer owns, and `NavDisplay` renders
it. ViewModels emit semantic effects and never touch the stack directly.
Nav 2 is covered only for work inside an existing Nav 2 codebase.

<HARD-GATE>
- CMP: NavDisplay comes from org.jetbrains.androidx.navigation3:navigation3-ui.
  NEVER put androidx.navigation3:navigation3-ui in commonMain — its non-Android targets are stubs.
- NEVER pass the back stack (or a NavController) into a ViewModel or a leaf composable.
- NEVER navigate from a composable body — navigate from an effect handler or an event callback.
- ALWAYS pass onBack, AND install both entry decorators (saveable state holder
  + ViewModel store) — as NavDisplay's entryDecorators on the backStack
  overload, or inside rememberDecoratedNavEntries on the entries overload,
  which has no entryDecorators parameter of its own. Exception:
  [references/results-scoping.md](references/results-scoping.md)'s
  shared-ViewModel decorator replaces the ViewModel-store one, not beside it.
- Persisted back stacks on non-Android targets, desktop JVM included, REQUIRE SavedStateConfiguration with every NavKey subclass registered polymorphically.
- Respect an existing Nav 2 codebase — do NOT force a migration.
</HARD-GATE>

## Checklist

Create a todo for each item and complete them in order:

1. Read the existing navigation code — which library, where the back stack
   lives, how effects reach it.
2. Identify the concern — core setup, scenes, tabs, results, deep links,
   modularization, or Nav 2.
3. Apply the Core Rules below.
4. Load the relevant reference for depth.
5. Check the work against the Red Flags table.
6. Confirm every back stack mutation sits in an effect handler or an event
   callback.

## Core Rules

### Route keys and back stack ownership
Model routes as a `@Serializable sealed interface AppRoute : NavKey`, one
member per destination, with primitive-only payloads. The app owns
`NavBackStack<NavKey>` as a `SnapshotStateList`; in shared `commonMain` code
create it unconditionally with `rememberNavBackStack(navConfig, ...)` — the
no-config overload is Android-only and lives in `androidMain` — and register
each `NavKey` subclass polymorphically in `SavedStateConfiguration`.
→ route keys, dependencies, back stack creation, NavDisplay wiring: [references/nav3-core.md](references/nav3-core.md)

### The MVI boundary
The `entry<Key>` lambda in `entryProvider` is the only place that touches
`backStack` — it defines the callbacks a Route composable receives. The
Route composable collects ViewModel state and translates each `Effect` into
a call to one of those callbacks; leaf composables below it see neither the
back stack nor a navigation callback, only `state` and `onEvent`.
→ the ViewModel/route/back-stack split: [references/nav3-core.md](references/nav3-core.md)

### Scenes decide layout
A `SceneStrategy` examines the current entry list and claims a set or
declines; entries opt in through `metadata`, not branching inside the
composable. Chain strategies most-specific first — dialog and bottom sheet
before an adaptive list-detail split — and set all three transition specs
(`transitionSpec`, `popTransitionSpec`, `predictivePopTransitionSpec`) so
predictive back doesn't fall back to the wrong preview.
→ scenes, adaptive layouts, and transitions: [references/scenes-animations.md](references/scenes-animations.md)

### A tab is a back stack
Root-swap a single stack only when every tab is one flat screen; the moment
a tab can be navigated into, give it its own `NavBackStack` behind a
`NavigationState`/`Navigator` pair. Gate a conditional flow (auth, a
restricted step) above `NavDisplay` or by swapping the stack from an effect
handler — never by redirecting from inside the destination composable.
→ per-tab back stacks and conditional flows: [references/tabs-flows.md](references/tabs-flows.md)

### Results do not travel in route arguments
A `NavKey` carries data forward, never a result backward — pick an event
channel for a one-shot action, a state channel for a rendered value, or
saved state when it must survive process death. ViewModels are entry-scoped
by default; only reach for `SharedViewModelStoreNavEntryDecorator` when two
adjacent entries are genuinely one flow.
→ result channels and ViewModel scoping: [references/results-scoping.md](references/results-scoping.md)

### Deep links are parsed in commonMain, registered per target
Navigation 3 ships no deep-link API — write `parseDeepLink(uri):
List<AppRoute>?` once in `commonMain`, returning the whole synthetic stack
(not a single destination) so Up and Back behave correctly. Each platform
hands the URI to that parser through its own entry point: an Android
manifest intent filter plus `onNewIntent`, iOS `onOpenURL`, a desktop
process argument, or a web history sync.
→ URI parsing and per-platform wiring: [references/deeplinks-platform.md](references/deeplinks-platform.md)

### Features contribute their entries
A feature module exposes an `EntryProviderScope<NavKey>` extension function
that adds its own entries; the app module calls that function by name and
never writes `entry<Details>` itself. Split each feature into `api` (route
keys only) and `impl`, with `impl` depending on other features' `api`
modules, never their `impl`.
→ feature-contributed entries and the api/impl split: [references/di-modularization.md](references/di-modularization.md)

### Nav 2 is for existing codebases
Navigation 2 is not deprecated — do not migrate a working codebase just
because Nav 3 exists. When a migration is warranted, convert leaf screens
first, move graph-scoped ViewModels last, and expect the two libraries to
coexist at a screen boundary for the length of the migration.
→ Nav 2 patterns, the concept mapping, and migration steps: [references/nav2-migration.md](references/nav2-migration.md)

## Red Flags — STOP

| Smell | Fix |
|---|---|
| `androidx.navigation3:navigation3-ui` in `commonMain` | Use `org.jetbrains.androidx.navigation3:navigation3-ui` — the androidx variants for non-Android targets are stubs |
| Back stack or `NavController` held by a ViewModel | ViewModel emits an `Effect`; the route layer navigates |
| `backStack.add(...)` in a composable body | Move it into an effect handler or an event callback |
| `NavDisplay` without `onBack` | System back does nothing — always provide it |
| Missing entry decorators | Install both the saveable-state and ViewModel-store decorators |
| Persisted back stack on iOS/JS/Wasm/desktop without `SavedStateConfiguration` | Register every `NavKey` subclass polymorphically |
| String routes | `@Serializable` key types implementing `NavKey` |
| Tab switch rebuilds that tab's stack | One `NavBackStack` per top-level route |
| Result handed back by navigating to the caller with new arguments | Pick a result channel — event, state, or saved state |
| Deep link pushes a single destination | Build the whole synthetic back stack so Up works |
| App module enumerates every `entry<...>` | Feature-contributed entry builders |
| Nav 2 chosen for new CMP code | Nav 3, unless the codebase is already Nav 2 |

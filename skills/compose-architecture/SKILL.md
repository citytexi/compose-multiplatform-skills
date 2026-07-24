---
name: compose-architecture
description: Use when designing or refactoring Compose Multiplatform (CMP) screen architecture, modeling MVI state, choosing a state owner, or fixing architecture anti-patterns. Covers Event/State/Effect, domain layer, module rules, and overengineering avoidance.
---

# Compose Multiplatform Architecture (MVI)

Screen architecture and state management for Compose Multiplatform using MVI
(Model-View-Intent). MVI is the default pattern for CMP: a single `onEvent()`
entry point, immutable state, and one-shot effects.

<HARD-GATE>
- NEVER put business logic (validation, calculations, data loading, submit
  enablement) inside a composable body.
- Decide the state owner BEFORE writing screen code.
- Respect an existing project's MVI conventions and base classes — do NOT force
  a migration or introduce a competing base class.
- Keep canonical values in state; derive display values at the UI boundary.
</HARD-GATE>

## Checklist

Create a todo for each item and complete them in order:

1. Read the existing code first — conventions, base classes, file layout. For a
   small ask, read only the relevant files.
2. Identify the concern — state owner, MVI contract, state shape, or an
   anti-pattern.
3. Apply the Core Rules below.
4. Load the relevant reference only when deeper guidance is needed.
5. Check the work against the Red Flags table.
6. Write the minimal correct solution — no overengineering.

## Core Rules

### Source of truth and state owner
One screen state holder owns `StateFlow<State>`. Visual-only concerns stay in
local Compose state; persisted data lives in the repository. Do not mix them.
→ state owner selection, where logic belongs, state slicing: [references/architecture.md](references/architecture.md)

### Layering and modules
The domain layer is pure business logic with zero platform imports (runs in
`commonTest`); repository interfaces live in domain, implementations in data;
use cases only for multi-step orchestration, never single repo pass-throughs.
Organize by feature; `feature:impl` never depends on another `feature:impl`,
`core:*` never depends on features. Cross-feature contact goes through an `:api`
module or a shared `core` repository — never another feature's ViewModel.
→ domain layer, module dependency rules, inter-feature communication: [references/architecture.md](references/architecture.md)

### MVI 3-type contract
`Event` is the only input, processed by a single `onEvent()`. `State` is an
immutable `data class` fully describing the render. `Effect` is a one-off command
(navigate, snackbar) delivered via `Channel`, never stored as consumable state.
→ full pattern, naming, and examples: [references/mvi.md](references/mvi.md)
→ using the Orbit MVI library (container/intent/reduce): [references/orbit-mvi.md](references/orbit-mvi.md)

### State shape
Immutable `data class` + immutable collections to preserve Compose skipping.
Store canonical values; use computed properties for trivial derivations; never
duplicate derived data.
→ forms, calculators, four-bucket model: [references/state-modeling.md](references/state-modeling.md)

### Avoid overengineering
One feature ViewModel, one state, one `onEvent()`. No 4-type MVI, no use-case
wrappers around single repository calls, no base class until 10+ features share
real boilerplate.
→ decision rules, naming, import hygiene: [references/clean-code.md](references/clean-code.md)

## Red Flags — STOP

| Smell | Fix |
|---|---|
| Business logic in a composable body | Move to ViewModel/domain; composable only renders |
| Giant god-ViewModel | One ViewModel per screen or independent flow |
| One-off event stored as `showX: Boolean` in state | Deliver via `Effect` over a `Channel` |
| Duplicated derived data (`total` + `formattedTotal` stored) | Keep canonical value; derive via computed property |
| Mutable collections / lambdas in state | Immutable `data class` + immutable collections |
| ViewModel doing platform work (share, nav, analytics) | Emit an `Effect`; handle in the Route composable |
| Pre-baked display strings in state | Keep canonical values until the presentation boundary |
| 4-type MVI / base class on a trivial screen | 3-type MVI with inline `updateState { copy(...) }` |
| Forcing MVI onto an existing codebase | Respect existing patterns; new features only |

→ full table and BAD/GOOD examples: [references/anti-patterns.md](references/anti-patterns.md)

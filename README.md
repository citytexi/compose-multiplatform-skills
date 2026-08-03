# compose-multiplatform-skills

Agent skills for building Compose Multiplatform (CMP) apps. Each skill is a
focused, independently-triggering module under `skills/`, following the
multi-skill structure of the Superpowers library. Knowledge base adapted from
the Compose Multiplatform community reference set; MVI is the default
architecture.

## Skills

| Skill | Purpose |
|---|---|
| [compose-architecture](skills/compose-architecture/SKILL.md) | MVI screen architecture and state management |
| [compose-di](skills/compose-di/SKILL.md) | Dependency injection: Koin, Koin Annotations, Metro |
| [compose-async](skills/compose-async/SKILL.md) | Kotlin Coroutines & Flow: hot streams, operators, dispatchers, stateIn, testing |
| [compose-networking](skills/compose-networking/SKILL.md) | Ktor client: setup, DTOs, error handling, auth, WebSockets/SSE, MockEngine |
| [compose-navigation](skills/compose-navigation/SKILL.md) | Navigation 3: route keys, back stack, scenes, tabs, results, deep links, modularization |

## Roadmap

Additional domains will be added as separate skills: persistence, UI, animation, cross-platform,
performance, testing, and build.

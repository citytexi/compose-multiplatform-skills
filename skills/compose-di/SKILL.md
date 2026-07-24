---
name: compose-di
description: Use when setting up or refactoring dependency injection in a Compose Multiplatform app — choosing between Koin, Koin Annotations, and Metro; writing modules or graphs; injecting ViewModels; or scoping dependencies. Koin is the default.
---

# Compose Multiplatform Dependency Injection

Dependency injection for Compose Multiplatform. **Koin is the default** — mature,
low-ceremony, first-class on every CMP target. Add **Koin Annotations** for
compile-time safety, or choose **Metro** when you want full build-time graph
validation without KSP.

<HARD-GATE>
- Inject through constructors. NEVER field injection (`lateinit var` + framework annotation).
- Depend on interfaces, not concrete implementations, at module boundaries.
- Call `startKoin` exactly once, at the app entry point.
- NEVER leak DI-framework types (Koin `Module`, Metro graph) into the domain layer.
- Android `Context` must never appear in a `commonMain` Koin module — use `expect/actual` platform modules.
</HARD-GATE>

## Checklist

Create a todo for each item and complete them in order:

1. Read the existing DI setup — which framework, where `startKoin`/the graph lives, module layout.
2. Pick the framework per the decision table (default: Koin). Respect an existing choice.
3. Apply the Core Rules — constructor injection, interface binding, correct scope.
4. Load the relevant reference for framework specifics.
5. Check the work against the Red Flags table.
6. Add graph verification (Koin `verify()` / `KOIN_CONFIG_CHECK` / Metro build-time check).

## Framework Decision

| | Koin (default) | Koin Annotations | Metro |
|---|---|---|---|
| Resolution | Runtime DSL | KSP codegen on Koin | Compile-time graph |
| Error detection | Runtime (`verify()` in tests) | Build-time (`KOIN_CONFIG_CHECK`) | Build-time (compiler plugin) |
| Setup | Lowest (DSL modules) | + KSP config | Gradle compiler plugin |
| CMP targets | All (Web experimental) | All | JVM/Android/iOS/JS/Wasm/Native |
| Reach for it when | Default; broad ecosystem, least ceremony | You want Koin + compile-time checks | Build-time guarantees; Dagger/Anvil migration |

## Core Rules

### Constructor injection + interface binding
Inject via constructor; bind interfaces to implementations at module/graph
boundaries so tests can swap fakes without mocking the framework.
→ principles, scope lifecycle, module organization: [references/di-concepts.md](references/di-concepts.md)

### Koin runtime DSL (default)
`module { single<Repo> { RepoImpl(get()) }; viewModelOf(::ScreenViewModel) }`,
`startKoin { modules(...) }` once, `koinViewModel()` / `koinInject()` in Compose.
→ packages, setup, modules, Compose injection, scopes, testing: [references/koin.md](references/koin.md)

### Compile-time safety on Koin
Adopt Koin Annotations (`@Single`, `@KoinViewModel`, `@Module` + `@ComponentScan`)
with KSP and `KOIN_CONFIG_CHECK` to verify the graph at build time.
→ KSP setup and annotations: [references/koin-annotations.md](references/koin-annotations.md)

### Metro compile-time graph
`@DependencyGraph(AppScope::class)`, `@Inject` constructors,
`@ContributesBinding`, `@SingleIn` — validated by the compiler plugin.
→ graph, bindings, contributions, scoping: [references/metro.md](references/metro.md)

## Red Flags — STOP

| Smell | Fix |
|---|---|
| Field injection (`lateinit var` + annotation) | Constructor injection |
| Depending on a concrete class across a boundary | Bind and depend on an interface |
| God-module: everything a `single` | Feature-first modules; `factory`/`scoped` per real lifetime |
| `startKoin` called more than once | Call once at app entry; `loadKoinModules` for dynamic additions |
| `factory { MyViewModel() }` for a ViewModel | `viewModelOf(::MyViewModel)` — lifecycle-aware |
| Android `Context` in a `commonMain` module | `expect/actual` platform modules |
| No graph verification | Koin `verify()` / `KOIN_CONFIG_CHECK` / Metro build check |
| DI-framework types in the domain layer | Keep DI at the composition root; domain stays framework-free |

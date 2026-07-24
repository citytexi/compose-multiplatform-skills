# Dependency Injection with Metro

> API verified against Metro 1.3.2 docs; re-verify `@DependencyGraph.Factory`
> and assisted-injection signatures against the current Metro documentation
> before use.

Metro is a compile-time dependency-injection framework for Kotlin
Multiplatform — a Kotlin compiler plugin that validates the dependency graph at
build time (no KAPT/KSP). It draws on Dagger, Anvil, and kotlin-inject. Reach
for Metro when you want full build-time graph validation or are migrating from
Dagger/Anvil/kotlin-inject. For the default runtime approach see
[koin.md](koin.md); for framework-agnostic principles see
[di-concepts.md](di-concepts.md).

## Setup

```kotlin
plugins {
    kotlin("multiplatform")
    id("dev.zacsweers.metro") version "1.3.2"   // verify latest
}
```

Applying the plugin adds the runtime; it targets JVM/Android/iOS/JS/Wasm/Native.

## Dependency Graph

A graph is an interface annotated with `@DependencyGraph`. It declares a scope
and exposes accessors for the roots you want to pull out:

```kotlin
import dev.zacsweers.metro.AppScope
import dev.zacsweers.metro.DependencyGraph
import dev.zacsweers.metro.createGraph

@DependencyGraph(AppScope::class)
interface AppGraph {
    val repository: Repository
}

val appGraph = createGraph<AppGraph>()
val repository = appGraph.repository
```

When the graph needs runtime seeds (config, platform handles), use a graph
factory:

```kotlin
import dev.zacsweers.metro.Provides
import dev.zacsweers.metro.createGraphFactory

@DependencyGraph(AppScope::class)
interface AppGraph {
    val repository: Repository
    @DependencyGraph.Factory
    interface Factory {
        fun create(@Provides config: AppConfig): AppGraph
    }
}

val appGraph = createGraphFactory<AppGraph.Factory>().create(config)
```

## Constructor Injection

Annotate the class with `@Inject`; Metro wires the constructor params from the
graph:

```kotlin
import dev.zacsweers.metro.Inject

@Inject
class Repository(private val apiClient: ApiClient)
```

## Providing Bindings

Use `@Binds` to map an interface to its implementation inside a graph:

```kotlin
import dev.zacsweers.metro.Binds

@Binds val RepositoryImpl.bind: Repository
```

Use `@Provides` inside a `@BindingContainer` for values you construct manually:

```kotlin
import dev.zacsweers.metro.BindingContainer
import dev.zacsweers.metro.ContributesTo
import dev.zacsweers.metro.Provides

@ContributesTo(AppScope::class)
@BindingContainer
object NetworkBindings {
    @Provides fun provideHttpClient(): HttpClient = HttpClient()
}
```

## Contributions

`@ContributesTo(scope)` aggregates a binding container into any graph of that
scope across modules; `@ContributesBinding(scope)` contributes an implementation
as its supertype:

```kotlin
import dev.zacsweers.metro.ContributesBinding
import dev.zacsweers.metro.Inject

@ContributesBinding(AppScope::class)
@Inject
class RepositoryImpl(private val apiClient: ApiClient) : Repository
```

This is the multi-module story — feature modules contribute; the app graph
aggregates them without explicit wiring.

## Scoping

`@SingleIn(AppScope::class)` makes a binding a singleton within that scope:

```kotlin
import dev.zacsweers.metro.SingleIn

@SingleIn(AppScope::class)
@ContributesBinding(AppScope::class)
@Inject
class DatabaseImpl : Database
```

Custom scopes are declared as scope marker classes/annotations (verify the exact
declaration syntax against the Metro docs for your version).

## Testing

Replace real bindings with fakes via a dynamic graph:

```kotlin
import dev.zacsweers.metro.createDynamicGraph

@BindingContainer
object FakeBindings {
    @Provides fun provideRepository(): Repository = FakeRepository()
}

val testGraph = createDynamicGraph<AppGraph>(FakeBindings)
```

Because the graph is validated at compile time, most wiring errors surface at
build rather than in tests.

## Metro vs Koin

Metro validates the whole graph at compile time with no runtime container and no
KSP. Koin (the default, [koin.md](koin.md)) is a runtime DSL with optional KSP
compile-checks ([koin-annotations.md](koin-annotations.md)). Choose Metro for
build-time guarantees or a Dagger/Anvil-style migration; choose Koin for lower
ceremony and the broadest CMP ecosystem.

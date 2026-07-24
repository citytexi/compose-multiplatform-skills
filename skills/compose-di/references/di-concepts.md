# Dependency Injection Concepts

Framework-agnostic DI principles for Compose Multiplatform. For framework
specifics, see [koin.md](koin.md) (default), [koin-annotations.md](koin-annotations.md),
or [metro.md](metro.md). **Koin is the default DI for CMP; Metro is the
compile-time alternative.**

These principles apply regardless of framework choice.

## Constructor Injection

Always inject dependencies through the constructor. Field injection
(`@Inject lateinit var`) couples the class to the DI framework, hides
dependencies from the type signature, and makes testing harder because you can
no longer construct the object directly with fakes. Constructor injection keeps
classes framework-agnostic and trivially testable.

## Interface-Based Design

Bind interfaces to implementations — repositories, data sources, and platform
services should be defined as interfaces. This enables swapping implementations
in tests without mocking the DI framework.

```kotlin
interface UserRepository {
    suspend fun getUser(id: String): User
}

// Bind implementation via DI:
// Koin DSL:          single<UserRepository> { UserRepositoryImpl(get()) }
// Koin Annotations:  @Single class UserRepositoryImpl(...) : UserRepository
// Metro:             @ContributesBinding(AppScope::class) @Inject class UserRepositoryImpl(...) : UserRepository
```

## Scope Lifecycle

Match each dependency's scope to its actual lifetime:

| Scope | When to use | Examples |
|---|---|---|
| Singleton | Lives for app lifetime | API client, database, analytics |
| Activity-retained | Survives config changes | User session, auth state |
| ViewModel-scoped | Tied to a feature screen | Feature-specific calculators, validators |
| Factory (new each time) | Stateless or short-lived | Formatters, mappers |

Over-scoping wastes memory; under-scoping creates redundant instances.

## Module Organization

Organize DI modules by feature, not by type. Each feature module declares its
own dependencies:

```text
feature-product/
    ProductModule         → repository, calculator, validator, ViewModel
feature-settings/
    SettingsModule        → repository, ViewModel
core/
    CoreModule            → API client, database, platform bindings
```

Combine feature modules in the app module. Platform-specific bindings go in
platform source sets (`androidMain`, `iosMain`).

## Testing Principle

Swap real implementations with fakes via DI configuration — don't mock the DI
framework itself. Each framework offers a build-safe way to verify the graph:

- **Koin**: `verify()` for graph verification, plus module overrides in tests.
- **Koin Annotations**: `KOIN_CONFIG_CHECK` fails the build on missing bindings.
- **Metro**: compile-time graph validation, with `createDynamicGraph` to inject
  fakes into a test graph.

For ViewModel unit testing, see the compose-architecture skill.

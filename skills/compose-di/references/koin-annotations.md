# Koin Annotations (KSP)

Compile-time-safe Koin via KSP code generation. Layer this on top of
[koin.md](koin.md) when you want the graph verified at build time.
Multiplatform-capable.

## Setup (KSP)

```kotlin
plugins { id("com.google.devtools.ksp") }

kotlin {
    sourceSets.commonMain.dependencies {
        implementation("io.insert-koin:koin-annotations:2.3.0")   // verify latest
    }
    sourceSets.named("commonMain").configure {
        kotlin.srcDir("build/generated/ksp/metadata/commonMain/kotlin")
    }
}

dependencies {
    add("kspCommonMainMetadata", "io.insert-koin:koin-ksp-compiler:2.3.0")
    add("kspAndroid", "io.insert-koin:koin-ksp-compiler:2.3.0")
    // ... add for each target (kspIosArm64, kspIosSimulatorArm64, etc.)
}

ksp {
    arg("KOIN_USE_COMPOSE_VIEWMODEL", "true")   // multiplatform ViewModel DSL
    arg("KOIN_CONFIG_CHECK", "true")            // compile-time verification
}
```

## Annotations

| Annotation | Equivalent DSL | Purpose |
|---|---|---|
| `@Single` | `single { }` | Singleton |
| `@Factory` | `factory { }` | New instance each time |
| `@KoinViewModel` | `viewModelOf(::Class)` | ViewModel declaration |
| `@InjectedParam` | `parametersOf(...)` | Runtime parameter |
| `@Module` + `@ComponentScan` | `module { }` | Auto-discover annotated classes in package |
| `@Named` | qualifier | Distinguish multiple bindings of the same type |
| `@Provided` | `get()` | Mark a constructor param as resolved from the graph |

```kotlin
@Single
class UserRepositoryImpl(private val api: Api) : UserRepository

@KoinViewModel
class ProductViewModel(private val repository: ProductRepository) : ViewModel()

@Module
@ComponentScan("com.example.feature.product")
class ProductModule
```

## Generated Modules

The KSP compiler generates a `.module` extension on each annotated `@Module`
class. Start Koin with it:

```kotlin
startKoin { modules(ProductModule().module) }
```

Combine with hand-written DSL modules via `includes(...)`:

```kotlin
val appModule = module { includes(ProductModule().module, coreModule) }
```

## Compile-Time Verification

`KOIN_CONFIG_CHECK=true` makes missing bindings fail the build — the
compile-time analogue of runtime `verify()` in [koin.md](koin.md). Every
dependency required by an annotated class must be resolvable at compile time or
the build breaks.

This is how Koin Annotations closes the gap with compile-time frameworks like
[metro.md](metro.md) while keeping Koin's runtime DSL and ecosystem.

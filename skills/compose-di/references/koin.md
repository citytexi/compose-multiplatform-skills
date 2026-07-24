# Dependency Injection with Koin

Runtime DSL multiplatform DI — the default for Compose Multiplatform. For
framework-agnostic principles see [di-concepts.md](di-concepts.md); for
compile-time safety see [koin-annotations.md](koin-annotations.md).

References:
- [Koin for Compose](https://insert-koin.io/docs/reference/koin-compose/compose)

## Package Selection

### CMP projects (recommended)

```kotlin
commonMain.dependencies {
    implementation(platform("io.insert-koin:koin-bom:4.2.2"))   // verify latest on Maven Central
    implementation("io.insert-koin:koin-core")
    implementation("io.insert-koin:koin-compose")
    implementation("io.insert-koin:koin-compose-viewmodel")
}
```

### Android-only projects

```kotlin
dependencies {
    implementation("io.insert-koin:koin-androidx-compose:4.2.2")  // includes compose + viewmodel; verify latest
}
```

| Package | Purpose |
|---|---|
| `koin-core` | Core DI engine (multiplatform) |
| `koin-compose` | Base Compose API (`koinInject`) |
| `koin-compose-viewmodel` | ViewModel injection (`koinViewModel`) |
| `koin-compose-viewmodel-navigation` | Navigation entry-provider integration (navigation DI is covered by a future compose-navigation skill) |
| `koin-androidx-compose` | Android convenience (includes compose + viewmodel) |

Platform support: Android, iOS, Desktop — full. Web — experimental.

## Setting Up Koin

Initialize outside Compose with a shared `initKoin` and a platform-specific
config lambda:

```kotlin
// commonMain
fun initKoin(config: KoinAppDeclaration? = null) {
    startKoin {
        config?.invoke(this)
        modules(appModule, featureModules)
    }
}

// Android — Application class
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        initKoin { androidContext(this@MyApplication); androidLogger() }
    }
}
```

iOS — call from Swift. The `do` prefix is added because `init` is reserved:

```swift
import ComposeApp
@main struct iOSApp: App {
    init() { InitKoinKt.doInitKoin(config: nil) }
    var body: some Scene { WindowGroup { ContentView() } }
}
```

Alternative — Compose-managed:

```kotlin
KoinApplication(configuration = koinConfiguration { modules(appModule) }) {
    MainScreen()
}
```

## Defining Modules

```kotlin
val appModule = module {
    // Classic DSL (manual wiring)
    single<UserRepository> { UserRepositoryImpl() }
    factory { ProductValidator() }
    viewModelOf(::ProductViewModel)

    // Compiler Plugin DSL (auto-wiring — requires Koin Compiler Plugin)
    single<ProductCalculator>()                                   // auto-resolves constructor params
    single<UserRepositoryImpl>() bind UserRepository::class       // bind exposes impl as interface
    viewModel<ProductViewModel>()
}
```

| DSL | Lifecycle | When to use |
|---|---|---|
| `single { }` | App lifetime (singleton) | Stateless services, repositories, API clients, databases |
| `factory { }` | New instance per call | Stateful/short-lived — validators, formatters, use-cases with request state |
| `scoped { }` | Bound to a Koin scope | Shared within a flow (e.g., checkout) but not globally |
| `viewModelOf(::Class)` | ViewModel lifecycle | Survives recomposition + config changes, cleared when owner destroyed |

The Koin Compiler Plugin gives compile-safe checks for the auto-wiring DSL; the
full annotation workflow is in [koin-annotations.md](koin-annotations.md).

### Feature-first module organization

```kotlin
val productModule = module {
    single<ProductRepository> { ProductRepositoryImpl(get()) }
    viewModelOf(::ProductViewModel)
}
val appModule = module { includes(productModule, settingsModule, coreModule) }
```

### Platform-specific implementations

Use `expect/actual` modules when implementations differ per platform:

```kotlin
// commonMain
expect val platformModule: Module

// androidMain
actual val platformModule = module { single<HapticFeedback> { AndroidHapticFeedback(get()) } }

// iosMain
actual val platformModule = module { single<HapticFeedback> { IosHapticFeedback() } }

startKoin { modules(appModule, platformModule) }
```

For platform dependencies (e.g., Android `Context`) in `expect/actual` classes,
use `KoinComponent` with `inject()` — justified because constructors must match
across platforms. Avoid `KoinComponent` elsewhere.

## Injection in Compose

```kotlin
// Any dependency
val service: MyService = koinInject()

// ViewModel — lifecycle-aware
val viewModel = koinViewModel<HomeViewModel>()

// With runtime parameters
val viewModel = koinViewModel<DetailViewModel> { parametersOf(itemId) }

// Keyed — unique instance per entity
val viewModel = koinViewModel<DetailViewModel>(key = "detail_$itemId", parameters = { parametersOf(itemId) })
```

Inject as default parameters for testability:
`fun MyScreen(service: MyService = koinInject())`.

| Function | Platform | When to use |
|---|---|---|
| `koinInject<T>()` | All | Non-ViewModel dependencies inside `@Composable` |
| `koinViewModel<T>()` | All | ViewModel — lifecycle-aware, survives recomposition |
| `koinActivityViewModel<T>()` | Android | Share ViewModel across all composables in an Activity |
| `parametersOf(...)` | All | Pass runtime values to `koinViewModel` or `koinInject` |
| `get<T>()` | All | Resolve inside `module { }` only — never in composables |

Navigation-specific DI (`koinEntryProvider`, `navigation<T>`) is covered by the
future compose-navigation skill.

## Scopes

```kotlin
val appModule = module {
    scope<CheckoutFlow> {
        scoped { CheckoutState() }
        viewModel<CheckoutViewModel>()
    }
}
```

`scope<T>` works on all platforms. On Android, `activityRetainedScope { }`
survives config changes (same idea, platform-specific).

## Koin with MVI

MVI is framework-agnostic (see the compose-architecture skill for the MVI
pattern). The Koin-specific parts are constructor injection and
`koinViewModel()`:

```kotlin
class ProductViewModel(private val repository: ProductRepository) : ViewModel() {
    // StateFlow<State>, Channel<Effect>, onEvent()
}
// Module: viewModelOf(::ProductViewModel)
// Route:  val viewModel = koinViewModel<ProductViewModel>()
```

## Testing

`verify()` performs a dry-run check — catches missing declarations before
runtime:

```kotlin
class KoinModuleCheck : KoinTest {
    @Test
    fun verifyAllModules() {
        appModule.verify(extraTypes = listOf(SavedStateHandle::class))
    }
}
// commonTest.dependencies { implementation("io.insert-koin:koin-test:4.2.2") }
```

## Anti-Patterns

| Anti-pattern | Why it is harmful | Better approach |
|---|---|---|
| `factory { MyViewModel() }` for ViewModels | Not lifecycle-aware, new instance on recomposition | `viewModelOf(::MyViewModel)` |
| Not using `parametersOf` for runtime params | Constructor params unresolved | `koinViewModel { parametersOf(id) }` |
| `koin-compose` without `koin-compose-viewmodel` | `koinViewModel()` unavailable | Add `koin-compose-viewmodel` |
| Calling `startKoin` multiple times | `KoinAppAlreadyStartedException` | Call once, use `loadKoinModules` for dynamic additions |
| Android `Context` in `commonMain` modules | Breaks multiplatform | `expect/actual` platform modules |

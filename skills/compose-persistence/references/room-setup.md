# Room 3 Setup

Room 3 is a rename as much as a release. The group `androidx.room` became
`androidx.room3`, artifacts `room-*` became `room3-*`, the package
`androidx.room.*` became `androidx.room3.*`, and the Gradle plugin id and its
extension block both changed, so nothing copied from an Android-era sample or
a 2.x codebase compiles against these coordinates. Verify every fact below
against <https://developer.android.com/kotlin/multiplatform/room> and the
release notes at <https://developer.android.com/jetpack/androidx/releases/room3>
before shipping — the multiplatform page currently renders the Gradle
extension as `room { schemaDirectory(...) }`, a documentation error; the
plugin registers the extension as `room3`, and the compiler's own error text
says `room3 { schemaDirectory(...) }`. This file writes `room3` throughout.

## Coordinates

| Group | Artifact | Version | Note |
|---|---|---|---|
| `androidx.room3` | `room3-runtime` | `3.0.2` | replaces the Room 2.x `room-runtime` group |
| `androidx.room3` | `room3-compiler` | `3.0.2` | KSP only — no `kapt`, no Java annotation processor |
| `androidx.room3` | `room3-paging` | `3.0.2` | `PagingSourceDaoReturnTypeConverter`, required for a `PagingSource`-returning DAO function |
| `androidx.room3` | `room3-testing` | `3.0.2` | `MigrationTestHelper` |
| `androidx.room3` | `room3-migration` | `3.0.2` | schema parsing for migration tooling |
| `androidx.sqlite` | `sqlite-bundled` | `2.7.0` | `BundledSQLiteDriver`, ships SQLite compiled from source |
| `androidx.sqlite` | `sqlite-web` | `2.7.0` | `WebWorkerSQLiteDriver`, JS and WasmJS only |

Targets this pin actually reaches, confirmed from `room3-runtime-3.0.2.module`:
`androidMain`, `jvmMain` (desktop), `iosMain` (`iosArm64`, `iosSimulatorArm64`),
`wasmJsMain` / `jsMain`, plus `macosArm64`, `linuxX64`, `linuxArm64`,
`tvosArm64`/`tvosSimulatorArm64`, and the `watchosArm32`/`watchosArm64`/
`watchosDeviceArm64`/`watchosSimulatorArm64` family — `sqlite-bundled` is the
driver there too, wired the way `iosMain` is below. `iosX64`, `macosX64`,
`mingwX64`, `tvosX64`, and `watchosX64` are not published by Room 3 at this
pin — a source set on one of those five targets cannot depend on `room3-runtime`
at all, regardless of driver choice.

```toml
[versions]
room = "3.0.2"        # verify latest: https://dl.google.com/android/maven2/androidx/room3/room3-runtime/maven-metadata.xml
sqlite = "2.7.0"      # verify latest: https://dl.google.com/android/maven2/androidx/sqlite/sqlite-bundled/maven-metadata.xml
ksp = "2.3.11"         # verify latest: https://github.com/google/ksp/releases — must track the project's Kotlin version

[libraries]
room-runtime   = { module = "androidx.room3:room3-runtime",   version.ref = "room" }
room-compiler  = { module = "androidx.room3:room3-compiler",  version.ref = "room" }
room-paging    = { module = "androidx.room3:room3-paging",    version.ref = "room" }
room-testing   = { module = "androidx.room3:room3-testing",   version.ref = "room" }
room-migration = { module = "androidx.room3:room3-migration", version.ref = "room" }
sqlite-bundled = { module = "androidx.sqlite:sqlite-bundled", version.ref = "sqlite" }
sqlite-web     = { module = "androidx.sqlite:sqlite-web",     version.ref = "sqlite" }

[plugins]
room = { id = "androidx.room3", version.ref = "room" }
ksp  = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

The `ksp` entry exists because the KSP wiring below resolves `libs.plugins.ksp`
— without it that reference does not compile. Pin the KSP version the
project's Kotlin plugin actually resolves against; the version above is only
a snapshot of what was current when this file was written.

## Gradle Wiring

```kotlin
plugins {
    alias(libs.plugins.room)
    alias(libs.plugins.ksp)
}

kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation(libs.room.runtime)
        }
        androidMain.dependencies {
            implementation(libs.sqlite.bundled)
        }
        jvmMain.dependencies {
            implementation(libs.sqlite.bundled)
        }
        iosMain.dependencies {
            implementation(libs.sqlite.bundled)
        }
        wasmJsMain.dependencies {
            implementation(libs.sqlite.web)
        }
        jsMain.dependencies {
            implementation(libs.sqlite.web)
        }
    }
}

room3 {
    schemaDirectory("$projectDir/schemas")
}

dependencies {
    add("kspAndroid", libs.room.compiler)
    add("kspJvm", libs.room.compiler)
    add("kspIosArm64", libs.room.compiler)
    add("kspIosSimulatorArm64", libs.room.compiler)
    add("kspWasmJs", libs.room.compiler)
    add("kspJs", libs.room.compiler)
}
```

Passing `room.schemaLocation` as an explicit KSP argument is now an error once
the `androidx.room3` plugin is applied — `schemaDirectory(...)` inside `room3
{ }` is the only supported way to set it. KSP must be added for **every**
declared target, not just the ones with obvious platform code: a missing
`add("ksp<Target>", …)` line leaves that target's `expect object
AppDatabaseConstructor` without an `actual`, and the resulting compiler error
names the `expect` declaration, not the missing KSP line — nothing points
back at the Gradle file that caused it. The `<Target>` suffix matches the
reader's own target declaration name, not a fixed list: `jvm()` gives
`kspJvm` as written above, but `jvm("desktop")` — the standard CMP template —
gives `kspDesktop` instead.

## The Driver, Per Source Set

| Source set | Driver artifact | Driver | Room 3 published? |
|---|---|---|---|
| `androidMain` | `androidx.sqlite:sqlite-bundled` | `BundledSQLiteDriver()` | yes |
| `jvmMain` (desktop) | `androidx.sqlite:sqlite-bundled` | `BundledSQLiteDriver()` | yes |
| `iosMain` (arm64, simulatorArm64) | `androidx.sqlite:sqlite-bundled` | `BundledSQLiteDriver()` | yes |
| `wasmJsMain` / `jsMain` | `androidx.sqlite:sqlite-web` | `WebWorkerSQLiteDriver` | yes |
| `iosX64`, `macosX64`, `mingwX64`, `tvosX64`, `watchosX64` | — | — | **no — Room 3 does not publish these** |

A single `commonMain` builder that calls `BundledSQLiteDriver()` unconditionally
does not compile for a web target, because `sqlite-bundled` publishes no `js`
or `wasm-js` variant — the dependency simply cannot be resolved on those source
sets. The driver is therefore supplied by the platform module, not by common
code; `commonMain` only knows `RoomDatabase.Builder<AppDatabase>`, never the
concrete driver type.

`WebWorkerSQLiteDriver` runs the SQLite engine inside a Web Worker and persists
through the Origin Private File System (OPFS). Its exact constructor — whether
it takes a `Worker` instance, a script URL, or a suspend factory — could not
be confirmed from the currently reachable documentation for this file. Check
the KDoc shipped with `androidx.sqlite:sqlite-web` 2.7.0 before writing the
`wasmJsMain` / `jsMain` builder; do not guess at the signature.

## The Database Class

```kotlin
// commonMain
@Database(entities = [ItemEntity::class], version = 1, exportSchema = true)
@ConstructedBy(AppDatabaseConstructor::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun itemDao(): ItemDao
}

// The Room compiler generates the `actual` implementation per target.
@Suppress("KotlinNoActualForExpect")
expect object AppDatabaseConstructor : RoomDatabaseConstructor<AppDatabase> {
    override fun initialize(): AppDatabase
}
```

`@ConstructedBy` is not optional in `commonMain` once any non-Android target is
declared — the compiler says *"The @Database class must be annotated with
@ConstructedBy since the source is targeting non-Android platforms."*
`@Suppress("KotlinNoActualForExpect")` on the `expect object` is required too:
the Room compiler generates the `actual` declarations at compile time, so the
Kotlin compiler never sees one at the point it checks for a matching `actual`.

## Building It

`setDriver` is mandatory on every platform, Android included; omitting it
throws *"Cannot create a RoomDatabase without providing a SQLiteDriver via
setDriver()."* at runtime, not at compile time. Common code builds, platform
code supplies the builder:

```kotlin
// commonMain
fun createAppDatabase(builder: RoomDatabase.Builder<AppDatabase>): AppDatabase = builder.build()
```

```kotlin
// androidMain
fun getDatabaseBuilder(context: Context): RoomDatabase.Builder<AppDatabase> {
    val appContext = context.applicationContext
    val dbFile = appContext.getDatabasePath("app.db")
    return Room.databaseBuilder<AppDatabase>(
        context = appContext,
        name = dbFile.absolutePath,
    ).setDriver(BundledSQLiteDriver())
}
```

```kotlin
// iosMain
fun getDatabaseBuilder(): RoomDatabase.Builder<AppDatabase> {
    val documentDirectory = requireNotNull(
        NSFileManager.defaultManager.URLForDirectory(
            directory = NSDocumentDirectory,
            inDomain = NSUserDomainMask,
            appropriateForURL = null,
            create = false,
            error = null,
        )?.path
    )
    return Room.databaseBuilder<AppDatabase>(
        name = "$documentDirectory/app.db",
    ).setDriver(BundledSQLiteDriver())
}
```

```kotlin
// jvmMain (desktop)
fun getDatabaseBuilder(): RoomDatabase.Builder<AppDatabase> {
    val appDataDir = File(System.getProperty("user.home"), ".myapp").apply { mkdirs() }
    val dbFile = File(appDataDir, "app.db")
    return Room.databaseBuilder<AppDatabase>(
        name = dbFile.absolutePath,
    ).setDriver(BundledSQLiteDriver())
}
```

```kotlin
// wasmJsMain / jsMain
fun getDatabaseBuilder(): RoomDatabase.Builder<AppDatabase> =
    Room.databaseBuilder<AppDatabase>(name = "app.db")
        // .setDriver(WebWorkerSQLiteDriver(...)) — constructor not confirmed;
        // check the androidx.sqlite:sqlite-web 2.7.0 KDoc before filling this in.
```

Only the builder factory call above is confirmed, the same shape already
verified for `iosMain`; the driver line stays commented out because its
arguments are not confirmed — see the previous section.

Never pass `Dispatchers.IO` to `setQueryCoroutineContext` in `commonMain`:
`room3-runtime` publishes `js` and `wasm-js` variants where `Dispatchers.IO`
does not exist — the platform matrix behind that rule is covered by the
compose-async skill; supply the query context, if at all, from the platform
module that already builds the driver.

## Wiring It Into Koin

`AppDatabase` and its DAOs are singletons, defined beside the builder that
creates them:

```kotlin
val databaseModule = module {
    single<AppDatabase> { createAppDatabase(get()) }
    single<ItemDao> { get<AppDatabase>().itemDao() }
}
```

The platform module provides the `RoomDatabase.Builder<AppDatabase>` that
`get()` resolves here — each platform's Koin module binds its own
`getDatabaseBuilder()` result to that type. Exactly one `AppDatabase` per
database file: a second instance throws nothing and silently stops
invalidating the first instance's `Flow`s, so a duplicate binding is a bug
that only shows up as stale UI, never as a crash.

Every pin above traces to `dl.google.com`'s `maven-metadata.xml`, not a Compose
Multiplatform release page — Room, SQLite, DataStore, and Paging are Google
KMP artifacts with no JetBrains mirror. Why that distinction decides the
coordinate you write is covered by the compose-dependencies skill.

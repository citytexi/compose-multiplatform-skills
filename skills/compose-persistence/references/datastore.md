# DataStore

DataStore is the multiplatform key-value and typed-object store — Room's
counterpart for data that isn't rows. Verify every fact below against the
artifact itself before shipping: `androidx.datastore.core.IOException`,
`androidx.datastore.core.okio.OkioStorage`, and the wasmJs actuals described
here were all confirmed by reading the `1.2.1` sources jars, not by citing a
guide page, because the official KMP DataStore page currently mismatches the
artifact on two of the points this file depends on — see the Reading and
Writing and Web sections below for exactly where.

## Choosing Between Room and DataStore

Decide this before writing anything else. A `WHERE` clause, a `JOIN`, or
roughly a hundred entries or more means Room: DataStore has no query
language, and every `edit` or typed `updateData` call rewrites the entire
file, so a growing collection is a growing write on every single change.
DataStore is for a small, flat bag of values a screen reads whole — settings,
flags, one cached timestamp — never a list that grows without bound.

## Coordinates

`datastore-core` and `datastore-preferences-core` are the `commonMain`
artifacts. `datastore-core` is the storage engine — `DataStore<T>`,
`DataStoreFactory`, `Storage<T>`. `datastore-preferences-core` is the
key-value layer built on it — `Preferences`, `PreferenceDataStoreFactory`,
the `*PreferencesKey` builders. The non-core `datastore` and
`datastore-preferences` artifacts are multiplatform too, not banned — but
each pulls in the Android-specific `File`-backed storage path as its default
and publishes no `wasmJs` variant, so a project with a web target fails to
resolve them at all. The `-core` artifacts are the default here because they
resolve on every target this skill supports, web included.

```toml
[versions]
datastore = "1.2.1"   # verify latest: https://dl.google.com/android/maven2/androidx/datastore/datastore-preferences-core/maven-metadata.xml

[libraries]
datastore-core             = { module = "androidx.datastore:datastore-core",             version.ref = "datastore" }
datastore-preferences-core = { module = "androidx.datastore:datastore-preferences-core", version.ref = "datastore" }
datastore-core-okio        = { module = "androidx.datastore:datastore-core-okio",        version.ref = "datastore" }
```

```kotlin
kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation(libs.datastore.core)
            implementation(libs.datastore.preferences.core)
        }
    }
}
```

## Creating the Instance

```kotlin
// commonMain
import androidx.datastore.core.DataStore
import androidx.datastore.core.handlers.ReplaceFileCorruptionHandler
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.PreferenceDataStoreFactory
import androidx.datastore.preferences.core.emptyPreferences
import okio.Path.Companion.toPath

const val SETTINGS_FILE = "app_settings.preferences_pb"

expect fun settingsPath(): String

fun createSettingsStore(producePath: () -> String): DataStore<Preferences> =
    PreferenceDataStoreFactory.createWithPath(
        corruptionHandler = ReplaceFileCorruptionHandler { emptyPreferences() },
        produceFile = { producePath().toPath() },
    )
```

Three details worth getting wrong once and remembering. `createWithPath`'s
parameter is named `produceFile` even though it returns an `okio.Path`, not
a file — hence the `toPath()` extension import, which is not automatic.
`OkioStorage` requires that path to be **absolute**; a relative one throws.
And the `.preferences_pb` extension is **enforced, not conventional**: the
factory's file-producing overloads run a `check()` against it and throw
`IllegalStateException`, lazily, on the first read or write — not at
`createWithPath`'s call site, which is why a wrong extension survives review
until the store is actually used. (`DataStoreFactory.create(storage =
OkioStorage(...))`, used by the typed flavour below, has no such check —
nothing forces an extension when the caller supplies `Storage<T>` directly.)

`settingsPath()` supplies the platform half — each actual joins its platform directory below with `SETTINGS_FILE`, one file per process:

| Source set | Path |
|---|---|
| `androidMain` | `context.filesDir` |
| `iosMain` | the app's documents directory (`NSDocumentDirectory`) |
| `jvmMain` | a per-OS application-data directory |
| `wasmJsMain` / `jsMain` | no filesystem — see the Web section below |

**One `DataStore` per file, per process, held as a DI singleton.** A second
instance over the same file constructs cleanly and only throws
`IllegalStateException` on the first read or write, never at construction —
`OkioStorage` tracks active files internally and checks on first use, not on
build. The guard is per-process; cross-process coordination needs
`MultiProcessDataStoreFactory`, which is Android-only and out of scope here.

```kotlin
val settingsModule = module {
    single<DataStore<Preferences>> { createSettingsStore(::settingsPath) }
    single { SettingsRepository(get()) }
}
```

> Full Koin module organization — feature-first grouping, `expect`/`actual`
> platform modules, scoping — is covered by the compose-di skill; nothing
> beyond the single binding above belongs in this file.

## Reading and Writing

```kotlin
// commonMain
import androidx.datastore.core.DataStore
import androidx.datastore.core.IOException
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.core.emptyPreferences
import androidx.datastore.preferences.core.longPreferencesKey
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.flow.map

private val LAST_SYNCED_AT = longPreferencesKey("last_synced_at")

class SettingsRepository(private val dataStore: DataStore<Preferences>) {

    val lastSyncedAt: Flow<Long?> = dataStore.data
        .catch { exception ->
            if (exception is IOException) emit(emptyPreferences()) else throw exception
        }
        .map { preferences -> preferences[LAST_SYNCED_AT] }

    suspend fun setLastSyncedAt(epochMillis: Long) {
        dataStore.edit { preferences -> preferences[LAST_SYNCED_AT] = epochMillis }
    }
}
```

`offline-first.md` reads `lastSyncedAt` to decide whether a screen needs a network refresh, so the key and both accessors live here, not re-declared downstream.

The `IOException` that `.catch` matches above is `androidx.datastore.core.IOException`
— DataStore's own `expect`/`actual` type, not `kotlinx.io.IOException`. That
is a deliberate departure from this skill's usual rule: `kotlinx.io.IOException`
is not on DataStore's dependency graph at all, so the repo-wide correction to
prefer it does not apply here. `CorruptionException` — thrown when a
serializer cannot parse what is on disk — always extends
`androidx.datastore.core.IOException`, so this `catch` reliably intercepts
it on every target: the JVM/Android actual is a typealias to the platform
`IOException`, and the native actual is DataStore's own class, unrelated to
any okio type. What it will **not** catch on native is a raw filesystem
failure — permission denied, disk full — because `OkioStorage` rethrows
`okio.IOException` verbatim rather than wrapping it, and that type shares no
supertype with `androidx.datastore.core.IOException` short of `Exception`.
The official KMP DataStore guide's samples do not cover this distinction;
this section is confirmed against the `1.2.1` `OkioStorage` and
`CorruptionException` sources directly. Skipping the `catch`, or omitting
the `corruptionHandler` above, means an unreadable or corrupted file throws
straight into the collector — the app crashes on first launch after a bad
write, instead of falling back to `emptyPreferences()`.

## Typed DataStore

A typed store skips `Preferences` and persists one `@Serializable` value
directly, through the okio serializer surface — `OkioSerializer<T>` and
`OkioStorage<T>`, both in `datastore-core-okio`, declared in the catalog
above at the same `datastore` version. `T` must be immutable, same rule
Room's converters enforce for entities.

```kotlin
// commonMain
import androidx.datastore.core.CorruptionException
import androidx.datastore.core.DataStore
import androidx.datastore.core.DataStoreFactory
import androidx.datastore.core.okio.OkioStorage
import androidx.datastore.core.okio.OkioSerializer
import kotlinx.serialization.SerializationException
import kotlinx.serialization.json.Json
import okio.BufferedSink
import okio.BufferedSource
import okio.FileSystem
import okio.Path.Companion.toPath

@kotlinx.serialization.Serializable
data class UserSettings(val theme: String = "system", val notificationsEnabled: Boolean = true)

object UserSettingsSerializer : OkioSerializer<UserSettings> {
    override val defaultValue: UserSettings = UserSettings()

    override suspend fun readFrom(source: BufferedSource): UserSettings =
        try {
            Json.decodeFromString(UserSettings.serializer(), source.readUtf8())
        } catch (exception: SerializationException) {
            throw CorruptionException("Cannot read user settings", exception)
        }

    override suspend fun writeTo(t: UserSettings, sink: BufferedSink) {
        sink.writeUtf8(Json.encodeToString(UserSettings.serializer(), t))
    }
}

fun createUserSettingsStore(producePath: () -> String): DataStore<UserSettings> =
    DataStoreFactory.create(
        storage = OkioStorage(
            fileSystem = FileSystem.SYSTEM,
            serializer = UserSettingsSerializer,
            producePath = { producePath().toPath() },
        ),
    )
```

The upstream JVM-era `Serializer<T>` interface reads and writes through a stream pair that has no Kotlin/Native equivalent — none of it ports by substitution, which is why the multiplatform surface above trades that pair for `BufferedSource`/`BufferedSink` and a serializer that throws `CorruptionException` itself, by hand, instead of relying on a caught parse exception from the platform stream types.

For `Preferences` specifically, map to a domain type at the repository
boundary before anything above the data layer sees it — a raw `Preferences`
reaching a ViewModel is the same violation as a `RoomEntity` reaching one.
The typed flavour above gets this for free: the serializer already hands
back `UserSettings`, never a bag of untyped keys.

## Web

Worse than merely unimplemented. Stable `1.2.1` does publish a `wasmJs`
artifact, but its `PreferenceDataStoreFactory` actuals — both `create` and
`createWithPath` — are bodies consisting only of the stdlib's not-yet-implemented
stub: the code compiles and links, then throws `NotImplementedError` the
first time it runs. There is no plain `js` artifact at this pin at all; real web storage first ships in the
`1.3.0-alpha` line. The official DataStore KMP page pins `1.2.1` and then
shows a `jsMain` sample built on a class that does not exist at that
version — do not cite that page for the web story, and re-verify the
`1.3.0` behavior directly from its sources before relying on it.

Until this project's minimum DataStore version moves off `1.2.1`, a web target has Room but no working key-value store on stable DataStore. Put settings access behind an `expect`/`actual` interface and give the `wasmJsMain` / `jsMain` actual its own storage — browser `localStorage` through an interop call, for instance — rather than routing it through `PreferenceDataStoreFactory`.

## Migrating From SharedPreferences (Android only)

`SharedPreferencesMigration` and the `Context.preferencesDataStore` delegate
both exist to move data out of a legacy Android `SharedPreferences` file and
into DataStore during a migration window — both are Android-only APIs and
belong nowhere near `commonMain`. `SharedPreferences` itself is not a target
for new code in a multiplatform project.

> Bearer tokens are one of the things a typed `DataStore` is asked to hold.
> The token storage interface and the refresh cycle that reads it are
> covered by the compose-networking skill; this file only says which
> DataStore flavour backs it and where the file lands per platform.

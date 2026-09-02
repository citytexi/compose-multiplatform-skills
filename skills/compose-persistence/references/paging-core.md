# Paging: Sources and Pagers

Paging 3 loads a list in chunks instead of all at once. This file covers
the two ways to produce a `PagingSource`, the `Pager` that drives one, and
the one place a `PagingSource<Int, ItemEntity>` becomes `PagingData<Item>`
before anything above the data layer sees it. [room-setup.md](room-setup.md)
and [schema-dao.md](schema-dao.md) set up the `AppDatabase` and the
`ItemDao` this file builds on; read those first if `ItemDao`, `ItemEntity`,
or `toDomain()` are unfamiliar.

## Coordinates

`paging-common`, `paging-compose`, and `paging-testing` are the artifacts
this skill uses, each verified against its own `3.5.1` `.module` file.

```toml
[versions]
paging = "3.5.1"   # verify latest: https://dl.google.com/android/maven2/androidx/paging/paging-common/maven-metadata.xml

[libraries]
paging-common  = { module = "androidx.paging:paging-common",  version.ref = "paging" }
paging-compose = { module = "androidx.paging:paging-compose", version.ref = "paging" }
paging-testing = { module = "androidx.paging:paging-testing", version.ref = "paging" }
```

`paging-common` and `paging-compose` publish a Gradle module variant named
`desktop` for the JVM desktop target; `paging-testing` publishes one named
`jvm` instead. Depend on the base artifact and let the Kotlin Multiplatform
plugin resolve the right variant — never hand-write a `-desktop` or `-jvm`
suffix onto any of these three artifactIds. The guard is not theoretical:
`paging-common-jvm` is a real, stale coordinate that stopped publishing
after `3.4.0-alpha02`, and `paging-compose-jvm` was never published at all.

Target reach, confirmed from `paging-common-3.5.1.module` and
`paging-compose-3.5.1.module` on 2026-09-01: this pin publishes `mingwX64` —
the one native target Room 3 does not reach at all, per
[room2-to-room3.md](room2-to-room3.md) — but none of the Intel/legacy
`iosX64`, `macosX64`, `tvosX64`, `watchosX64`. It does publish their arm64
counterparts (`iosArm64`, `iosSimulatorArm64`, `macosArm64`, `tvosArm64`,
`tvosSimulatorArm64`, the `watchos*Arm*` family) plus `linuxArm64`,
`linuxX64`, `js`, `wasmJs`, and `android`.

Two more traps. `paging-runtime`, `paging-guava`, and `paging-rxjava3`
publish an Android AAR only — each `.module` lists a single Android
`release` variant — so depending on any from `commonMain` fails variant
resolution outright; they adapt `PagingData` to a `RecyclerView` adapter, a
`ListenableFuture`, or RxJava, none of which a Compose app needs. And
`paging-compose:3.5.1` pulls `androidx.compose.runtime:runtime` transitively
into the desktop variant — benign, it resolves to the Compose Multiplatform
anchor, but the diagnosis procedure for a Google KMP artifact pulling a
non-`org.jetbrains.compose` one belongs to the compose-dependencies skill.

## PagingSource By Hand

A hand-written source suits a network-only feed with no local cache — the
Room-backed source next is what an offline-first app actually uses.
`getRefreshKey(state)` is an abstract member of `PagingSource`; a subclass that
omits it does not compile. `load()` must rethrow `CancellationException` before
reaching for `LoadResult.Error`, or scroll cancellation gets reported to the UI
as a load failure instead of silently stopping:

```kotlin
import androidx.paging.PagingSource
import androidx.paging.PagingSource.LoadParams
import androidx.paging.PagingSource.LoadResult
import androidx.paging.PagingState
import io.ktor.client.HttpClient
import io.ktor.client.call.body
import io.ktor.client.request.get
import io.ktor.client.request.parameter
import kotlinx.coroutines.CancellationException

class ItemSearchPagingSource(
    private val client: HttpClient,
    private val query: String,
) : PagingSource<Int, ItemDto>() {

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, ItemDto> {
        val page = params.key ?: 1
        return try {
            val response: List<ItemDto> = client.get("items") {
                parameter("q", query)
                parameter("page", page)
                parameter("size", params.loadSize)
            }.body()
            LoadResult.Page(
                data = response,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.isEmpty()) null else page + 1,
            )
        } catch (e: CancellationException) {
            throw e
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }

    override fun getRefreshKey(state: PagingState<Int, ItemDto>): Int? {
        val anchor = state.anchorPosition ?: return null
        val page = state.closestPageToPosition(anchor)
        return page?.prevKey?.plus(1) ?: page?.nextKey?.minus(1)
    }
}
```

Two rules that are easy to get right once and violate silently later. First,
the factory passed to `Pager` must produce a **new** instance on every
invocation — Paging recreates the source whenever the underlying data
changes, and reusing one throws *"An instance of PagingSource was re-used
when a new instance was expected … recreate your PagingSource"* on the next
collection. `{ ItemSearchPagingSource(client, query) }` as a lambda
satisfies this; capturing one instance in a `val` does not. Second, a
hand-written source must not stay typed on a DTO past the repository
boundary — the same violation as an entity escaping the data layer.
`ItemDto` stops where this source's results get mapped, same as
`ItemEntity` below.

## PagingSource From A DAO

The Room-backed source, added to the `ItemDao` that
[schema-dao.md](schema-dao.md) already declares — not a second `@Dao
interface ItemDao { … }` block, which ships two incompatible declarations:

```kotlin
// added to the ItemDao interface in schema-dao.md
@Query("SELECT * FROM items ORDER BY createdAt DESC")
fun pagingSource(): PagingSource<Int, ItemEntity>
```

That interface carries the class-level annotation
`@DaoReturnTypeConverters(PagingSourceDaoReturnTypeConverter::class)`, which
makes a `PagingSource` return type legal on a DAO function. The annotation,
`androidx.room3.DaoReturnTypeConverters`, ships in
`androidx.room3:room3-common` and takes `value: Array<KClass<*>>` — confirmed
by unpacking `room3-common-jvm-3.0.2.jar`, which ships
`DaoReturnTypeConverters.class`. The converter it names,
`androidx.room3.paging.PagingSourceDaoReturnTypeConverter`, ships in
`androidx.room3:room3-paging`, listed at the same `3.0.2` pin in
[room-setup.md](room-setup.md)'s coordinate table, not in `room3-runtime`. Room
2 baked this support into `room-runtime` directly, so omitting `room3-paging`
leaves `pagingSource()` uncompilable even though the call site matches Room 2.

`RoomDatabase.Builder<T>` also exposes `addDaoReturnTypeConverter(instance:
Any)` for a converter marked `@ProvidedDaoReturnTypeConverter`, which needs a
runtime-constructed instance. `PagingSourceDaoReturnTypeConverter` has a public
no-arg constructor and no such annotation, so Room instantiates it itself —
nothing to register.

## Pager

```kotlin
enum class ItemFilter { ALL, ACTIVE, ARCHIVED }

private val config = PagingConfig(
    pageSize = 20,
    enablePlaceholders = false,
)
```

| Parameter | What it controls | Default if omitted |
|---|---|---|
| `pageSize` | rows loaded per page | required, no default |
| `prefetchDistance` | how many rows from the end of loaded data triggers the next page | `pageSize` |
| `initialLoadSize` | rows loaded on the very first page | `pageSize * 3` |
| `enablePlaceholders` | whether unloaded rows render as `null` placeholders sized to the true item count | `true` |

Restating `prefetchDistance` or `initialLoadSize` at defaults is noise: the
table marks a diverging value as deliberate. Two constructor-time failures
surface only at runtime: `enablePlaceholders = false` with `prefetchDistance =
0` throws `IllegalArgumentException` — with no placeholders there's no other
"load more" signal — and `maxSize`, when set, must be at least `pageSize +
prefetchDistance * 2`, or Paging can't hold a full prefetch window on both
sides of the loaded page.

## Mapping Before It Leaves

This is the only place a `PagingSource<Int, ItemEntity>` is allowed to stay
typed on the entity — inside the repository, never past it. The entity is
a *type parameter* threaded through `Pager<Int, ItemEntity>`, so a
repository that returns `Flow<PagingData<ItemEntity>>` puts a storage type
in every consumer's function signature, the same violation `toDomain()` in
[schema-dao.md](schema-dao.md) exists to prevent for a plain list.
`PagingData.map` is the one place to cut it:

```kotlin
import androidx.paging.Pager
import androidx.paging.PagingData
import androidx.paging.map
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map

class ItemPagingRepository(private val dao: ItemDao) {

    fun pagedItems(): Flow<PagingData<Item>> =
        Pager(config = config, pagingSourceFactory = { dao.pagingSource() })
            .flow
            .map { pagingData -> pagingData.map { it.toDomain() } }

    fun pagedItems(query: String, filter: ItemFilter): Flow<PagingData<Item>> =
        Pager(config = config, pagingSourceFactory = { dao.pagingSource(query, filter) })
            .flow
            .map { pagingData -> pagingData.map { it.toDomain() } }
}
```

Both `.map` calls above need their own import, and the two resolve on
different receivers. `PagingData.map` — the inner call — is declared in the
top-level `androidx.paging.PagingDataTransforms` file, package
`androidx.paging`; skip `import androidx.paging.map` and it silently
resolves to `Flow.map` instead, with the compiler error naming `Flow`'s
`map`, not the missing import. The outer `.flow.map { ... }` is that same
`Flow.map`, from `kotlinx.coroutines.flow.map` — easy to assume covered once
a `map` import already sits above it, but it is not.

The second overload assumes a `pagingSource(query: String, filter:
ItemFilter): PagingSource<Int, ItemEntity>` also exists on `ItemDao` — a
`@Query` with a `LIKE` clause and a `CASE`/`WHEN` on `filter`, the same
shape as any other parameterized query in [schema-dao.md](schema-dao.md).
Both signatures are fixed here because later references in this skill
call them by name.

Bind the repository beside the construct, the same shape
[room-setup.md](room-setup.md) and [datastore.md](datastore.md) use for
theirs:

```kotlin
val pagingModule = module {
    single { ItemPagingRepository(get()) }
}
```

Order matters once `cachedIn` enters the chain: the mapping above must run
*before* `cachedIn(scope)`, wherever the caller applies it, because
`cachedIn` only caches the `PagingData` stream as it exists at the point
it's called. A transformation applied after `cachedIn` is not lost — it
isn't cached, so it re-executes on every new collection of that cached
stream: wasted work, not a correctness bug, contrary to the upstream
source's "lost on cache hit" — but still why the mapping belongs inside the
repository, ahead of wherever `cachedIn` gets applied.

## Invalidation

A Room-backed `PagingSource` invalidates itself automatically: Room's
invalidation tracker watches the `items` table, and any write through
`ItemDao` — `upsertAll`, `delete`, `replaceAll` — causes the next `Pager`
collection to load a fresh source. This is the main reason to prefer the
DAO-backed source wherever the data has a local table to watch.

A hand-written source gets none of that. After a mutation the owning
repository must call `invalidate()` on the current `PagingSource` instance
itself — there is no table for it to observe. The trap in the obvious fix is
the same re-use crash the factory rule above exists to prevent: holding "the
current instance" in a field is one edit away from handing that same
field's value back to `pagingSourceFactory` next collection instead of a
fresh one. Keep the field to invalidate it; never let it also be what the
factory returns.

> Paging's transform surface and the `Flow` a `LazyColumn` collects are
> covered in [paging-ui.md](paging-ui.md). This file stops at the repository
> boundary.

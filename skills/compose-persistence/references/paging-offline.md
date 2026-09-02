# Offline Paging with RemoteMediator

> Why the database is the source of truth, how the DTO becomes an entity, and
> where the staleness stamp lives are covered by
> [offline-first.md](offline-first.md). This file covers only the mechanics
> `RemoteMediator` adds: load types, remote keys, and end-of-pagination.

## The Experimental Opt-In

`RemoteMediator` is annotated `@ExperimentalPagingApi`, and that opt-in is
error-level: a class extending `RemoteMediator` without
`@OptIn(ExperimentalPagingApi::class)` fails to compile, it does not merely
warn. Every use site here needs it — the mediator class and the `Pager` call
that wires it in below — because the API is not frozen.

## Remote Keys

`RemoteMediator` needs somewhere to persist what page comes next, and that
cannot be a column on `ItemEntity`. `ItemRemoteKeys` describes the network's
pagination, not the item — cleared on every `REFRESH` while the items it
describes are replaced in the same breath, which on `ItemEntity` would race
a column clear against deleting the row that carries it.

```kotlin
import androidx.room3.Entity
import androidx.room3.PrimaryKey

@Entity(tableName = "item_remote_keys")
data class ItemRemoteKeys(
    @PrimaryKey val itemId: String,
    val nextKey: Int?,
    val lastUpdated: Long,
)
```

`lastUpdated` is what `initialize()` below reads to decide whether the cache
is still fresh. Its own DAO:

```kotlin
import androidx.room3.Dao
import androidx.room3.Query
import androidx.room3.Upsert

@Dao
interface ItemRemoteKeysDao {
    @Upsert
    suspend fun upsertAll(keys: List<ItemRemoteKeys>)

    @Query("SELECT * FROM item_remote_keys WHERE itemId = :itemId")
    suspend fun remoteKeysByItemId(itemId: String): ItemRemoteKeys?

    @Query("SELECT MAX(lastUpdated) FROM item_remote_keys")
    suspend fun lastUpdated(): Long?

    @Query("DELETE FROM item_remote_keys")
    suspend fun deleteAll()
}
```

`@Upsert`, not `@Insert(onConflict = OnConflictStrategy.REPLACE)` — the same
rule [schema-dao.md](schema-dao.md#dao-functions) states for `ItemDao`
applies here without exception: `REPLACE` deletes the existing row before
inserting the new one, and `@Upsert` costs one word to avoid ever having a
delete-then-insert window to reason about.

Adding a second entity means editing the `@Database` declaration from
[room-setup.md](room-setup.md#the-database-class) to `entities =
[ItemEntity::class, ItemRemoteKeys::class]`, plus `abstract fun
itemRemoteKeysDao(): ItemRemoteKeysDao` beside the existing `itemDao()`. That
also bumps `version` and needs a migration — a plain `@AutoMigration` covers
a brand-new table — per [migrations.md](migrations.md). Skip either half and
the failure names neither: Room's own error points at "cannot find
implementation" or a schema-export mismatch, never back at this file.

Keying `ItemRemoteKeys` by `itemId` alone assumes one unfiltered feed. The
filtered `pagedItems(query, filter)` overload in
[paging-core.md](paging-core.md) is out of scope: mediating it needs its own
keys shape — a composite key, or a query column — or two differently
filtered `Pager`s overwrite each other's cursor on the same item.

## initialize()

`initialize()` decides whether the very first collection of the `Pager`
waits for a network refresh or serves whatever Room already has. The
decision is a cache window, not a network probe, and the timestamp behind it
comes from `kotlin.time`, never a raw millisecond read off the JVM's clock:

```kotlin
import kotlin.time.Clock
import kotlin.time.Duration.Companion.hours
import kotlin.time.Instant

private val CACHE_TIMEOUT = 1.hours

override suspend fun initialize(): InitializeAction {
    val lastUpdated = remoteKeysDao.lastUpdated()
    val isStale = lastUpdated == null ||
        Clock.System.now() - Instant.fromEpochMilliseconds(lastUpdated) > CACHE_TIMEOUT
    return if (isStale) InitializeAction.LAUNCH_INITIAL_REFRESH else InitializeAction.SKIP_INITIAL_REFRESH
}
```

`Clock` and `Instant` moved from `kotlinx-datetime` into `kotlin.time` and
are stable — no `@OptIn(ExperimentalTime::class)` needed — at the `2.3.x`
Kotlin line [room-setup.md](room-setup.md)'s `ksp` coordinate pins this
skill to; check an older project version before dropping that opt-in.
`Instant - Instant` returns a `Duration` directly, so the comparison
against `CACHE_TIMEOUT` needs no manual unit conversion. `InitializeAction`
resolves unqualified because it is nested inside the `RemoteMediator`
superclass this mediator extends.

`CACHE_TIMEOUT` here and `lastSyncedAt` in [offline-first.md](offline-first.md)
are two deliberately separate staleness clocks, not a duplicate: this one is
intrinsic to the `RemoteMediator` contract and gates `initialize()` alone; that
one gates a plain repository's `refresh()`. An app wiring both answers "is it
stale" twice on purpose, once per caller.

## load()

One function handles all three load types. `REFRESH` starts from the first
page; `PREPEND` never has anything to load since paging only walks forward
through this feed; `APPEND` reads the key stored against the last loaded item:

```kotlin
import androidx.paging.LoadType
import androidx.paging.PagingState
import kotlin.time.Clock
import kotlinx.coroutines.CancellationException

override suspend fun load(
    loadType: LoadType,
    state: PagingState<Int, ItemEntity>,
): MediatorResult {
    return try {
        val page = when (loadType) {
            LoadType.REFRESH -> STARTING_PAGE
            LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
            LoadType.APPEND -> {
                val lastItem = state.lastItemOrNull()
                    ?: return MediatorResult.Success(endOfPaginationReached = true)
                remoteKeysDao.remoteKeysByItemId(lastItem.id)?.nextKey
                    ?: return MediatorResult.Success(endOfPaginationReached = true)
            }
        }

        val response = api.getItems(page = page, limit = state.config.pageSize)
        val endOfPaginationReached = response.items.isEmpty()
        val nextKey = if (endOfPaginationReached) null else page + 1
        val now = Clock.System.now().toEpochMilliseconds()

        database.useWriterConnection { transactor ->
            transactor.immediateTransaction {
                if (loadType == LoadType.REFRESH) {
                    remoteKeysDao.deleteAll()
                    itemDao.deleteAll()
                }
                itemDao.upsertAll(response.items.map { it.toEntity() })
                remoteKeysDao.upsertAll(
                    response.items.map { dto -> ItemRemoteKeys(itemId = dto.id, nextKey = nextKey, lastUpdated = now) },
                )
            }
        }

        MediatorResult.Success(endOfPaginationReached = endOfPaginationReached)
    } catch (e: CancellationException) {
        throw e
    } catch (e: Exception) {
        MediatorResult.Error(e)
    }
}

private companion object {
    const val STARTING_PAGE = 1
}
```

`PREPEND` returning `endOfPaginationReached = true` unconditionally is a
choice specific to this feed shape: an append-only list with a single stable
first page. A bidirectional cursor feed would need `state.firstItemOrNull()`
and a real prepend key the same way `APPEND` reads one; out of scope here.

`itemDao.upsertAll(...)` and `remoteKeysDao.upsertAll(...)` run inside one
`transactor.immediateTransaction { }` — the writer-connection API
[schema-dao.md](schema-dao.md#transactions) documents, not the Android-only
`RoomDatabase` shorthand missing from `commonMain`. Splitting the two writes
risks a process death between them: a `REFRESH` that cleared and rewrote
`items` but not yet `ItemRemoteKeys` leaves keys pointing at rows that no
longer exist, or at rows never written — the same reason
[offline-first.md](offline-first.md) shares one `@Transaction` across
`replaceAll`'s delete and reinsert.

## Wiring The Pager

`ItemRemoteMediator`'s constructor takes `ItemApi` from the
compose-networking skill and `AppDatabase` from
[room-setup.md](room-setup.md):

```kotlin
import androidx.paging.ExperimentalPagingApi
import androidx.paging.RemoteMediator

@OptIn(ExperimentalPagingApi::class)
class ItemRemoteMediator(
    private val api: ItemApi,
    private val database: AppDatabase,
) : RemoteMediator<Int, ItemEntity>() {
    private val itemDao = database.itemDao()
    private val remoteKeysDao = database.itemRemoteKeysDao()

    // initialize() and load() above
}
```

The repository that owns it is a distinct class from the local-only
`ItemPagingRepository` in [paging-core.md](paging-core.md), which has no
mediator and wraps `ItemDao` alone:

```kotlin
import androidx.paging.ExperimentalPagingApi
import androidx.paging.Pager
import androidx.paging.PagingConfig
import androidx.paging.PagingData
import androidx.paging.map
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map

@OptIn(ExperimentalPagingApi::class)
class OfflineFirstItemPagingRepository(
    private val api: ItemApi,
    private val database: AppDatabase,
) {
    private val config = PagingConfig(pageSize = 20, enablePlaceholders = false)

    fun pagedItems(): Flow<PagingData<Item>> =
        Pager(
            config = config,
            remoteMediator = ItemRemoteMediator(api = api, database = database),
            pagingSourceFactory = { database.itemDao().pagingSource() },
        ).flow.map { pagingData -> pagingData.map { it.toDomain() } }
}
```

`config` is restated here rather than pointed at [paging-core.md](paging-core.md)'s
value, private to that file's own repository. The `Pager` overload accepting
`remoteMediator` carries `@ExperimentalPagingApi` and has no default for that
parameter, so every argument above is named rather than relying on order.
`pagedItems()` returns `Flow<PagingData<Item>>`, never
`Flow<PagingData<ItemEntity>>`: `.map { it.toDomain() }` keeps the entity
from leaving the data layer, the boundary
[paging-core.md](paging-core.md#mapping-before-it-leaves) draws.

## Errors

A failed `load()` becomes `MediatorResult.Error(e)`, built directly from
whatever `api.getItems()` threw. This file does not classify that exception
— including why `kotlinx.io.IOException` is the only `IOException` nameable
in `commonMain` — that belongs to compose-networking. The upstream source
catches Retrofit's `HttpException` here, an import this library has
nowhere: Retrofit is excluded by stance, and `ItemApi` wraps Ktor, never
Retrofit.

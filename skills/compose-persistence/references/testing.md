# Testing Persistence

A fake `ItemDao` or a fake `DataStore<Preferences>` proves the calling code
compiles against an interface; it proves nothing about the query, the index,
or the file format underneath. Every fixture below builds the real thing —
a real `AppDatabase`, a real `DataStore<Preferences>` — pointed at a location
private to that one test. What changes from the production wiring in
[room-setup.md](room-setup.md) and [datastore.md](datastore.md) is only the
location and the lifecycle; the driver and the store shape stay exactly what
those files already define.

## A Real Database Per Test

`Room.inMemoryDatabaseBuilder<AppDatabase>()` and
`Room.databaseBuilder<AppDatabase>(name = ...)` are actuals of the same
`expect object Room` [room-setup.md](room-setup.md) builds `AppDatabase`
from — confirmed by reading the `room3-runtime-jvm` and `room3-runtime-android`
`3.0.2` sources, where the reified, context-free `inMemoryDatabaseBuilder<T>()`
overload exists on both, unlike the Android-only
`Room.inMemoryDatabaseBuilder(context, klass)` shape an Android 2.x sample
shows. Neither factory is a member of `commonMain`'s `expect object Room`,
so a `commonTest` fixture cannot call either — the test builder lives in the
same platform test source set as the production builder it mirrors, `jvmTest`
for desktop, matching the per-source-set shape [room-setup.md](room-setup.md)
already uses for `getDatabaseBuilder()`. `setDriver(...)` is not optional
here either: a test database that skips it fails with the same *"Cannot
create a RoomDatabase without providing a SQLiteDriver via setDriver()"*
message production code gets. The fixture below also builds
`ItemPagingRepository` from the same `dao`, reused by later sections.

```kotlin
// jvmTest
import androidx.room3.Room
import androidx.sqlite.driver.bundled.BundledSQLiteDriver
import app.cash.turbine.test
import kotlin.test.AfterTest
import kotlin.test.BeforeTest
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlinx.coroutines.test.runTest

class ItemPersistenceTest {

    private lateinit var database: AppDatabase
    private lateinit var dao: ItemDao
    private lateinit var repository: ItemPagingRepository

    @BeforeTest
    fun createDatabase() {
        val builder = Room.inMemoryDatabaseBuilder<AppDatabase>()
            .setDriver(BundledSQLiteDriver())
        database = createAppDatabase(builder)
        dao = database.itemDao()
        repository = ItemPagingRepository(dao)
    }

    @AfterTest
    fun closeDatabase() = database.close()

    @Test
    fun replaceAll_thenObserveAll_emitsInsertedRow() = runTest {
        dao.observeAll().test {
            assertEquals(emptyList(), awaitItem())
            dao.replaceAll(listOf(ItemEntity(id = "1", name = "One", status = ItemStatus.ACTIVE, createdAt = 1L)))
            assertEquals("1", awaitItem().single().id)
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

Every `// added to ItemPersistenceTest above` block below extends this class and inherits its imports, listing only what it newly needs.

Build a fresh `AppDatabase` in `@BeforeTest`, close it in `@AfterTest`; never
share one instance, in-memory or file-backed, across two `@Test` functions.
The isolation rule is the one [room-setup.md](room-setup.md) states for
production, applied to a test: one database instance per test, over a path
unique to that test. `Room.inMemoryDatabaseBuilder<AppDatabase>()` gives every
call an unnamed, private connection, so a fresh builder per test already
satisfies this for free. A file-backed fixture does not — it needs its own
uniquely-named temp path per test, or two tests sharing one path reproduce
[room-setup.md](room-setup.md)'s duplicate-instance bug: a second `AppDatabase`
over a path another instance already holds doesn't throw, it silently stops
invalidating the first instance's `Flow`s, which in a test looks like a
`Flow` that never re-emits after a write, not like a fixture mistake.

## A Real DataStore Per Test

The same shape applies to `DataStore`: build it in `@BeforeTest` against a
path unique to that test, using the `createWithPath` factory
[datastore.md](datastore.md) defines for production, pointed at a temp path
instead of `settingsPath()`. The upstream test fixture this pattern replaces
builds its `DataStore` on the JVM-only `File`-backed `create()` overload,
which does not exist on Kotlin/Native or web. Nothing here needs that type:
a path is a `String`, and `createSettingsStore` already turns one into an
`okio.Path` internally.

```kotlin
// jvmTest
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.Preferences
import kotlin.random.Random
import kotlin.test.BeforeTest
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.test.runTest

class SettingsRepositoryTest {

    private lateinit var repository: SettingsRepository

    @BeforeTest
    fun createStore() {
        val testPath = "${System.getProperty("java.io.tmpdir")}/test-${Random.nextLong()}-$SETTINGS_FILE"
        val dataStore: DataStore<Preferences> = createSettingsStore { testPath }
        repository = SettingsRepository(dataStore)
    }

    @Test
    fun setLastSyncedAt_thenRead_returnsWrittenValue() = runTest {
        repository.setLastSyncedAt(42L)
        assertEquals(42L, repository.lastSyncedAt.first())
    }
}
```

Unlike `AppDatabase`, `DataStore` exposes no `close()` for teardown — the
per-process guard `datastore.md` describes is keyed on the file path itself,
not an open handle, so the unique path built in `@BeforeTest` is what does
the isolating work here, not a teardown step. Give two tests the same path
instead — hardcoding `SETTINGS_FILE` rather than building `testPath` per test
— and the failure does not show up where it was introduced: the first test's
read passes normally, and the second test's *first* read throws
`IllegalStateException` from `OkioStorage`'s active-file tracking, since that
guard checks on first use, not construction. That reads as flaky test
ordering, not as what it is — a fixture reusing a path a still-open
`DataStore` already claimed.

## Testing a PagingSource

A `PagingSource` needs no `Pager` to test — call `load()` directly with a
`LoadParams.Refresh`, and assert on the `LoadResult.Page` it returns:

```kotlin
// added to ItemPersistenceTest above
import androidx.paging.PagingSource.LoadParams
import androidx.paging.PagingSource.LoadResult
import kotlin.test.assertNull

@Test
fun load_refresh_returnsPageWithData() = runTest {
    dao.replaceAll(listOf(
        ItemEntity(id = "item-1", name = "One", status = ItemStatus.ACTIVE, createdAt = 2L),
        ItemEntity(id = "item-2", name = "Two", status = ItemStatus.ACTIVE, createdAt = 1L),
    ))

    val result = dao.pagingSource().load(
        LoadParams.Refresh(key = null, loadSize = 20, placeholdersEnabled = false),
    )

    check(result is LoadResult.Page)
    assertEquals(listOf("item-1", "item-2"), result.data.map { it.id })
    assertNull(result.prevKey)
    assertNull(result.nextKey)
}
```

Two rows, one page, both keys `null`: nothing left to load in either
direction. Seed more rows than `loadSize` and `nextKey` stops being `null` —
that boundary is what the next section drives across more than one load.

## Testing the Paged Flow

`TestPager` and `asSnapshot` are both declared in `paging-testing`'s
`commonMain` — confirmed by reading the `3.5.1` sources directly, not a
guide page — with no experimental or platform-restricting annotation on
either, so both are usable from a shared test wherever `paging-testing`
itself resolves, per the target reach [paging-core.md](paging-core.md)
already states. `TestPager` drives one `PagingSource` instance directly,
simulating a `refresh()` followed by an `append()`:

```kotlin
// added to ItemPersistenceTest above
import androidx.paging.PagingConfig
import androidx.paging.testing.TestPager

@Test
fun testPager_refreshThenAppend_loadsSuccessivePages() = runTest {
    dao.replaceAll(
        (0 until 10).map { index ->
            ItemEntity(id = "item-${index.toString().padStart(2, '0')}", name = "Item $index", status = ItemStatus.ACTIVE, createdAt = 9L - index)
        },
    )
    val pager = TestPager(
        config = PagingConfig(pageSize = 4, initialLoadSize = 4, enablePlaceholders = false),
        pagingSource = dao.pagingSource(),
    )

    val refreshed = pager.refresh()
    val appended = pager.append()

    check(refreshed is LoadResult.Page && appended is LoadResult.Page)
    assertEquals(listOf("item-00", "item-01", "item-02", "item-03"), refreshed.data.map { it.id })
    assertEquals(listOf("item-04", "item-05", "item-06", "item-07"), appended.data.map { it.id })
}
```

`asSnapshot` drives the other end: `Flow<PagingData<Item>>`, exactly what
`ItemPagingRepository.pagedItems()` hands a consumer, replaying a refresh
plus a scripted scroll and returning the flattened, domain-mapped list a
`LazyColumn` would have rendered:

```kotlin
// added to ItemPersistenceTest above
import androidx.paging.testing.asSnapshot
import kotlin.test.assertTrue

@Test
fun pagedItems_afterRefreshAndAppend_showsLoadedItems() = runTest {
    dao.replaceAll(
        (0 until 70).map { index ->
            ItemEntity(id = "item-${index.toString().padStart(3, '0')}", name = "Item $index", status = ItemStatus.ACTIVE, createdAt = 69L - index)
        },
    )

    val loaded = repository.pagedItems().asSnapshot {
        appendScrollWhile { it.id != "item-065" }
    }

    assertEquals("item-000", loaded.first().id)
    assertTrue(loaded.size > 60)
    assertTrue(loaded.any { it.id == "item-065" })
}
```

`ItemPagingRepository.pagedItems()` carries [paging-core.md](paging-core.md)'s
default `PagingConfig`, whose `initialLoadSize` defaults to `pageSize * 3` —
so a single `refresh()` alone already covers 60 of the 70 seeded rows. The
assertions only claim what `appendScrollWhile` guarantees — the snapshot
starts at the first row and reaches past that initial batch — rather than
pin down the exact final row, which would assert on `appendScrollWhile`'s
page-boundary rounding, a `paging-testing` implementation detail this file
does not need.

Migration testing is not here: it lives in [migrations.md](migrations.md),
in the platform test source set its helper's constructor forces — `jvmTest`
for desktop, `androidTest` for Android — because that helper has no shape a
shared test could build once.

> The Turbine API used above — `.test { }`, `awaitItem()`,
> `cancelAndIgnoreRemainingEvents()` — and `runTest`'s virtual-time behaviour
> are covered by the compose-async skill; testing a ViewModel's event, state,
> and effect cycle is covered by the compose-architecture skill. This file
> covers only what a real database and a real file add on top of those:
> instance isolation, temporary paths, and an explicit driver.

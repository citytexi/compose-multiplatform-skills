---
name: compose-persistence
description: Use when adding or changing local storage in a Compose Multiplatform app — a Room 3 database, DAOs and migrations, DataStore settings, Paging 3 lists, or an offline-first repository backed by a local database.
---

# Compose Multiplatform Persistence

Relational data and queries go to Room 3; key-value and typed settings go to
DataStore; large lists go to Paging 3. A `WHERE` clause, a `JOIN`, or roughly
a hundred entries or more means Room, not DataStore — every DataStore `edit`
rewrites the whole file, so a growing collection is a growing write on every
change.

Room 3 is the single local-database default here. SQLDelight is the right
call for two projects and only two: one whose targets include `iosX64`,
`macosX64`, `mingwX64`, `tvosX64`, or `watchosX64`, which Room 3 no longer
publishes, and one that needs a dialect other than SQLite. Neither is this
skill's subject.

<HARD-GATE>
- Every DAO function compiled for a non-Android target is `suspend`, returns
  `Flow`, or returns a type registered with `@DaoReturnTypeConverters` —
  non-abstract `@Transaction` methods included. Anything else fails the
  build: "Only suspend functions are allowed in DAOs declared in source sets
  targeting non-Android platforms." A DAO in `androidMain` is exempt; a DAO
  in `commonMain` is not, even in a project with only an Android target today.
- Every `RoomDatabase.Builder` ends in `.setDriver(...)` — every platform,
  in-memory test databases included; omitting it throws "Cannot create a
  RoomDatabase without providing a SQLiteDriver via setDriver()." Every
  `commonMain` `@Database` carries `@ConstructedBy(...)`.
- ONE instance per storage file, held as a DI singleton — one `RoomDatabase`
  per database file, one `DataStore` per preferences file. A second
  `DataStore` throws `IllegalStateException` on the first read or write; a
  second `RoomDatabase` throws nothing and silently stops invalidating the
  other instance's `Flow`s.
- Every `@Database(version = N)` bump ships a `Migration` or an
  `@AutoMigration`. `fallbackToDestructiveMigration()` deletes the user's
  data and is acceptable only on a database whose entire contents are
  re-fetchable, declared with a comment.
- NEVER let a storage type escape the data layer: `@Entity`, `@DatabaseView`,
  an `@Embedded`/`@Relation` holder, a `@ColumnInfo` projection class, or a
  raw `Preferences`.
- `PagingData`, and the `Flow` that carries it, never lives in the state
  class. Routed through state it is replayed to a second collector and
  throws "Attempt to collect twice from pageEventFlow."
</HARD-GATE>

## Checklist

Create a todo for each item and complete them in order:

1. Read the existing persistence code — Room 2 or 3? which driver, built
   where? where is the `DataStore` constructed? — and confirm the target
   set, web first, because a JS/Wasm target forks both the driver and
   DataStore's availability.
2. Identify which concern the change touches: schema, migration, settings,
   paging, or the repository boundary.
3. Apply the Core Rules for that concern.
4. Load the reference the rule points at.
5. Check the result against the Red Flags table.
6. Verify the build resolves for every declared target, and confirm no
   storage type appears in a signature above the data layer.

## Core Rules

### Wire Room 3 before writing a DAO
The rename is total — group `androidx.room` → `androidx.room3`, package
`androidx.room3.*`, plugin id `androidx.room3`, extension block
`room3 { schemaDirectory(...) }` — so nothing copied from an Android-era
sample or a 2.x codebase compiles. Add KSP per declared target, choose the
driver per source set (`sqlite-bundled` for android/jvm/native,
`sqlite-web` for js/wasm), and declare `@ConstructedBy(...)` on every
`commonMain` `@Database` with a matching `expect object :
RoomDatabaseConstructor<T>`.
→ Gradle wiring, coordinates, the driver matrix, and the constructor pattern: [references/room-setup.md](references/room-setup.md)

### Model the schema, then query it
Index every column named in `WHERE`, `ORDER BY`, and `JOIN ON`; project
instead of `SELECT *`; put `@Transaction` on every `@Relation` query;
`@Upsert` rather than `@Insert(onConflict = REPLACE)` where cascading
children exist; scope TypeConverters to timestamps and enums, never blobs or
nested JSON. Room dispatches its own queries — never wrap a DAO call in
`withContext(Dispatchers.IO)`; this is the one exception to the
compose-async skill's main-safe rule. Transactions are connection-level —
`useWriterConnection`/`useReaderConnection` — because `withTransaction` is
Android-only.
→ entities, DAO patterns, and connection-level transactions: [references/schema-dao.md](references/schema-dao.md), and the `@DaoReturnTypeConverters` exception a `PagingSource`-returning DAO function needs: [references/paging-core.md](references/paging-core.md)

### Evolve the schema deliberately
Every version bump needs a migration path before it ships: a manual
`Migration` against the driver's connection API, or `@AutoMigration` with
`@DeleteColumn`/`@RenameTable`/`@DeleteTable` for the cases it covers.
Decide schema export deliberately, and know when `exportSchema = false` is
Room-sanctioned rather than a shortcut.
→ manual and automatic migrations, schema export: [references/migrations.md](references/migrations.md)

### A project on Room 2.x moves by substitution, not by rewrite
Room 2.8.4 is current, supported, and already multiplatform — moving is a
coordinate and package substitution, not a redesign. Check the target
blocker first: a project needing `iosX64`, `macosX64`, `tvosX64`, or
`watchosX64` cannot move to Room 3 at all and stays on 2.8.4; a project
needing `mingwX64` has no Room at any version, at either release.
→ the substitution table and the target blocker: [references/room2-to-room3.md](references/room2-to-room3.md)

### Settings live in DataStore, one instance per file
Depend on `-preferences-core` or `-core` in `commonMain` — only the `-core`
pair publishes wasmJs. Construct the instance over okio with an `expect`/`actual`
path ending in `.preferences_pb`. Collect `dataStore.data` with `.catch` and
a corruption handler on every read — an unreadable file throws straight into
the collector otherwise. Usable DataStore on web is alpha-only, so a web
target needs a settings interface behind `expect`/`actual`.
→ Preferences and typed DataStore, paths, reading, writing, the web gap: [references/datastore.md](references/datastore.md)

### Build the paging stack in the data layer and map before it leaves
A hand-written `PagingSource` returns a fresh instance from every factory
call; a Room-backed one needs `room3-paging` plus a DAO function returning
`PagingSource` registered through `@DaoReturnTypeConverters`.
`androidx.paging:paging-runtime` (and `-guava`/`-rxjava*`) is an Android AAR
with no KMP publication — never add it to `commonMain`; `paging-common` +
`paging-compose` + `paging-testing` is the multiplatform set. Map at the
repository — `pager.flow.map { it.map(ItemEntity::toDomain) }` — before
`PagingData<ItemEntity>` ever reaches a ViewModel.
→ `PagingSource`, `Pager`, and the repository mapping: [references/paging-core.md](references/paging-core.md)

### Collect paging beside state, never through it
Expose `Flow<PagingData<Item>>` as its own property on the state holder,
`cachedIn(viewModelScope)` last, beside — never inside — `StateFlow<State>`.
Collect it with `collectAsLazyPagingItems()`, key items with `itemKey`, and
drive loading UI from `LoadState`, checking `loadState.source.refresh` when a
`RemoteMediator` is present. For filtering or search, combine the
parameters and `flatMapLatest` into a new `Pager` — never `combine(...)`
over `Flow<PagingData>` itself.
→ the two-flow state shape, collection, LoadState, filtering: [references/paging-ui.md](references/paging-ui.md)

### Offline paging writes to Room and reads from Room
`RemoteMediator` is `@ExperimentalPagingApi` — opt in on the mediator class
and the `Pager` call that wires it in. Give remote keys their own entity and
DAO, write inside a connection transaction across REFRESH/PREPEND/APPEND,
and derive `endOfPaginationReached` from the response, not from a guess.
This file covers mechanics only — why Room is the source of truth lives in
the offline-first rule below.
→ `RemoteMediator`, remote keys, and load-type handling: [references/paging-offline.md](references/paging-offline.md)

### The database is the source of truth, the network is an updater
The UI observes `dao.observeAll()` and nothing else — no response, no API
call ever reaches a screen directly. A `suspend refresh()` writes DTOs
mapped to entities into the same tables the UI is already collecting,
gated by a `lastSyncedAt` staleness check held in DataStore. This is the
boundary `paging-offline.md`'s mechanics serve, and the module shape
`compose-architecture` already documents — `core:data` depending on
`core:network`, `core:database`, `core:datastore`.
→ the repository, the DTO → Entity → Domain mapping, staleness: [references/offline-first.md](references/offline-first.md)

### Test against a real database and a real file, then wire the singletons
A fake DAO or a fake `DataStore<Preferences>` proves calling code compiles
against an interface; it proves nothing about the query, the index, or the
file format. Build a real `AppDatabase` per test — in-memory or an isolated
temp path — and a real `DataStore` over a unique temp path per test; sharing
one path across tests reproduces the duplicate-instance failure the
one-instance gate forbids. `paging-testing`'s `TestPager` and `asSnapshot`
exercise a `PagingSource` without a full `Pager`.
→ database, DataStore, and paged-flow test fixtures: [references/testing.md](references/testing.md)

## Red Flags — STOP

| Smell | Fix |
|---|---|
| `import androidx.room.*`, or a `room { }` block, beside `androidx.room3:` artifacts | Room 3 renamed all of it — package `androidx.room3.*`, plugin id `androidx.room3`, extension `room3 { schemaDirectory(...) }`. The official KMP docs page still renders `room { }` |
| `.build()` on a `RoomDatabase.Builder` with no `.setDriver(...)` | Every platform, Android and in-memory test databases included |
| `withContext(Dispatchers.IO) { dao.… }` | Delete the wrapper — Room already dispatches queries; in `commonMain` with a JS/Wasm target, naming `Dispatchers.IO` also fails to compile |
| `@Insert(onConflict = REPLACE)` on a table with `ON DELETE CASCADE` children | `@Upsert` — REPLACE deletes then re-inserts, cascading the children away silently |
| Two `RoomDatabase` or two `DataStore` instances over one file | DI singleton. DataStore throws on first read or write; Room throws nothing and stops invalidating |
| `@Database(version = )` bumped with no `Migration` or `@AutoMigration` | Every existing install throws on upgrade; destructive fallback does not fix that, it deletes the data |
| `@Relation`/`@Embedded` query without `@Transaction` | Room issues several queries and only warns; the snapshot tears |
| `@Entity`, `@DatabaseView`, an `@Embedded` holder, a `@ColumnInfo` projection, or a raw `Preferences` in a ViewModel or state signature | Map to a domain model in the repository — projections are data-layer types too |
| `db.withTransaction { }`, `System.currentTimeMillis()`, `TimeUnit`, or `HttpException` in a `commonMain` `RemoteMediator` | All JVM/Android-only — connection-level transactions, a multiplatform clock and `Duration`, and the `NetworkResult` shape from the compose-networking skill |
| A DataStore file path not ending in `.preferences_pb` | Preferences DataStore rejects any other extension |
| `InputStream`, `OutputStream`, or `java.io.File` in a `commonMain` serializer | JVM-only — typed DataStore in common code goes through okio |
| `dataStore.data` collected with no `.catch` and no corruption handler | An unreadable file throws straight into the collector |
| `androidx.datastore:datastore-preferences` or `:datastore` in `commonMain` | Depend on `-preferences-core` / `-core` instead — the plain artifacts default to Android's `File`-backed storage path and publish no wasmJs variant |
| `androidx.paging:paging-runtime` (or `-guava`/`-rxjava*`) in `commonMain` | Android AAR, no KMP publication — `paging-common` + `paging-compose` + `paging-testing` |
| `paging-compose` unresolved on the JVM target | It publishes a `desktop` variant, not `jvm` |
| `Flow<PagingData<SomeEntity>>` in a ViewModel signature | `pagingData.map { it.toDomain() }` at the repository, before `cachedIn` |
| `PagingData`, or its `Flow`, as a property of the state class | Its own property on the state holder; a replayed `PagingData` throws on the second collector |
| A pager flow with no `cachedIn(viewModelScope)`, or `.map`/`.filter` after it | `cachedIn` last — it caches only what precedes it, so a later transformation re-runs on every collection |
| A `pagingSourceFactory` returning a cached instance, or `combine(...)` over `Flow<PagingData>` | A new `PagingSource` per invocation; combine the parameters, then `flatMapLatest` |

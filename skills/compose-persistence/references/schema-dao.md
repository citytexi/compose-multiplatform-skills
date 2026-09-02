# Schema and DAOs

`AppDatabase` from [room-setup.md](room-setup.md) already declares `abstract
fun itemDao(): ItemDao`. This file defines the entity, the DAO itself, and
the mapping from a row to a domain model.

## Entities

`ItemEntity` mirrors the domain `Item` from the compose-networking skill,
field for field, with the storage annotations Room needs:

```kotlin
@Entity(tableName = "items", indices = [Index("createdAt")])
data class ItemEntity(
    @PrimaryKey val id: String,
    val name: String,
    val status: ItemStatus,
    val createdAt: Long,
)
```

`Index("createdAt")` exists because `observeAll()` sorts on it — every column
in a `WHERE`, `ORDER BY`, or `JOIN ON` needs an index, or SQLite scans the
whole table.

A composite primary key skips `@PrimaryKey` on any property and lists the
columns on `@Entity` instead — `@Entity(primaryKeys = ["userId", "itemId"])
data class UserItemCrossRef(val userId: String, val itemId: String)`; this
same shape doubles as the many-to-many junction table below.

`@ForeignKey` with `onDelete = ForeignKey.CASCADE` ties a child row's
lifetime to its parent. The FK column always gets its own `@Index` too —
SQLite never indexes it for you, so every lookup and cascading delete scans
the whole child table until it's indexed.

```kotlin
@Entity(
    tableName = "item_notes",
    foreignKeys = [
        ForeignKey(
            entity = ItemEntity::class,
            parentColumns = ["id"],
            childColumns = ["itemId"],
            onDelete = ForeignKey.CASCADE,
        ),
    ],
    indices = [Index("itemId")],
)
data class ItemNoteEntity(
    @PrimaryKey(autoGenerate = true) val noteId: Long = 0,
    val itemId: String,
    val body: String,
)
```

## Relations

`@Embedded` plus `@Relation` assembles a one-to-many holder from two queries
Room runs for you:

```kotlin
data class ItemWithNotes(
    @Embedded val item: ItemEntity,
    @Relation(parentColumn = "id", entityColumn = "itemId")
    val notes: List<ItemNoteEntity>,
)
```

Many-to-many goes through the junction table above, via `@Relation`'s
`associateBy` (`UserEntity` here is just `@Entity data class
UserEntity(@PrimaryKey val userId: String)`):

```kotlin
data class UserWithItems(
    @Embedded val user: UserEntity,
    @Relation(
        parentColumn = "userId",
        entityColumn = "id",
        associateBy = Junction(UserItemCrossRef::class, parentColumn = "userId", entityColumn = "itemId"),
    )
    val items: List<ItemEntity>,
)
```

Every DAO function returning a `@Relation` holder needs `@Transaction`: Room
issues the parent and child queries separately to build `ItemWithNotes` or
`UserWithItems`, and only *warns* when it's missing — the result still
compiles, just without a guaranteed snapshot, so a concurrent write can tear it.

```kotlin
@Transaction
@Query("SELECT * FROM items WHERE id = :id")
suspend fun findItemWithNotes(id: String): ItemWithNotes?
```

## Type Converters

A `@TypeConverter` pair turns a column into a richer Kotlin type and back —
one for a timestamp, one for an enum:

```kotlin
class Converters {
    @TypeConverter
    fun fromEpochMillis(value: Long): Instant = Instant.fromEpochMilliseconds(value)

    @TypeConverter
    fun toEpochMillis(instant: Instant): Long = instant.toEpochMilliseconds()

    @TypeConverter
    fun fromStatusName(value: String): ItemStatus = ItemStatus.valueOf(value)

    @TypeConverter
    fun toStatusName(status: ItemStatus): String = status.name
}
```

`ItemEntity.createdAt` stays a raw `Long` above — no converter needed; the
timestamp pair is for an entity wanting a richer type instead. The status
pair is what converts `ItemEntity.status: ItemStatus`, declared three lines
above it. Register `@TypeConverters(Converters::class)` once, on
`AppDatabase` itself.

The rule stops there: a converter is for a scalar round-tripping through one
column, never a blob or a nested object graph. Binary data goes on disk with
the path stored in a column; a value with its own identity goes in its own
normalized table, related the way `ItemNoteEntity` is — a JSON-blob converter
defeats every query and index Room offers.

### Full-Text Search

`@Fts3`, `@Fts4`, and `@Fts5` (package `androidx.room3`) are all present at
the `3.0.2` pin — confirmed by unpacking `room3-common-jvm-3.0.2.jar`, which
ships `Fts3.class`, `Fts4.class`, `Fts5.class`, and `FtsOptions`. Room 2
already had all three, Android-only; what Room 3 adds is availability from
`commonMain`, across every target `room3-common` publishes. Reach for
`@Fts5` by default — it's the actively developed SQLite FTS module — and
keep `@Fts4` only for a schema inherited from before FTS5.

## DAO Functions

The canonical DAO, everything after this file reuses unchanged:

```kotlin
@Dao
@DaoReturnTypeConverters(PagingSourceDaoReturnTypeConverter::class)
interface ItemDao {
    @Query("SELECT * FROM items ORDER BY createdAt DESC")
    fun observeAll(): Flow<List<ItemEntity>>

    @Query("SELECT * FROM items WHERE id = :id")
    suspend fun findById(id: String): ItemEntity?

    @Upsert
    suspend fun upsertAll(items: List<ItemEntity>)

    @Delete
    suspend fun delete(item: ItemEntity)

    @Query("DELETE FROM items")
    suspend fun deleteAll()

    @Transaction
    suspend fun replaceAll(items: List<ItemEntity>) {
        deleteAll()
        upsertAll(items)
    }
}
```

The class-level `@DaoReturnTypeConverters(PagingSourceDaoReturnTypeConverter::class)`
is carried from the start even though nothing here returns a `PagingSource` —
a later reference adds `pagingSource()` to this interface, and a rival
`ItemDao` there would ship two incompatible declarations. `:id` is a bind
parameter, resolved by Room's statement cache; never concatenate a query
string, which opens the door to injection.

Every DAO function compiled for a non-Android target must be `suspend`,
return `Flow`, or return a type registered with `@DaoReturnTypeConverters` —
including `@Transaction`, why `replaceAll` is `suspend`, not a plain default
method. The compiler says: *"Only suspend functions are allowed in DAOs
declared in source sets targeting non-Android platforms."* `androidMain` is
exempt; `commonMain` is not, even in an Android-only project today — add a
second target and every blocking function there stops compiling, a
compile-time error, never the "crashes non-Android" the upstream source claims.

`@Upsert` over `@Insert(onConflict = OnConflictStrategy.REPLACE)`: `REPLACE`
deletes the row and inserts a new one, so any row related to it via
`onDelete = CASCADE` — `ItemNoteEntity` above — goes with it. `@Upsert`
updates in place: silent data loss traded for one word.

## Transactions

The connection-level transaction API is what `replaceAll` above compiles down
to: a writer connection with an immediate transaction for writes, a reader
connection with a deferred transaction for a consistent multi-query read. Per
`developer.android.com/kotlin/multiplatform/room`, `useWriterConnection` takes
`Transactor` as a lambda **parameter**, not a receiver — its examples always
write `transactor.immediateTransaction { }`, never a bare one:

```kotlin
database.useWriterConnection { transactor ->
    transactor.immediateTransaction {
        // writes here see each other; readers see either all of them or none
    }
}
```

`immediateTransaction` acquires its lock up front and is the right default
for writes. `useReaderConnection { transactor -> transactor.deferredTransaction { } }`
is the read-side equivalent — a reader connection only permits the deferred
kind; Room throws if asked to start an immediate or exclusive transaction,
since those are writes. `withTransaction { }` is the Android-only shorthand
for `useWriterConnection { it.immediateTransaction { } }` and cannot be
called from `commonMain` — the defect the upstream source's `RemoteMediator`
sample carries by using it there.

## Query Performance

`SELECT *` is fine when the entity is narrow — `ItemEntity` is four columns,
so a projection would be ceremony for no savings. On a wide table, project
only the columns a query needs into a `@ColumnInfo`-annotated class instead:

```kotlin
data class ItemSummary(
    @ColumnInfo(name = "id") val id: String,
    @ColumnInfo(name = "name") val name: String,
)

@Query("SELECT id, name FROM items ORDER BY createdAt DESC")
fun observeSummaries(): Flow<List<ItemSummary>>
```

A projection class is a data-layer type like any entity, never crossing into
the domain layer. Beyond projections: index every filtered/sorted column
(`Index("createdAt")` above is exactly this), and let `@Relation` assemble a
parent-with-children result instead of looping a query per parent.

> Do not wrap a DAO call in `withContext(ioDispatcher)`. The compose-async
> skill's main-safe rule holds for hand-written suspend functions, but a Room
> DAO already runs on the database's own query coroutine context — the
> wrapper is a redundant, web-target-breaking hop.

## Mapping Out

```kotlin
fun ItemEntity.toDomain(): Item = Item(id = id, name = name, status = status, createdAt = createdAt)

// The write path maps the network DTO straight to a row. There is deliberately no
// Item.toEntity(): the compose-networking skill's refresh() maps ItemDto, and a domain
// model never travels back down into storage in this design.
fun ItemDto.toEntity(): ItemEntity = ItemEntity(id = id, name = name, status = ItemStatus.valueOf(status.name), createdAt = createdAt)
```

`ItemDto` belongs to the compose-networking skill — referenced by name here,
never redeclared.

> `@Entity` types stop at the DAO's caller — the repository maps them before
> anything above the data layer sees a row. Why domain models must differ
> from entities is covered by the compose-architecture skill.

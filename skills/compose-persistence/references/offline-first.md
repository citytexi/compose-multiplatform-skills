# Offline-First Repositories

An offline-first repository treats the local database as the only thing the
UI ever reads. The network exists to keep that database current, not to hand
data to a screen directly. `schema-dao.md` and `datastore.md` each define one
half of the machinery this pattern assembles; this file is where they meet.

## Room Is The Source Of Truth

The UI observes `dao.observeAll()` — a local `Flow` — and nothing else.
Network calls run underneath it, writing rows into the same table the UI is
already collecting; the UI never subscribes to a response, an API call, or
anything else that lives outside Room. That single decision buys three
things at once:

- **The list survives a cold start with no network.** Room hands back
  whatever rows were written on the last successful sync, before any request
  goes out.
- **A failed refresh leaves the last-known-good rows in place.** Nothing
  deletes local data just because a request failed; the `Flow` keeps
  emitting exactly what it emitted before the attempt.
- **Exactly one place decides what the UI sees.** A screen never chooses
  between "the cached list" and "the response that just came back" — there
  is only ever the table, so that choice can't be made inconsistently across
  screens.

The trade is that a screen can be looking at data that is stale by however
long it has been since the last successful refresh. Whether that data is
stale enough to warrant a new refresh is a separate question, answered below
in Staleness — it is deliberately not answered by the `Flow` itself.

## Three Layers, Two Mappers

Three types describe the same "item," none of them interchangeable:

| Layer | Type | Lives in |
|---|---|---|
| Wire format | `ItemDto` | compose-networking skill |
| Storage row | `ItemEntity` | `schema-dao.md` |
| Domain model | `Item` | compose-networking skill |

Two mapping functions connect them, and each sits next to the code that
needs it rather than off in a shared "mappers" file:

- `ItemDto.toEntity()` is called where a network response lands — inside
  `refresh()`, right after `api.getItems()` returns. It belongs beside the
  network call because that is the only place an `ItemDto` exists.
- `ItemEntity.toDomain()` is called where a database row leaves the DAO —
  inside the `items` property below, mapping over what `observeAll()`
  emits. It belongs beside the DAO because that is the only place an
  `ItemEntity` exists.

Both extension functions are defined once, in
[schema-dao.md](schema-dao.md#mapping-out); nothing here redeclares them.
The consequence of routing every value through these two mappers is that
neither `ItemDto` nor `ItemEntity` is ever the type a use case or a
ViewModel names — only `Item` crosses that boundary, in either direction.

## Refreshing

```kotlin
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map

class OfflineFirstItemRepository(
    private val api: ItemApi,
    private val dao: ItemDao,
) {
    val items: Flow<List<Item>> = dao.observeAll().map { entities -> entities.map { it.toDomain() } }

    suspend fun refresh(): NetworkResult<Unit> = safeCall {
        val remote = api.getItems().items
        dao.replaceAll(remote.map { it.toEntity() })
    }
}
```

This is the same class the compose-networking skill ships from the other
side. `Item` here is that skill's `Item(id, name, status, createdAt)`, and
`it.toEntity()` inside `refresh()` has `ItemDto` as its receiver —
`api.getItems().items` is a `List<ItemDto>`, not a `List<Item>`. There is no
`Item.toEntity()` anywhere in this skill; a domain model never travels back
down into storage.

> The network half of this repository — the API service, `safeCall`, and the
> `NetworkResult<Unit>` that `refresh()` returns — is covered by the
> compose-networking skill, which shows this same repository from the
> network side. This file owns the other half: the entity, the DAO behind
> `observeAll()`, the transactional `replaceAll`, and the timestamp that
> decides whether `refresh()` is called at all.

`replaceAll` is transactional for a reason. A plain `deleteAll()` followed by
a separate `upsertAll()` call, run outside a transaction, leaves a window
between the two statements where `observeAll()` can emit an empty list —
every row deleted, none yet reinserted. `schema-dao.md` defines `replaceAll`
as a [`@Transaction suspend fun`](schema-dao.md#dao-functions) precisely to
close that window; nothing about the transaction itself is redefined here.

## Staleness

`refresh()` is cheap to call but not free, and calling it on every
recomposition would turn "offline-first" into "always refetching." The
decision of whether to call it at all is a timestamp check, not a network
check: read `lastSyncedAt` before refreshing, write it only after a refresh
succeeds.

`datastore.md` already defines the accessor pair this needs —
[`SettingsRepository`](datastore.md#reading-and-writing) exposes
`lastSyncedAt: Flow<Long?>` and `suspend fun setLastSyncedAt(epochMillis:
Long)`. Nothing here adds a second `lastSyncedAt` key or a competing
accessor; a caller composes the two repositories instead:

```kotlin
import kotlinx.coroutines.flow.first

class ItemSyncCoordinator(
    private val items: OfflineFirstItemRepository,
    private val settings: SettingsRepository,
    private val maxAgeMillis: Long,
) {
    suspend fun refreshIfStale(nowMillis: Long) {
        val lastSyncedAt = settings.lastSyncedAt.first()
        val isStale = lastSyncedAt == null || nowMillis - lastSyncedAt > maxAgeMillis
        if (!isStale) return
        if (items.refresh() is NetworkResult.Success) {
            settings.setLastSyncedAt(nowMillis)
        }
    }
}
```

The stamp lives in DataStore rather than in the Room database on purpose. A
destructive schema migration (`fallbackToDestructiveMigration`, covered in
[migrations.md](migrations.md)) wipes every table, `lastSyncedAt` included if
it were stored there — the next launch would see no timestamp, correctly
treat the cache as stale, and refresh. But a **non-destructive** migration
that merely adds a column would leave an old `lastSyncedAt` row sitting in
the database untouched, telling the app a sync happened more recently than
it actually did. Keeping the stamp in a separate DataStore file removes that
coupling entirely: a schema change to the item table has no way to touch it,
in either direction.

## Module Shape

This repository is where a data module's three dependencies actually get
used, not just declared: `OfflineFirstItemRepository` needs `ItemApi` from
the network module, `ItemDao` from the database module, and — once staleness
is wired in — `SettingsRepository` from the datastore module. That is the
same `core:data -> core:network, core:database, core:datastore` shape the
compose-architecture skill already documents for every data-layer module,
not a rival layout invented here; this file only points out that offline-first
sync is the concrete reason a data module ends up depending on all three at
once, and defers everything else about module boundaries to that skill.

> Collecting the DAO `Flow` into screen state is covered by the
> compose-architecture skill; holding it hot with `stateIn` is covered by
> the compose-async skill. Neither concern belongs in this file.

# Paging in the UI

This file covers `Flow<PagingData<Item>>` once it leaves
`ItemPagingRepository` in [paging-core.md](paging-core.md): where it lives on
the state holder, how a composable collects it, how `LoadState` drives
loading UI, and how filtering, search, and separators fit around it without
breaking `cachedIn`.

## Two Flows, Not One

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import androidx.paging.PagingData
import androidx.paging.cachedIn
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow

data class ItemListState(
    val query: String = "",
    val filter: ItemFilter = ItemFilter.ALL,
)

class ItemListViewModel(private val repository: ItemPagingRepository) : ViewModel() {
    private val _state = MutableStateFlow(ItemListState())
    val state: StateFlow<ItemListState> = _state.asStateFlow()

    val items: Flow<PagingData<Item>> =
        repository.pagedItems().cachedIn(viewModelScope)
}
```

`PagingData`, and the `Flow` that carries it, never live inside the state
class. Three independent reasons, each catching a different reader:

- `PagingData` has no meaningful `equals`, so a `MutableStateFlow<ItemListState>`
  holding one would never conflate two emissions and would re-emit — and
  recompose — on every unrelated state change.
- A `PagingData` reached through a replaying `StateFlow` gets handed to a
  second collector, throwing *"Attempt to collect twice from
  pageEventFlow, which is an illegal operation"* — its event flow is
  single-collector by design.
- A state class holding nothing paging-shaped today gains a second field
  tomorrow — a loading flag, a selection set — and reintroduces the scroll
  reset the first bullet describes, silently, the moment it does.

A `Flow` as a property of a `data class` also breaks the state-stability
rule the compose-architecture skill sets for anything read with `by …
.collectAsState()`: a stable type must be structurally comparable, and a
cold or hot stream is neither. Keeping `items` beside `state`, not inside
it, is what makes both problems disappear at once.

> The `Event`/`State`/`Effect` contract this second flow sits beside is
> covered by the compose-architecture skill; this file only covers what
> paging adds on top of it.

## Collecting In Compose

```kotlin
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.paging.LoadState
import androidx.paging.compose.collectAsLazyPagingItems
import androidx.paging.compose.itemContentType
import androidx.paging.compose.itemKey

@Composable
fun ItemListRoute(viewModel: ItemListViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val lazyItems = viewModel.items.collectAsLazyPagingItems()
    // state.query / state.filter drive the search bar and filter chips above the list.

    val refresh = lazyItems.loadState.refresh
    when {
        refresh is LoadState.Loading && lazyItems.itemCount == 0 -> FullScreenSpinner()
        refresh is LoadState.Error -> ErrorBanner(refresh.error, onRetry = lazyItems::retry)
        else -> LazyColumn {
            items(
                count = lazyItems.itemCount,
                key = lazyItems.itemKey { it.id },
                contentType = lazyItems.itemContentType { it.status },
            ) { index ->
                val item = lazyItems[index]
                if (item != null) ItemRow(item)
            }
            if (lazyItems.loadState.append is LoadState.Loading) {
                item { BottomSpinner() }
            }
        }
    }
}
```

`refresh is LoadState.Loading` is gated on `lazyItems.itemCount == 0`: a
`LoadState.Loading` refresh also fires on a user-triggered retry with rows
already on screen, and swapping in the full-screen spinner there would
discard the very scroll position this file opened by arguing to protect.

`collectAsLazyPagingItems()` is an `@Composable` extension on
`Flow<PagingData<T>>` in `androidx.paging.compose`; it returns a
`LazyPagingItems<T>` exposing `itemCount`, `get(index)`, `loadState`, and
`retry()`. `itemKey` and `itemContentType` are extensions on
`LazyPagingItems<T>` in that same package — each wraps the lambda you give
it into an `(index: Int) -> Any` (or `Any?`) function, exactly the shape
`items(...)`'s `key` and `contentType` parameters take. The helper is
**`itemContentType`**; `contentType` is only the name of the
`LazyListScope.items(...)` parameter that receives its result — mixing the
two names up is the mistake this file exists to preempt.

The key matters beyond style: without it, every append re-keys every row
already on screen by its position, and Compose loses each item's identity
— visible as a scroll jump and lost per-row animation state on every page
load, not just a missed optimization.

## LoadState

`lazyItems.loadState` is a `CombinedLoadStates`, one `LoadState` per load
type:

| Load type | Reports |
|---|---|
| `refresh` | initial load, an explicit refresh, or a retry after `refresh` failed |
| `prepend` | loading data above the currently visible range |
| `append` | loading more data at the end of the list — the "load more" spinner |

Each is a `LoadState.Loading`, `LoadState.NotLoading`, or `LoadState.Error(error:
Throwable)`, from `androidx.paging`.

`CombinedLoadStates` also exposes `source: LoadStates` and `mediator:
LoadStates?`, each with its own `refresh`/`prepend`/`append`. With no
`RemoteMediator` attached — every case this skill covers so far — `source`
is the only signal and the convenience properties above track it directly.
Once a `RemoteMediator` is introduced, `refresh`/`prepend`/`append` only
settle to `NotLoading` after *both* `mediator` and `source` report
`NotLoading`, so a spinner bound to the top-level `loadState.refresh` can
keep spinning after Room already has fresh rows, waiting on the network
leg — while a spinner bound to `loadState.source.refresh` instead flips
off as soon as the local query completes, possibly before the mediator has
written anything. Neither is unconditionally correct; bind deliberately
once a mediator lands, not by habit.

## Filter and Search

Combine the *parameters* that choose a page source, never the
`PagingData` streams themselves:

```kotlin
import androidx.lifecycle.viewModelScope
import androidx.paging.PagingData
import androidx.paging.cachedIn
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.FlowPreview
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.flow.debounce
import kotlinx.coroutines.flow.distinctUntilChanged
import kotlinx.coroutines.flow.flatMapLatest
import kotlinx.coroutines.flow.map

// inside ItemListViewModel, replacing the single-source `items` above
@OptIn(FlowPreview::class, ExperimentalCoroutinesApi::class)
val items: Flow<PagingData<Item>> =
    combine(
        state.map { it.query }.distinctUntilChanged().debounce(300),
        state.map { it.filter }.distinctUntilChanged(),
        ::Pair,
    )
        .flatMapLatest { (query, filter) -> repository.pagedItems(query, filter) }
        .cachedIn(viewModelScope)
```

The `@OptIn` is not decoration: `debounce` is still `@FlowPreview` and
`flatMapLatest` is still `@ExperimentalCoroutinesApi`, so a project
building with `-Werror` fails on the bare warnings without it. `::Pair` as
the transform argument to `combine` relies on Kotlin's suspend conversion
for callable references — `combine`'s transform parameter is `suspend`,
`::Pair`'s constructor reference is not, and the compiler converts between
them automatically, stable since Kotlin 1.6.

This ordering prevents two failures: `combine` over two
`Flow<PagingData<Item>>` values throws at runtime — those streams are not
meant to be merged, only produced fresh per query — so what gets combined
is the plain `query` and `filter` that choose *which* `Pager` to build,
decided once per distinct pair by `flatMapLatest`. And building a `Pager`
inside a composable body, keyed on nothing, rebuilds it on every
recomposition, discarding every loaded page; `flatMapLatest` in the
`ViewModel` keeps one `Pager` alive per distinct `(query, filter)` pair
instead.

> The operator semantics behind `debounce`, `distinctUntilChanged`, and
> `flatMapLatest` are covered by the compose-async skill; this file only
> states the paging-specific constraint on top of them — that `cachedIn`
> comes last in the chain.

## Transformations

`insertSeparators` turns a `PagingData<T>` into a `PagingData<R>` given a
common UI supertype `R`, so a single list can mix rows and separators
without either side casting:

```kotlin
import androidx.lifecycle.viewModelScope
import androidx.paging.PagingData
import androidx.paging.cachedIn
import androidx.paging.insertSeparators
import androidx.paging.map
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map

sealed interface ItemListUiModel {
    data class Row(val item: Item) : ItemListUiModel
    data class DateHeader(val label: String) : ItemListUiModel
}

// inside ItemListViewModel
val items: Flow<PagingData<ItemListUiModel>> =
    repository.pagedItems()
        .map { pagingData ->
            pagingData
                .map(ItemListUiModel::Row)
                .insertSeparators { before, after ->
                    val afterLabel = after?.let { dayLabel(it.item.createdAt) }
                    val beforeLabel = before?.let { dayLabel(it.item.createdAt) }
                    if (afterLabel != null && afterLabel != beforeLabel) {
                        ItemListUiModel.DateHeader(afterLabel)
                    } else {
                        null
                    }
                }
        }
        .cachedIn(viewModelScope)
```

`insertSeparators`'s full signature is `PagingData<T>.insertSeparators(
terminalSeparatorType: TerminalSeparatorType = FULLY_COMPLETE, generator:
suspend (T?, T?) -> R?): PagingData<R>`, declared in `androidx.paging`
alongside `map` and `filter`; `dayLabel(epochMillis: Long): String` above
is an ordinary formatting helper, not part of the Paging API. `before` and
`after` are `null` at the two ends of the list; returning `null` from
`generator` skips the separator for that pair. `filter` lives in the same
file, same `suspend (T) -> Boolean` shape, for dropping items client-side
without a second round trip through the repository.

Every one of these — `map`, `filter`, `insertSeparators` — must run
*before* `cachedIn`, restated here because this is where people break it:
`cachedIn` caches only the stream as it exists at the point it is called.
A transformation applied after it is not lost — nothing disappears — it
simply is not part of what got cached, so it re-executes on every new
collection of that cached stream: wasted work, not a correctness bug, and
still a reason to keep every transformation on the repository-facing side
of `cachedIn`, same as [paging-core.md](paging-core.md) states for the
entity-to-domain mapping.

# Flow Operators

Prefer declarative operators over manual collection and mutable accumulation.

## Transforming

| Operator | Purpose |
|---|---|
| `map { }` | Transform each value |
| `mapNotNull { }` | Transform and drop nulls |
| `filter { }` | Keep values matching predicate |
| `take(n)` / `drop(n)` | Take first n / skip first n |

## Flattening

| Operator | Behavior | Use when |
|---|---|---|
| `flatMapLatest { }` | Cancel previous inner flow | Search queries — only latest |
| `flatMapConcat { }` | Sequential, wait for completion | Order matters |
| `flatMapMerge { }` | Concurrent inner flows | Parallel, order irrelevant |

## Combining

| Operator | Behavior | Use when |
|---|---|---|
| `combine(flowA, flowB) { a, b -> }` | Emit when ANY emits, latest from each | Multiple independent state sources |
| `zip(flowA, flowB) { a, b -> }` | Paired emissions only | Synchronized pairs |
| `merge(flowA, flowB)` | Interleave emissions | Unified event stream |

**Gotcha:** `combine` waits until every upstream emits at least once before producing output. Fix by seeding the missing upstream with `onStart { emit(default) }`.

## Timing, Error, Side Effects

| Operator | Purpose |
|---|---|
| `debounce(300)` | Wait for pause (search input) |
| `sample(1000)` | Latest at fixed intervals |
| `distinctUntilChanged()` | Skip consecutive duplicates |
| `catch { }` | Handle upstream errors, can `emit()` fallback |
| `retry(3)` / `retryWhen { cause, attempt -> }` | Retry with optional backoff |
| `onEach { }` / `onStart { }` / `onCompletion { }` | Side effects |

## Terminal Operators

| Operator | Purpose |
|---|---|
| `collect { }` / `collectLatest { }` | Collect values (suspends) |
| `first()` / `toList()` | Single value / all values |
| `launchIn(scope)` | Start collection in scope |
| `stateIn(scope)` / `shareIn(scope)` | Convert to hot StateFlow/SharedFlow — see [statein-sharein.md](statein-sharein.md) |

Backpressure operators (`buffer`, `conflate`, `collectLatest`) are detailed in [advanced.md](advanced.md).

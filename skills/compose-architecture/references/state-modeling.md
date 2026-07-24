# State Modeling for Forms and Calculators

How to shape screen state for forms and calculators. For the MVI State type in general, see [mvi.md](mvi.md).

Split screen state into four buckets:

1. **Editable input** — raw text/choice values as the user edits
2. **Derived/computed** — parsed, validated, calculated values
3. **Persisted snapshot** — existing saved entity for dirty tracking
4. **Transient UI-only** — only when purely visual and not business-significant

| Concern | Where | Example |
|---|---|---|
| Raw field text | `state` | `"12"`, `"12."`, `""` |
| Parsed value | computed property or `state` | `val amount get() = amountText.toDoubleOrNull()` |
| Validation | `state.errors` | `mapOf("area" to "Required")` |
| Calculated totals | `state` or computed | subtotal, tax |
| Loading/refresh | `state` flags | `isSaving`, `isLoading` |
| One-off commands | `Effect` via Channel | snackbar, navigate |
| Scroll/focus/animation | local Compose state | `LazyListState`, expansion toggle |

Use computed properties for trivial derivations:

```kotlin
data class CreateItemState(
    val title: String = "",
    val amount: String = "",
    val isSaving: Boolean = false,
    val errors: Map<String, String> = emptyMap()
) {
    val canSave: Boolean get() = title.isNotBlank() && amount.isNotBlank()
    val hasErrors: Boolean get() = errors.isNotEmpty()
}
```

**Avoid duplicated state:** don't store `total` + `formattedTotal` + `totalText`, or `showErrorDialog` + `pendingError` when one implies the other. Keep the canonical value and derive the rest at the UI boundary.

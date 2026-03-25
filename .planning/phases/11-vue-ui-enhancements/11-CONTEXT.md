# Phase 11: Vue UI Enhancements - Context

**Gathered:** 2026-03-25
**Status:** Ready for planning

<domain>
## Phase Boundary

Six targeted changes to `S7PlusAlarmsViewerPage.vue` — one file, pure frontend. No backend changes (listS7PlusAlarms already returns all fields including `priority` and `isAcknowledgeable`).

1. Replace separate Date + Time columns with a single Timestamp column (`2026-03-24_12:57:10.758` format, local time)
2. Add sortable Priority column (user-initiated sort, no default)
3. Extend Acknowledge column to reflect `isAcknowledgeable` — dash for false, existing Ack/spinner/checkmark for true
4. Add Source PLC filter dropdown (connectionName, same pattern as Status/AlarmClass)
5. Add "Ack All" button in the filter row next to Delete Filtered
6. Preserve current page across 5-second auto-refresh

</domain>

<decisions>
## Implementation Decisions

### Ack Indicator (VIEWER-03)
- **D-01:** No new column. The existing Acknowledge column is extended:
  - `isAcknowledgeable === false` → display a dash (`-`)
  - `isAcknowledgeable === true` → existing behavior unchanged (Ack button / spinner / mdi-check)
- **D-02:** The Ack button visibility condition remains driven by `ackState` (not `isAcknowledgeable`), consistent with REQUIREMENTS.md note. The dash is the indicator for info-only alarms.

### Ack All Button (VIEWER-05)
- **D-03:** "Ack All" button lives in the filter row (`v-row`), next to the "Delete Filtered" button. Consistent bulk-action placement.
- **D-04:** Ack All targets alarms matching current filter where `!alarm.ackState && alarm.isAcknowledgeable === true`. The confirmation dialog shows the count of those alarms before proceeding.
- **D-05:** Each ack is attempted independently via the existing `ackAlarm()` function — a single failure does not block the rest (matches REQUIREMENTS.md VIEWER-05).
- **D-06:** Button is disabled when the count of unacked+acknowledgeable alarms matching the active filter is 0.

### Priority Column (VIEWER-02)
- **D-07:** Priority column is unsorted on load. User clicks the column header to sort (Vuetify default click-to-sort behavior). No `:sort-by` prop.
- **D-08:** `priority` is already in every MongoDB document and returned by `listS7PlusAlarms` — no API changes needed.

### Column Order
- **D-09:** New column order: `Source | Timestamp | Priority | Status | Acknowledge | Delete | Alarm class name | Event text | ID | Origin DB Name | DB Number | Additional text 1 | Additional text 2 | Additional text 3`
  - Source and Timestamp first (identity + when), then Priority (criticality), then existing state/action columns, then detail columns.

### Timestamp Column (VIEWER-01)
- **D-10:** `date` and `time` columns replaced by single `timestamp` column. Format: `2026-03-24_12:57:10.758` — ISO date with underscore separator, time with milliseconds. Use local time (same as existing `toLocaleDateString()`/`toLocaleTimeString()` intent). Implement via a new `formatTimestamp()` function.

### Source Filter (VIEWER-04)
- **D-11:** Follow the same pattern as `alarmClassOptions` — a computed property that extracts distinct `connectionName` values from `alarms.value`, sorted, with `'All'` prepended. A `connectionFilter` ref defaults to `'All'`. Added to `filteredAlarms` computed alongside existing filters.

### Page Preservation (VIEWER-06)
- **D-12:** Use `v-model:page` on `v-data-table` bound to a `currentPage` ref (initialized to `1`). `fetchAlarms` updates `alarms.value` without touching `currentPage`, so the table stays on the current page across refreshes.

### Claude's Discretion
- Exact label for the Ack All confirmation dialog (e.g., "Acknowledge X unacked alarms matching current filter?") — use clear operator language consistent with existing Delete dialogs.
- `formatTimestamp()` implementation detail — manual string construction is clearest: `YYYY-MM-DD_HH:MM:SS.mmm` from `Date` object properties.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Implementation File
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — The only file changed in this phase. Contains all existing filters, headers, `ackAlarm()`, `fetchAlarms()`, `v-data-table`, and dialog logic.

### Requirements
- `.planning/REQUIREMENTS.md` §VIEWER-01 through §VIEWER-06 — Acceptance criteria for all six changes.

### Notes
- REQUIREMENTS.md §VIEWER-03 note: "display hint only — Ack button visibility is driven by `ackState`, not this field" — the dash is purely visual, does not gate the Ack button.
- REQUIREMENTS.md Out-of-scope: "Hiding Ack button for non-acknowledgeable alarms" — confirmed out of scope; D-01 complies.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ackAlarm(cpuAlarmId, connectionNumber)` (line 156): Reused as-is for the Ack All loop. Iterates over filtered unacked+acknowledgeable alarms.
- `alarmClassOptions` computed (line 206): Pattern to copy for `connectionOptions` (source filter).
- `filteredAlarms` computed (line 213): Add `connectionFilter` condition alongside existing `stateMatch` and `classMatch`.
- `confirmState` ref + dialog (line 149, 104–136): Extend to handle a new `'ack-all'` type for the Ack All confirmation dialog.

### Established Patterns
- Filters: `v-select` with `:items` from computed array, `v-model` bound to a `ref('All')`. All filters applied in `filteredAlarms` computed.
- Bulk action buttons: `density="compact"`, `:disabled` when count is 0, `@click` triggers confirmation dialog.
- Column slots: `template #[item.{key}]` for custom cell rendering.
- `v-data-table` `:items-per-page="50"` with options `[25, 50, 100, 200]` — unchanged.

### Integration Points
- `v-data-table`: Add `v-model:page="currentPage"` for VIEWER-06.
- `headers` array: Replace `date` + `time` entries with `timestamp`; add `priority`; reorder per D-09.
- `fetchAlarms`: No changes needed — already updates `alarms.value = json`; page preserved via `currentPage` ref.

</code_context>

<specifics>
## Specific Ideas

- User explicitly chose to extend the Acknowledge column (not add a new column) for the ack indicator — dash for non-acknowledgeable, existing logic for acknowledgeable.
- Ack All count is filtered to `!ackState && isAcknowledgeable` within active filter — not all filtered alarms.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 11-vue-ui-enhancements*
*Context gathered: 2026-03-25*

# Phase 8: Frontend — Delete Buttons + Origin Columns - Context

**Gathered:** 2026-03-24
**Status:** Ready for planning

<domain>
## Phase Boundary

Frontend-only changes to `S7PlusAlarmsViewerPage.vue`. Deliver:
1. Two new columns in the alarm table: "Origin DB Name" and "DB Number" (ORIGIN-06, ORIGIN-07)
2. A per-row Delete icon button in a new column immediately right of the Ack column (DELETE-04, DELETE-06)
3. A "Delete Filtered (N)" bulk delete button in the filter toolbar (DELETE-05)

No backend changes. No new API endpoints. Consumes `_id` from the list response and calls `POST /Invoke/auth/deleteS7PlusAlarms` (both delivered by Phase 7).

</domain>

<decisions>
## Implementation Decisions

### Origin Columns
- **D-01:** Add two new columns: "Origin DB Name" (key: `originDbName`) and "DB Number" (key: `dbNumber`). Insert both immediately **after** the existing "ID" column (`cpuAlarmId`). Final column order: … | Event text | ID | Origin DB Name | DB Number | Additional text 1-3.
- **D-02:** **RelationId column is intentionally omitted.** Despite ORIGIN-08 specifying a RelationId column, the user decided it adds no operator value (raw 64-bit integer). The requirement is dropped in this phase. If needed later, it can be a separate phase. Note this deviation in the plan.

### Delete Column
- **D-03:** Add a dedicated "Delete" column as its own column, positioned **immediately right of the Acknowledge column**. Final sequence: … | Status | Acknowledge | Delete | Alarm class name | … The column header title is "Delete". Use a compact trash icon button (`mdi-delete` icon) matching the existing Vuetify icon-button pattern in the table.
- **D-04:** The delete column uses `app.post` on `[authJwt.isAdmin]` — same cookie auth as the list endpoint. No explicit Authorization header needed in the `fetch()` call (consistent with `fetchAlarms` which calls without headers).

### Delete Filtered Button
- **D-05:** Add the "Delete Filtered (N)" button as a **third `v-col`** in the existing filter `v-row`, alongside the Status and Alarm Class dropdowns. The button is **disabled when `filteredAlarms.length === 0`**. The button label includes the live count: "Delete Filtered ({{ filteredAlarms.length }})".

### Confirmation & Warning Dialogs
- **D-06:** Per-row Delete on a **normal row** (any state except Coming+unacked) → **no confirmation dialog**. Delete fires immediately; the row is removed optimistically from `alarms.value`.
- **D-07:** Per-row Delete on a **Coming+unacked row** (`alarmState === 'Coming' && ackState === false`) → show a **warning dialog** before deleting. Dialog content:
  - Title: "This alarm is still active on the PLC"
  - Body: "Deleting it will only remove the history record — the alarm remains active on the PLC."
  - Buttons: "Delete anyway" (red / danger color) + "Cancel"
  - On "Delete anyway": proceed with delete and optimistic removal.
- **D-08:** "Delete Filtered (N)" bulk delete → **always show a confirmation dialog** before proceeding.
  - Title: "Delete Filtered Alarms"
  - Body: "Delete [N] alarm record(s)? This cannot be undone."
  - Buttons: "Delete" (red) + "Cancel"
  - On confirm: send `{ filter: { alarmState, alarmClassName } }` if filters are active, OR iterate and call with ids if filter is "All" for both (planner to determine the right body shape given the filtered set).

### Error Handling
- **D-09:** On a failed delete API call (network error or non-2xx response), **do nothing**. The 5-second auto-refresh will restore the row. No rollback, no snackbar, no error message. Keeps the component simple.
- **D-10:** On a successful delete (`response.ok`), remove the row immediately from `alarms.value` (optimistic — the doc is already deleted server-side).

### Claude's Discretion
- Dialog implementation: whether to use Vuetify `v-dialog` with a `v-card` inline in the template, or a separate reactive `confirmState` object — planner decides whichever is cleanest.
- Bulk delete body shape: if both filters are "All", `filteredAlarms` === all alarms. Planner decides whether to send all `_id`s or some other approach. Sending `{ ids: [...] }` with all visible IDs is safe and explicit.
- Column widths/alignment: not specified; planner uses sensible Vuetify defaults.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Frontend — File to modify
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — The only file changed in this phase. Read the full file before planning.

### Backend — APIs consumed
- `json-scada/src/server_realtime_auth/index.js` lines 349–369 (listS7PlusAlarms) — now returns `_id`, `originDbName`, `dbNumber` per document
- `json-scada/src/server_realtime_auth/index.js` lines ~370–410 (deleteS7PlusAlarms) — POST endpoint, accepts `{ ids: [...] }` or `{ filter: { alarmState, alarmClassName } }`, returns 204 on success

### Requirements
- `.planning/REQUIREMENTS.md` — ORIGIN-06, ORIGIN-07, DELETE-04, DELETE-05, DELETE-06: exact column names, button behavior, warning text
- ORIGIN-08 (RelationId column) is **intentionally skipped** per D-02 — do not implement it

### Prior Phase Context
- `.planning/phases/07-backend-delete-endpoint-id-exposure/07-CONTEXT.md` — D-01 (204 response), D-05 (_id serialization as hex string)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `v-data-table` with `headers` array and `#[item.{key}]` slot pattern — all new columns follow this same pattern
- `filteredAlarms` computed (lines 105–116) — already applies statusFilter + alarmClassFilter; "Delete Filtered" uses this exact array
- `alarms.value = json` full replacement on `fetchAlarms` — failed deletes are self-healing via the 5s timer
- `fetch('/Invoke/auth/listS7PlusAlarms')` without explicit auth headers — same pattern for delete calls

### Established Patterns
- Icon-based column cells: `v-icon` with conditional color (see ackState column) — use same pattern for delete button
- Filter UI: `v-row` > `v-col` > `v-select` with `density="compact"` — add Delete Filtered button as a third `v-col` in this row
- `ref()` + `computed()` reactivity — add a `pendingDelete` ref for the confirmation state

### Integration Points
- `_id` field: now returned by listS7PlusAlarms as a 24-char hex string (per Phase 7 D-05 resolved)
- Delete call: `fetch('/Invoke/auth/deleteS7PlusAlarms', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ ids: [item._id] }) })`
- `originDbName` and `dbNumber` fields: present in each document from the list response (written by Phases 5/6, exposed by Phase 7)

</code_context>

<specifics>
## Specific Ideas

- Warning dialog wording from ROADMAP: "This alarm is still active on the PLC. Delete anyway?" — captured as D-07 with the full dialog spec.
- Delete Filtered button label pattern: "Delete Filtered (N)" where N = `filteredAlarms.length`.
- RelationId intentionally omitted — user said "no operator value for a raw 64-bit integer".

</specifics>

<deferred>
## Deferred Ideas

- RelationId column (ORIGIN-08) — explicitly dropped. If a future phase wants raw protocol debug columns, add as separate phase.
- "Delete Filtered" body shape when both filters are "All" — left to planner (D-08 / Claude's Discretion).

</deferred>

---

*Phase: 08-frontend-delete-buttons-origin-columns*
*Context gathered: 2026-03-24*

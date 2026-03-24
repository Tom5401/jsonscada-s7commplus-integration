# Phase 8: Frontend — Delete Buttons + Origin Columns - Research

**Researched:** 2026-03-24
**Domain:** Vue 3 + Vuetify 3 component modification (single-file change)
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Add two new columns: "Origin DB Name" (key: `originDbName`) and "DB Number" (key: `dbNumber`). Insert both immediately **after** the existing "ID" column (`cpuAlarmId`). Final column order: … | Event text | ID | Origin DB Name | DB Number | Additional text 1-3.
- **D-02:** **RelationId column is intentionally omitted.** ORIGIN-08 is dropped. Note the deviation in the plan.
- **D-03:** Add a dedicated "Delete" column positioned **immediately right of the Acknowledge column**. Final sequence: … | Status | Acknowledge | Delete | Alarm class name | … Header title: "Delete". Use `mdi-delete` icon, compact trash icon button pattern.
- **D-04:** The delete column uses `app.post` on `[authJwt.isAdmin]` — same cookie auth as the list endpoint. No explicit Authorization header in `fetch()`.
- **D-05:** "Delete Filtered (N)" button as a **third `v-col`** in the existing filter `v-row`. Disabled when `filteredAlarms.length === 0`. Label: `"Delete Filtered ({{ filteredAlarms.length }})"`.
- **D-06:** Per-row Delete on a normal row → **no confirmation dialog**. Fires immediately; row removed optimistically from `alarms.value`.
- **D-07:** Per-row Delete on a **Coming+unacked row** (`alarmState === 'Coming' && ackState === false`) → **warning dialog** before deleting. Title: "This alarm is still active on the PLC". Body: "Deleting it will only remove the history record — the alarm remains active on the PLC." Buttons: "Delete anyway" (red) + "Cancel". On confirm: proceed with delete and optimistic removal.
- **D-08:** "Delete Filtered (N)" bulk delete → **always show confirmation dialog**. Title: "Delete Filtered Alarms". Body: "Delete [N] alarm record(s)? This cannot be undone." Buttons: "Delete" (red) + "Cancel". On confirm: send `{ ids: [...] }` with all IDs of visible rows.
- **D-09:** On failed delete API call → **do nothing**. The 5-second auto-refresh will restore the row. No rollback, no snackbar.
- **D-10:** On successful delete (`response.ok`), remove the row immediately from `alarms.value`.

### Claude's Discretion

- Dialog implementation: `v-dialog` with `v-card` inline in template, or a reactive `confirmState` object — planner decides whichever is cleanest.
- Bulk delete body shape: if both filters are "All", send `{ ids: [...] }` with all visible IDs. Planner decides.
- Column widths/alignment: not specified; planner uses sensible Vuetify defaults.

### Deferred Ideas (OUT OF SCOPE)

- RelationId column (ORIGIN-08) — explicitly dropped. If a future phase wants raw protocol debug columns, add as a separate phase.
- "Delete Filtered" body shape when both filters are "All" — left to planner (D-08 / Claude's Discretion); sending `{ ids: [...] }` with all visible IDs is safe and explicit.

</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ORIGIN-06 | Alarm viewer displays Origin DB Name as a new column | `originDbName` field returned by `listS7PlusAlarms`; add to `headers` array + no slot needed (plain text render) |
| ORIGIN-07 | Alarm viewer displays DB Number as a new column | `dbNumber` field returned by `listS7PlusAlarms`; add to `headers` array + no slot needed (plain text render) |
| ORIGIN-08 | **INTENTIONALLY SKIPPED per D-02** — RelationId column omitted | Deviation recorded; no implementation |
| DELETE-04 | Per-row Delete button removes alarm document; row removed optimistically on success | `v-icon` button slot pattern; `fetch` POST to `/Invoke/auth/deleteS7PlusAlarms`; `alarms.value.splice(index, 1)` |
| DELETE-05 | "Delete Filtered (N)" toolbar button deletes all visible rows; disabled when zero visible | Third `v-col` in filter `v-row`; `v-btn :disabled="filteredAlarms.length === 0"` |
| DELETE-06 | Coming + unacked row shows warning before delete | `v-dialog` with conditional check `item.alarmState === 'Coming' && !item.ackState` |

</phase_requirements>

---

## Summary

Phase 8 is a **single-file frontend modification** to `S7PlusAlarmsViewerPage.vue`. The component is a Vue 3 `<script setup>` SFC using Vuetify 3.10 `v-data-table` with a `headers` reactive array and named slot templates (`#[item.{key}]`). All existing patterns (icon slots, filter UI, fetch-without-auth-headers) are directly reusable.

The two origin columns (`originDbName`, `dbNumber`) require only `headers` array additions — no slot templates are needed since the fields are plain strings/numbers rendered as-is. The delete column requires a new `headers` entry plus a `#[item.delete]` slot containing a `v-btn` icon. The "Delete Filtered (N)" button slots into the existing `v-row` filter bar as a third `v-col`. Two `v-dialog` instances handle confirmations: one for Coming+unacked rows, one for bulk delete.

The backend API is fully implemented (Phase 7). The delete endpoint accepts `{ ids: [...] }` (array of 24-char hex strings) and returns HTTP 204 on success. The `_id` field is serialized as a plain hex string (verified in `index.js` line 405 — no explicit mapping, using MongoDB driver's default EJSON serialization). The `response.ok` check covers 204. No new npm dependencies are required.

**Primary recommendation:** Add all changes inline in `S7PlusAlarmsViewerPage.vue` using a single `confirmState` reactive object to drive both dialog variants — this is the cleanest pattern for `<script setup>` and avoids duplicating dialog markup.

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Vue 3 | ^3.4.31 | Reactivity, composition API | Already in project |
| Vuetify | 3.10 | `v-data-table`, `v-dialog`, `v-btn`, `v-icon` | Already in project |
| Vite | ^7.3.1 | Build | Already in project |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| MDI icons | (via Vuetify) | `mdi-delete` icon for delete button | Already referenced in component (`mdi-check`, `mdi-close`) |

**No new npm packages required.** All UI primitives are already available via Vuetify 3.10.

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Inline `v-dialog` | Separate dialog component | Not needed — single-file phase, inline is simpler |
| `confirmState` object | Two separate `ref()` booleans | Object groups dialog data (type, item, count) — cleaner |

---

## Architecture Patterns

### Existing Component Structure (verified from source)

```
S7PlusAlarmsViewerPage.vue
├── <template>
│   ├── v-container
│   │   ├── v-row (filter bar — 2 v-col today, 3 after this phase)
│   │   ├── v-data-table :headers="headers" :items="filteredAlarms"
│   │   │   ├── #[item.alarmState]  — v-chip slot
│   │   │   ├── #[item.ackState]    — v-icon slot
│   │   │   ├── #[item.date]        — formatted slot
│   │   │   ├── #[item.time]        — formatted slot
│   │   │   └── #[item.additionalText1/2/3] — array-index slots
│   │   └── [NEW] v-dialog for confirmations
└── <script setup>
    ├── ref: alarms, statusFilter, alarmClassFilter, confirmState
    ├── computed: alarmClassOptions, filteredAlarms
    ├── functions: formatDate, formatTime, fetchAlarms
    └── lifecycle: onMounted (fetch + 5s timer), onUnmounted (clear timer)
```

### Pattern 1: Adding Plain-Text Columns (Origin DB Name, DB Number)

**What:** Add an entry to the `headers` array. No slot template required for plain text fields.
**When to use:** Field is a string or number rendered verbatim with no special formatting.

```javascript
// Source: verified from current headers array pattern in S7PlusAlarmsViewerPage.vue
{ title: 'Origin DB Name', key: 'originDbName', sortable: true },
{ title: 'DB Number', key: 'dbNumber', sortable: true },
```

Insert both entries after the `cpuAlarmId` entry in the `headers` array (D-01).

### Pattern 2: Icon Button Column (Delete)

**What:** Add a headers entry + `#[item.delete]` slot with a `v-btn` icon.
**When to use:** Column contains an interactive control, not a data value.

```javascript
// Source: Vuetify 3 v-data-table slot pattern — consistent with ackState slot
// In headers array (insert after ackState entry):
{ title: 'Delete', key: 'delete', sortable: false },
```

```html
<!-- Source: mirrors existing #[item.ackState] slot pattern -->
<template #[`item.delete`]="{ item }">
  <v-btn
    icon
    density="compact"
    variant="text"
    @click="handleDeleteRow(item)"
  >
    <v-icon color="red">mdi-delete</v-icon>
  </v-btn>
</template>
```

Note: `key: 'delete'` does not correspond to a field in the data object. Vuetify `v-data-table` renders the slot for any header key, even synthetic ones with no backing data field. This is the standard Vuetify pattern for action columns.

### Pattern 3: Delete Filtered Button in Filter Row

**What:** Add a third `v-col` in the filter `v-row` containing a `v-btn`.
**When to use:** Toolbar action that operates on the filtered set.

```html
<!-- Source: mirrors existing v-col > v-select pattern -->
<v-col cols="12" sm="4" md="3">
  <v-btn
    color="red"
    density="compact"
    :disabled="filteredAlarms.length === 0"
    @click="handleDeleteFiltered"
  >
    Delete Filtered ({{ filteredAlarms.length }})
  </v-btn>
</v-col>
```

### Pattern 4: Confirmation Dialog with `confirmState` Object

**What:** A single `v-dialog` driven by a reactive `confirmState` object, covering both the Coming+unacked warning and the bulk delete confirmation. The dialog content is conditional on `confirmState.type`.
**When to use:** Multiple dialog variants in one component — avoids duplicate `v-dialog` markup.

```javascript
// Source: Vue 3 composition API pattern
const confirmState = ref({
  visible: false,
  type: null,       // 'row-active' | 'bulk'
  item: null,       // set for 'row-active'
  count: 0          // set for 'bulk'
})

const handleDeleteRow = (item) => {
  if (item.alarmState === 'Coming' && !item.ackState) {
    confirmState.value = { visible: true, type: 'row-active', item, count: 0 }
  } else {
    executeDeleteRow(item)
  }
}

const handleDeleteFiltered = () => {
  confirmState.value = { visible: true, type: 'bulk', item: null, count: filteredAlarms.value.length }
}

const executeDeleteRow = async (item) => {
  const idx = alarms.value.findIndex(a => a._id === item._id)
  if (idx !== -1) alarms.value.splice(idx, 1)   // optimistic removal
  confirmState.value.visible = false
  const response = await fetch('/Invoke/auth/deleteS7PlusAlarms', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ ids: [item._id] })
  })
  // D-09: do nothing on failure — 5s refresh will restore
}

const executeDeleteFiltered = async () => {
  const ids = filteredAlarms.value.map(a => a._id)
  const toRemove = new Set(ids)
  alarms.value = alarms.value.filter(a => !toRemove.has(a._id))  // optimistic removal
  confirmState.value.visible = false
  await fetch('/Invoke/auth/deleteS7PlusAlarms', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ ids })
  })
  // D-09: do nothing on failure — 5s refresh will restore
}
```

**Critical:** Capture the filtered IDs into a local variable (`const ids = filteredAlarms.value.map(...)`) **before** the optimistic removal step. After `alarms.value` is filtered, `filteredAlarms` recomputes and the IDs would be gone.

### Pattern 5: `v-dialog` Markup for Confirmation

```html
<!-- Source: Vuetify 3.10 v-dialog + v-card inline pattern -->
<v-dialog v-model="confirmState.visible" max-width="400">
  <v-card>
    <v-card-title>
      <template v-if="confirmState.type === 'row-active'">
        This alarm is still active on the PLC
      </template>
      <template v-else>
        Delete Filtered Alarms
      </template>
    </v-card-title>
    <v-card-text>
      <template v-if="confirmState.type === 'row-active'">
        Deleting it will only remove the history record — the alarm remains active on the PLC.
      </template>
      <template v-else>
        Delete {{ confirmState.count }} alarm record(s)? This cannot be undone.
      </template>
    </v-card-text>
    <v-card-actions>
      <v-spacer />
      <v-btn @click="confirmState.visible = false">Cancel</v-btn>
      <v-btn
        color="red"
        @click="confirmState.type === 'row-active'
          ? executeDeleteRow(confirmState.item)
          : executeDeleteFiltered()"
      >
        <template v-if="confirmState.type === 'row-active'">Delete anyway</template>
        <template v-else>Delete</template>
      </v-btn>
    </v-card-actions>
  </v-card>
</v-dialog>
```

### Final `headers` Array Order

After Phase 8, the headers array must be in this order (matching D-01 and D-03):

```javascript
const headers = [
  { title: 'Source',            key: 'connectionId',   sortable: true  },
  { title: 'Date',              key: 'date',            sortable: false },
  { title: 'Time',              key: 'time',            sortable: false },
  { title: 'Status',            key: 'alarmState',      sortable: true  },
  { title: 'Acknowledge',       key: 'ackState',        sortable: true  },
  { title: 'Delete',            key: 'delete',          sortable: false },  // NEW — D-03
  { title: 'Alarm class name',  key: 'alarmClassName',  sortable: true  },
  { title: 'Event text',        key: 'alarmText',       sortable: true  },
  { title: 'ID',                key: 'cpuAlarmId',      sortable: true  },
  { title: 'Origin DB Name',    key: 'originDbName',    sortable: true  },  // NEW — D-01
  { title: 'DB Number',         key: 'dbNumber',        sortable: true  },  // NEW — D-01
  { title: 'Additional text 1', key: 'additionalText1', sortable: false },
  { title: 'Additional text 2', key: 'additionalText2', sortable: false },
  { title: 'Additional text 3', key: 'additionalText3', sortable: false },
]
```

### Anti-Patterns to Avoid

- **Capturing `filteredAlarms` after optimistic removal:** Always snapshot IDs before mutating `alarms.value`.
- **Using `item.index` for splice:** Use `findIndex(a => a._id === item._id)` — item indices from Vuetify slots are not always stable references.
- **Sending `{ filter: {...} }` for bulk delete when both dropdowns are "All":** Per D-08 / discretion note, always send `{ ids: [...] }` with explicit IDs for all visible rows. This avoids the empty-filter 400 guard on the backend.
- **Adding `Authorization` header to fetch:** The existing `fetchAlarms` and the ack pattern do not set auth headers — cookies handle auth. Match this pattern (D-04).

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Modal confirmation dialogs | Custom overlay/backdrop component | Vuetify `v-dialog` + `v-card` | Already in Vuetify 3; handles accessibility, focus trap, z-index |
| Icon button | `<button>` with CSS | `v-btn icon variant="text"` | Consistent density, ripple, and disabled state with existing Vuetify table |
| Disabled button state | Manual CSS class toggle | `:disabled="filteredAlarms.length === 0"` on `v-btn` | Vuetify handles aria-disabled and visual state |

**Key insight:** Every UI primitive needed in this phase already exists in Vuetify 3.10, which is installed and configured. No additional packages are required.

---

## Common Pitfalls

### Pitfall 1: IDs Captured After Optimistic Removal

**What goes wrong:** `filteredAlarms.value.map(a => a._id)` is called after `alarms.value` is mutated — returns empty array, nothing is sent to the backend.
**Why it happens:** `filteredAlarms` is a computed that derives from `alarms.value`; mutation invalidates it immediately.
**How to avoid:** Snapshot IDs in a `const ids = filteredAlarms.value.map(a => a._id)` line before any mutation.
**Warning signs:** `fetch` body contains `{ ids: [] }` in network tab.

### Pitfall 2: `response.ok` Is False for 204

**What goes wrong:** Some implementations check `response.status === 200` which misses 204. The delete endpoint returns 204 (No Content).
**Why it happens:** 204 is not 200. `response.ok` is true for 200–299, so use `response.ok` (D-10), not a status code equality check.
**How to avoid:** Always use `if (response.ok)` — already the pattern specified in D-10.

### Pitfall 3: `key: 'delete'` Column Causes Vuetify Warning About Missing Field

**What goes wrong:** Vuetify `v-data-table` may log a warning when a header key has no corresponding field in the data items (because `delete` is a JS keyword / reserved word in some contexts, and it's not a real data field).
**Why it happens:** Vuetify validates headers against item keys in development mode.
**How to avoid:** Provide a `#[item.delete]` slot — this suppresses the warning because the column is fully controlled by the slot. No data binding is attempted.

### Pitfall 4: Dialog Closes Before `executeDeleteRow` Receives Item

**What goes wrong:** `confirmState.value.visible = false` is set before the item reference is read, causing `item` to be null when `executeDeleteRow` is called.
**Why it happens:** `visible = false` triggers reactivity that resets the dialog state if `confirmState` is a plain object that gets reset.
**How to avoid:** In `executeDeleteRow`/`executeDeleteFiltered`, close the dialog (`confirmState.value.visible = false`) only after reading the item/ids from `confirmState`. Or pass item as a parameter when calling from the dialog's confirm button.

### Pitfall 5: Bulk Delete Sends `{ filter: {} }` Instead of `{ ids: [...] }`

**What goes wrong:** The backend rejects `{ filter: {} }` with HTTP 400 (empty-filter guard). If both dropdowns are "All", constructing a filter body produces an empty object.
**Why it happens:** Developer follows the filter-based path when both filters are "All".
**How to avoid:** Always use `{ ids: filteredAlarms.value.map(a => a._id) }` for bulk delete from the UI (D-08 discretion decision).

---

## Code Examples

### Delete Row — Single Call

```javascript
// Source: derived from D-04 (same cookie auth, no Authorization header)
// and D-09/D-10 (no error handling, optimistic removal on ok)
const executeDeleteRow = async (item) => {
  // Optimistic removal first (D-10)
  const idx = alarms.value.findIndex(a => a._id === item._id)
  if (idx !== -1) alarms.value.splice(idx, 1)
  confirmState.value.visible = false

  // Best-effort delete — D-09: do nothing on failure
  await fetch('/Invoke/auth/deleteS7PlusAlarms', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ ids: [item._id] })
  })
}
```

### Delete Filtered — Bulk Call

```javascript
// Source: D-08 (ids-based), D-09 (no error handling)
const executeDeleteFiltered = async () => {
  // Snapshot BEFORE mutation (Pitfall 1)
  const ids = filteredAlarms.value.map(a => a._id)
  const toRemove = new Set(ids)

  // Optimistic removal (D-10)
  alarms.value = alarms.value.filter(a => !toRemove.has(a._id))
  confirmState.value.visible = false

  // Best-effort delete — D-09: do nothing on failure
  await fetch('/Invoke/auth/deleteS7PlusAlarms', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ ids })
  })
}
```

### `handleDeleteRow` — Gate Logic

```javascript
// Source: D-06, D-07
const handleDeleteRow = (item) => {
  if (item.alarmState === 'Coming' && !item.ackState) {
    // D-07: show warning dialog for active+unacked alarm
    confirmState.value = { visible: true, type: 'row-active', item, count: 0 }
  } else {
    // D-06: no dialog for normal rows — fire immediately
    executeDeleteRow(item)
  }
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Vuetify 2 `v-data-table` slot: `#item.{key}` | Vuetify 3: `#[item.{key}]` (dynamic slot) | Vuetify 3.0 | Already using correct syntax |
| `v-dialog` with `v-model` boolean | Same — no change in Vuetify 3.10 | — | `v-model` on `v-dialog` binds to `visible` |

No deprecated patterns identified in the current component.

---

## Open Questions

1. **`_id` serialization as plain string vs EJSON object**
   - What we know: Phase 7 code at index.js line 363 uses `res.status(200).send(docs)` without explicit `.map(d => ({ ...d, _id: d._id.toHexString() }))`. MongoDB Node.js driver v5+ serializes ObjectId via `JSON.stringify` as `{ "$oid": "..." }` in strict EJSON mode, but Express `res.send(object)` calls `JSON.stringify` which triggers `.toJSON()` on ObjectId — returning a plain hex string in modern driver versions.
   - What's unclear: Whether the installed MongoDB driver version serializes ObjectId to a plain string or `{ $oid: "..." }` via `res.send()`.
   - Recommendation: The planner should verify by checking `json-scada/src/server_realtime_auth/package.json` for the `mongodb` driver version. If `mongodb >= 4.0`, `ObjectId.toJSON()` returns `{ id: Buffer, ... }` but `JSON.stringify(new ObjectId())` returns the 24-char hex string. The safest plan action is to verify the `_id` shape at runtime (network tab) before writing the fetch call, or explicitly map `_id` to `toHexString()` in the backend (which Phase 7 planner may have already done). The frontend code should treat `item._id` as a plain string.

---

## Environment Availability

Step 2.6: SKIPPED — Phase 8 is a pure frontend code change (single `.vue` file modification). No external dependencies beyond the already-installed npm packages in AdminUI. No CLI tools, databases, or services need to be provisioned.

---

## Validation Architecture

`nyquist_validation` is enabled in `.planning/config.json`.

### Test Framework

| Property | Value |
|----------|-------|
| Framework | None detected — no test config files or test directories found in AdminUI project |
| Config file | None |
| Quick run command | N/A — no automated tests exist |
| Full suite command | N/A |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| ORIGIN-06 | "Origin DB Name" column renders in table | manual-only | — | ❌ Wave 0 |
| ORIGIN-07 | "DB Number" column renders in table | manual-only | — | ❌ Wave 0 |
| ORIGIN-08 | **SKIPPED per D-02** | — | — | — |
| DELETE-04 | Per-row delete removes row optimistically | manual-only | — | ❌ Wave 0 |
| DELETE-05 | "Delete Filtered (N)" button count and disable | manual-only | — | ❌ Wave 0 |
| DELETE-06 | Coming+unacked row shows warning dialog | manual-only | — | ❌ Wave 0 |

**Justification for manual-only:** The AdminUI project has no automated test infrastructure (no vitest/jest config, no `tests/` directory). All prior phases in this project have followed a browser/runtime verification pattern. Adding a full vitest + `@vue/test-utils` + vuetify-mocking stack is out of scope for this phase.

### Sampling Rate

- **Per task commit:** Browser visual check — load the alarm viewer, verify columns/buttons render
- **Per wave merge:** Full manual verification checklist (see PLAN.md verification section)
- **Phase gate:** Manual verification passes before `/gsd:verify-work`

### Wave 0 Gaps

None — the phase does not introduce automated tests. Existing test infrastructure: none to configure. Manual verification is the established project pattern for AdminUI changes.

---

## Sources

### Primary (HIGH confidence)

- Direct file read: `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` (lines 1–140) — full component source, all existing patterns confirmed
- Direct file read: `json-scada/src/server_realtime_auth/index.js` (lines 340–411) — list and delete endpoint implementations confirmed; 204 response, `{ ids }` / `{ filter }` body shapes confirmed
- Direct file read: `json-scada/src/AdminUI/package.json` — Vuetify 3.10, Vue 3.4.31 confirmed
- Direct file read: `.planning/phases/08-frontend-delete-buttons-origin-columns/08-CONTEXT.md` — all decisions D-01 through D-10 confirmed

### Secondary (MEDIUM confidence)

- Vuetify 3 `v-data-table` slot patterns — inferred from existing component code using `#[item.{key}]` syntax; consistent with Vuetify 3 documentation
- `v-dialog` v-model pattern — standard Vuetify 3 usage; no breaking changes in 3.10 for this API

### Tertiary (LOW confidence)

- MongoDB Node.js driver ObjectId JSON serialization behavior — described from training knowledge; should be verified at runtime or by checking the installed driver version

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all dependencies verified from package.json and existing component
- Architecture: HIGH — all patterns derived from reading the actual source file
- Pitfalls: HIGH — derived from direct analysis of the code; Pitfall 1 (IDs capture order) is a real concurrency trap in the reactive pattern
- API contracts: HIGH — delete endpoint fully implemented and readable in index.js
- _id serialization: LOW — training data only; runtime verification recommended

**Research date:** 2026-03-24
**Valid until:** 2026-04-24 (stable — no fast-moving dependencies; Vuetify 3.10 / Vue 3.4 are stable)

---

## ORIGIN-08 Deviation Note

REQUIREMENTS.md marks ORIGIN-08 as a Phase 8 requirement. Per **D-02** (locked decision from CONTEXT.md), the RelationId column is **intentionally omitted**. The plan MUST document this deviation explicitly so reviewers understand ORIGIN-08 was a conscious drop, not an oversight.

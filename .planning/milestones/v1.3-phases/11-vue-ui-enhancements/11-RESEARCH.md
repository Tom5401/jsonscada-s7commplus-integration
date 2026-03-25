# Phase 11: Vue UI Enhancements - Research

**Researched:** 2026-03-25
**Domain:** Vuetify 3 / Vue 3 — single-file component modifications
**Confidence:** HIGH

## Summary

This phase is a contained, single-file frontend change. All six requirements target `S7PlusAlarmsViewerPage.vue` only — no backend, no new dependencies, no new routes. The existing component already provides the patterns for every change (filters, bulk-action buttons, `ackAlarm()`, dialog dispatch). The work is additive and mechanical: extend existing patterns rather than introduce new ones.

The most technically interesting item is VIEWER-06 (page preservation). The installed Vuetify 3.10.12 exposes a `page` prop that uses `useProxiedModel`, making `v-model:page` fully supported and the correct binding. The internal watch in `providePagination` clamps the page when `page > pageCount` after a refresh — this is safe behaviour and does not cause unwanted resets as long as the page is within range.

All other items (timestamp formatting, priority sorting, ack indicator, source filter, Ack All) follow patterns already present in the file and require no library research beyond what is already verified.

**Primary recommendation:** Implement all six changes in a single wave against `S7PlusAlarmsViewerPage.vue`, following existing patterns strictly. No new packages, no new files.

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** No new column for ack indicator. The existing Acknowledge column is extended: `isAcknowledgeable === false` → display dash (`-`); `isAcknowledgeable === true` → existing behavior unchanged.
- **D-02:** Ack button visibility remains driven by `ackState` (not `isAcknowledgeable`). The dash is purely visual.
- **D-03:** "Ack All" button lives in the filter row (`v-row`), next to the "Delete Filtered" button.
- **D-04:** Ack All targets alarms matching current filter where `!alarm.ackState && alarm.isAcknowledgeable === true`. Confirmation dialog shows that count.
- **D-05:** Each ack is attempted independently via `ackAlarm()` — single failure does not block the rest.
- **D-06:** Ack All button is disabled when count of unacked+acknowledgeable alarms matching active filter is 0.
- **D-07:** Priority column is unsorted on load. User clicks header to sort. No `:sort-by` prop.
- **D-08:** `priority` is already in every MongoDB document — no API changes needed.
- **D-09:** Column order: `Source | Timestamp | Priority | Status | Acknowledge | Delete | Alarm class name | Event text | ID | Origin DB Name | DB Number | Additional text 1 | Additional text 2 | Additional text 3`
- **D-10:** `date` and `time` columns replaced by single `timestamp` column. Format: `2026-03-24_12:57:10.758` — ISO date with underscore separator, time with milliseconds, local time. Implement via `formatTimestamp()`.
- **D-11:** Source filter follows same pattern as `alarmClassOptions`: computed property extracting distinct `connectionName` values from `alarms.value`, sorted, `'All'` prepended. `connectionFilter` ref defaults to `'All'`. Added to `filteredAlarms` alongside existing filters.
- **D-12:** Use `v-model:page` on `v-data-table` bound to `currentPage` ref (initialized to `1`). `fetchAlarms` updates `alarms.value` without touching `currentPage`.

### Claude's Discretion

- Exact label for the Ack All confirmation dialog — use clear operator language consistent with existing Delete dialogs.
- `formatTimestamp()` implementation detail — manual string construction (`YYYY-MM-DD_HH:MM:SS.mmm` from `Date` object properties).

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| VIEWER-01 | Single combined timestamp column replacing Date + Time, formatted `2026-03-24_12:57:10.758` (local time, ms precision) | Verified: existing `formatDate`/`formatTime` slot pattern; `Date` object has `getFullYear`, `getMonth`, `getDate`, `getHours`, `getMinutes`, `getSeconds`, `getMilliseconds` for manual construction |
| VIEWER-02 | Sortable Priority column using existing `priority` field | Verified: Vuetify `headers` entry with `sortable: true` enables click-to-sort; no `:sort-by` prop needed for unsorted default |
| VIEWER-03 | Ack indicator in Acknowledge column — dash for `isAcknowledgeable === false`, existing logic for true | Verified: existing `#[item.ackState]` slot pattern extended with `v-if` on `item.isAcknowledgeable` |
| VIEWER-04 | Source PLC filter dropdown based on `connectionName` | Verified: exact pattern exists as `alarmClassOptions` computed + `alarmClassFilter` ref in filteredAlarms |
| VIEWER-05 | "Ack All" button with confirmation dialog showing count; independent per-alarm ack | Verified: `confirmState` ref + dialog type dispatch already supports multiple types; `ackAlarm()` loop pattern confirmed |
| VIEWER-06 | Current page preserved across 5-second auto-refresh | Verified: `v-model:page` supported via `useProxiedModel` in Vuetify 3.10.12 `paginate.js`; `page` prop is `Number | String` with default 1 |
</phase_requirements>

---

## Standard Stack

### Core (all already in project)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Vue 3 | ^3.4.31 | Reactive composition API | Project standard |
| Vuetify | 3.10.12 | UI components (`v-data-table`, `v-select`, `v-btn`, `v-dialog`) | Project standard |
| @mdi/font | 7.4.47 | Icon set (mdi-check, mdi-delete, etc.) | Project standard |

### No New Dependencies

All changes are achievable with the existing stack. No `npm install` step is required.

## Architecture Patterns

### Single-File Component (SFC) Structure

The component is a Vue 3 SFC using `<script setup>` with Composition API. All six changes follow patterns already established in the file:

```
S7PlusAlarmsViewerPage.vue
├── <template>
│   ├── v-row (filter controls + bulk actions)     ← add connectionFilter v-select + Ack All btn here
│   ├── v-data-table                               ← add v-model:page; update headers; add timestamp/priority slots
│   └── v-dialog (confirmState dispatch)           ← extend to handle 'ack-all' type
└── <script setup>
    ├── refs: alarms, statusFilter, alarmClassFilter, pendingAcks, confirmState
    │         + NEW: connectionFilter, currentPage
    ├── computed: alarmClassOptions, filteredAlarms
    │             + NEW: connectionOptions, ackAllCount
    ├── functions: ackAlarm, fetchAlarms, handleDelete*, executeDelete*
    │              + NEW: formatTimestamp, handleAckAll, executeAckAll
    └── lifecycle: onMounted, onUnmounted
```

### Pattern 1: Filter Dropdown (for VIEWER-04)

Copy the `alarmClassOptions` pattern exactly:

```javascript
// Source: existing alarmClassOptions (line 206)
const connectionOptions = computed(() => {
  const names = [...new Set(
    alarms.value.map(a => a.connectionName).filter(Boolean)
  )]
  return ['All', ...names.sort()]
})

const connectionFilter = ref('All')
```

Add to `filteredAlarms` computed:
```javascript
const connectionMatch =
  connectionFilter.value === 'All' ||
  alarm.connectionName === connectionFilter.value
return stateMatch && classMatch && connectionMatch
```

### Pattern 2: Confirmation Dialog Type Extension (for VIEWER-05)

Extend the existing `confirmState` type dispatch — add `'ack-all'` as a third type alongside `'row-active'` and `'bulk'`:

```javascript
// In <v-card-title>:
<template v-else-if="confirmState.type === 'ack-all'">
  Acknowledge All Matching Alarms
</template>

// In <v-card-text>:
<template v-else-if="confirmState.type === 'ack-all'">
  Acknowledge {{ confirmState.count }} unacked alarm(s) matching current filter?
</template>

// In the confirm button @click:
confirmState.type === 'row-active'
  ? executeDeleteRow(confirmState.item)
  : confirmState.type === 'ack-all'
    ? executeAckAll()
    : executeDeleteFiltered()
```

### Pattern 3: Ack All Execute (for VIEWER-05)

```javascript
const handleAckAll = () => {
  confirmState.value = { visible: true, type: 'ack-all', item: null, count: ackAllCount.value }
}

const executeAckAll = async () => {
  confirmState.value.visible = false
  const targets = filteredAlarms.value.filter(a => !a.ackState && a.isAcknowledgeable === true)
  for (const alarm of targets) {
    await ackAlarm(alarm.cpuAlarmId, alarm.connectionId)
  }
}

const ackAllCount = computed(() =>
  filteredAlarms.value.filter(a => !a.ackState && a.isAcknowledgeable === true).length
)
```

Note: `await` on each `ackAlarm` call ensures they run sequentially. A failed ack removes the ID from `pendingAcks` (existing error handling in `ackAlarm`) and the loop continues. This matches D-05.

### Pattern 4: Timestamp Format (for VIEWER-01)

Manual string construction from `Date` object properties — avoids locale variation:

```javascript
const formatTimestamp = (isoStr) => {
  if (!isoStr) return ''
  const d = new Date(isoStr)
  const YYYY = d.getFullYear()
  const MM = String(d.getMonth() + 1).padStart(2, '0')
  const DD = String(d.getDate()).padStart(2, '0')
  const hh = String(d.getHours()).padStart(2, '0')
  const mm = String(d.getMinutes()).padStart(2, '0')
  const ss = String(d.getSeconds()).padStart(2, '0')
  const ms = String(d.getMilliseconds()).padStart(3, '0')
  return `${YYYY}-${MM}-${DD}_${hh}:${mm}:${ss}.${ms}`
}
```

Template slot:
```html
<template #[`item.timestamp`]="{ item }">
  {{ formatTimestamp(item.timestamp) }}
</template>
```

Header entry:
```javascript
{ title: 'Timestamp', key: 'timestamp', sortable: false }
```

(Remove the existing `date` and `time` header entries and their template slots.)

### Pattern 5: Page Preservation (for VIEWER-06)

```javascript
// Verified from paginate.js: useProxiedModel(props, 'page', ...) emits 'update:page'
const currentPage = ref(1)
```

```html
<v-data-table
  v-model:page="currentPage"
  ...
>
```

`fetchAlarms` needs no changes — it only updates `alarms.value`. The Vuetify internal watch in `providePagination` resets page to `pageCount` if `page > pageCount` after a data refresh. Since `currentPage` is never written by `fetchAlarms`, the page is preserved as long as the new data has enough items for the page to exist.

### Pattern 6: Ack Indicator (for VIEWER-03)

Extend the existing `#[item.ackState]` slot:

```html
<template #[`item.ackState`]="{ item }">
  <template v-if="item.isAcknowledgeable === false">
    <span>-</span>
  </template>
  <template v-else>
    <v-icon v-if="item.ackState" color="green">mdi-check</v-icon>
    <template v-else>
      <v-progress-circular
        v-if="pendingAcks.has(item.cpuAlarmId)"
        indeterminate
        size="16"
        width="2"
      />
      <v-btn
        v-else
        size="x-small"
        variant="tonal"
        @click="ackAlarm(item.cpuAlarmId, item.connectionId)"
      >
        Ack
      </v-btn>
    </template>
  </template>
</template>
```

### Anti-Patterns to Avoid

- **Modifying `currentPage` inside `fetchAlarms`:** Do not reset `currentPage` to 1 in the fetch function. The entire point of VIEWER-06 is that fetch does NOT touch page state.
- **Using `ackAlarm()` without `await` in the Ack All loop:** Fire-and-forget would make the "no-abort-on-failure" contract harder to verify and could flood the server.
- **Gating Ack button visibility on `isAcknowledgeable`:** Explicitly out of scope per REQUIREMENTS.md. The dash (`-`) is display-only.
- **Adding a new column for the ack indicator:** Locked by D-01. Extend the existing Acknowledge column only.
- **Using `:sort-by` to set a default sort on Priority:** D-07 specifies no default. The `sortable: true` header property alone enables click-to-sort with no initial order.
- **Using `toLocaleDateString()` / `toLocaleTimeString()` for the timestamp:** These produce locale-dependent output. The required format `2026-03-24_12:57:10.758` is fixed — use manual string construction.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Sort on click | Custom sort handler | Vuetify `{ sortable: true }` in headers | Already works via `provideSort` composable |
| Page tracking | Custom scroll/DOM logic | `v-model:page` on `v-data-table` | `useProxiedModel` handles two-way binding correctly |
| Distinct filter values | Manual dedup loop | `[...new Set(...)]` (already in `alarmClassOptions`) | One-liner, already established pattern |

## Common Pitfalls

### Pitfall 1: Page Clamping on Filter Change

**What goes wrong:** If the operator is on page 3 and applies a filter that reduces results to fewer items than page 3 holds, the internal `providePagination` watch clamps page to `pageCount`. This is correct behaviour. However, if `currentPage` ref becomes out of sync, the table may appear stuck.
**Why it happens:** `v-model:page` binds bidirectionally — the table can write back to `currentPage` when it clamps. This is expected.
**How to avoid:** Accept the clamp. Do not add logic to prevent it — the clamping is safe.
**Warning signs:** Page jumps to last available page when filter is applied. This is correct, not a bug.

### Pitfall 2: `isAcknowledgeable` as `undefined` vs `false`

**What goes wrong:** Alarms stored before DRIVER-01 (Phase 9) was applied will have `isAcknowledgeable` as `undefined`, not `false`. The condition `item.isAcknowledgeable === false` (strict equality) will NOT show a dash for those alarms — they will fall into the existing Ack button path.
**Why it happens:** Strict equality `=== false` excludes `undefined`.
**How to avoid:** This is intentional per D-01 and the REQUIREMENTS.md note ("display hint only"). Alarms without the field keep the existing Ack button behaviour. No special handling needed.

### Pitfall 3: `ackAlarm` Receives `connectionId` (Number), Not `connectionNumber`

**What goes wrong:** The existing `ackAlarm(cpuAlarmId, connectionNumber)` function sends `connectionNumber` in the POST body. Looking at the template, the current call is `ackAlarm(item.cpuAlarmId, item.connectionId)`. These names are consistent — `item.connectionId` is the connection number.
**Why it happens:** Field naming divergence between what the MongoDB document stores (`connectionId`) and what the API parameter is called (`connectionNumber`).
**How to avoid:** In the Ack All loop, pass `alarm.connectionId` as the second argument to `ackAlarm()`, matching the existing per-row call.

### Pitfall 4: Removing `formatDate` / `formatTime` While Slots Still Reference Them

**What goes wrong:** If the `date` and `time` template slots (`#[item.date]` and `#[item.time]`) are not removed along with the header entries, Vue will produce warnings about orphaned slots. The `formatDate` and `formatTime` functions can be removed once no slots reference them.
**How to avoid:** Remove both template slots AND the two header entries AND the two helper functions in the same edit.

### Pitfall 5: `executeAckAll` Running on Stale Filtered List After Dialog Opens

**What goes wrong:** Between the time the confirmation dialog opens (count is captured) and the time the operator clicks Confirm, the 5-second auto-refresh may run and change `filteredAlarms`. `executeAckAll` re-computes targets from the live `filteredAlarms` at execution time, so the actual acknowledged count may differ from the confirmed count.
**Why it happens:** Reactive computed re-evaluates on data change.
**How to avoid:** This is acceptable for a PoC. The confirmation dialog is advisory. Document this as expected behaviour. Do not snapshot the list.

## Code Examples

### Verified: `v-model:page` works in Vuetify 3.10.12

Source: `node_modules/vuetify/lib/components/VDataTable/composables/paginate.js` (line 17)

```javascript
// Inside Vuetify source — createPagination
const page = useProxiedModel(props, 'page', undefined, value => Number(value ?? 1));
```

Source: `node_modules/vuetify/lib/composables/proxiedModel.js` (line 37)

```javascript
vm?.emit(`update:${prop}`, newValue);  // emits 'update:page'
```

This confirms `v-model:page="currentPage"` is a valid two-way binding.

### Verified: `sortable: true` enables click-to-sort with no default

Source: existing headers array (line 180–194 of the component):

```javascript
{ title: 'Source', key: 'connectionId', sortable: true },
{ title: 'Status', key: 'alarmState', sortable: true },
```

Adding `{ title: 'Priority', key: 'priority', sortable: true }` follows the same pattern. No `:sort-by` prop on `v-data-table` means unsorted on load.

## State of the Art

This is an internal component change — no library upgrades or migrations.

| Area | Current | After Phase 11 |
|------|---------|----------------|
| Timestamp display | Two columns: Date (locale) + Time (locale) | One column: fixed `YYYY-MM-DD_HH:MM:SS.mmm` format |
| Priority column | Present in data, not shown in table | Sortable column in table |
| Ack indicator | Binary: Ack button / checkmark | Tri-state: dash / Ack button / checkmark |
| Source filter | Not present | Dropdown matching Status/AlarmClass pattern |
| Bulk ack | Not present | "Ack All" button with count confirmation |
| Page on refresh | Jumps to page 1 on every 5-second refresh | Preserved via `v-model:page` |

## Open Questions

1. **`executeAckAll` — sequential vs parallel**
   - What we know: D-05 says single failure must not block the rest; `ackAlarm` already handles errors internally
   - What's unclear: Sequential `await` adds latency when there are many alarms; parallel `Promise.all` would be faster but error isolation is the same
   - Recommendation: Use sequential `await` in a `for...of` loop. PoC scale (hundreds of alarms at most); latency is not a concern. Simpler to reason about.

2. **`connectionFilter` reset on data refresh**
   - What we know: `connectionOptions` is a computed that depends on `alarms.value`; if a PLC disappears from the alarm list during a refresh, the selected `connectionFilter` value will no longer appear in the dropdown
   - What's unclear: Whether Vuetify `v-select` handles a `v-model` value that is absent from `:items` gracefully
   - Recommendation: No special handling needed for PoC. Vuetify renders the stale value as-is in the select until the user opens the dropdown. Safe to leave.

## Environment Availability

Step 2.6: SKIPPED (no external dependencies — pure frontend component change, no CLI tools, services, or runtimes needed beyond what the dev server already provides).

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | None detected — no test config files found in `json-scada/src/AdminUI/` |
| Config file | None |
| Quick run command | N/A — Wave 0 must establish test infrastructure if automated testing is required |
| Full suite command | N/A |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| VIEWER-01 | `formatTimestamp()` produces `2026-03-24_12:57:10.758` for a known ISO string | unit | Not yet configured | Wave 0 gap |
| VIEWER-02 | Priority column header is sortable (click triggers sort) | manual smoke | N/A | N/A |
| VIEWER-03 | Dash shows for `isAcknowledgeable === false`; Ack button shows for `true` | manual smoke | N/A | N/A |
| VIEWER-04 | connectionOptions computed returns `['All', ...distinct connectionNames]` | unit | Not yet configured | Wave 0 gap |
| VIEWER-05 | ackAllCount computed counts unacked+acknowledgeable in filteredAlarms | unit | Not yet configured | Wave 0 gap |
| VIEWER-06 | `currentPage` ref unchanged after `fetchAlarms` runs | unit | Not yet configured | Wave 0 gap |

**Note on test infrastructure:** No Vitest, Jest, or other test runner is configured in `AdminUI`. Given the PoC context and the single-file nature of these changes, manual smoke testing against a running instance is the practical verification path for this phase. If the planner elects to add unit tests, Wave 0 must install Vitest and configure it before implementation tasks.

### Wave 0 Gaps (if unit tests are desired)
- [ ] Install Vitest: `npm install -D vitest @vue/test-utils jsdom` in `json-scada/src/AdminUI/`
- [ ] `vitest.config.ts` — configure jsdom environment
- [ ] `tests/S7PlusAlarmsViewerPage.spec.js` — covers VIEWER-01, VIEWER-04, VIEWER-05, VIEWER-06 computed/function logic

**If no test infrastructure is added:** Manual smoke testing is sufficient for this PoC phase. The planner may omit Wave 0 test setup.

## Sources

### Primary (HIGH confidence)
- `node_modules/vuetify/lib/components/VDataTable/composables/paginate.js` — confirms `v-model:page` via `useProxiedModel`; confirms `page` prop type `[Number, String]` default `1`; confirms internal page-clamping watch
- `node_modules/vuetify/lib/composables/proxiedModel.js` — confirms `emit('update:page', ...)` pattern
- `node_modules/vuetify/lib/components/VDataTable/VDataTable.d.ts` — confirms `page` prop at line 930 (`string | number`)
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — all existing patterns read directly

### Secondary (MEDIUM confidence)
- Vuetify 3 docs (vuetifyjs.com) — confirmed `v-data-table` pagination controls exist; full content not directly accessible but source code verification supersedes docs

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — installed versions read directly from package.json and node_modules
- Architecture: HIGH — all patterns read directly from the production component file
- `v-model:page` support: HIGH — verified in installed Vuetify 3.10.12 source
- Pitfalls: MEDIUM — three of five pitfalls are direct code observations; two (filter stale value, sequential vs parallel ack) are design judgment calls
- Validation: MEDIUM — no test infrastructure exists; recommendations are based on project context (PoC)

**Research date:** 2026-03-25
**Valid until:** 2026-04-25 (Vuetify 3.x stable; patterns won't change)

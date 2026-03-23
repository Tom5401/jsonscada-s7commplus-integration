# Phase 3: Read-Only Alarm Viewer - Research

**Researched:** 2026-03-18
**Domain:** Vue 3 / Vuetify 3 frontend component + Node.js/Express backend endpoint + MongoDB raw driver query
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Add a Dashboard tile in `DashboardPage.vue` for the new viewer alongside the existing "Alarms Viewer" tile
- Label: **"S7Plus Alarms"** (i18n key: `dashboard.s7plusAlarms`)
- Route: `/s7plus-alarms` (as specified in roadmap)
- New component: `S7PlusAlarmsViewerPage.vue`
- Load the **last 200 most recent** alarm events (sort by `createdAt` descending, limit 200)
- Default sort: **newest first** in the displayed table
- 200 is a compile-time constant in the component — no configuration UI needed
- Filters in a **toolbar row above the table** (`v-row` or `v-toolbar`-style layout)
- Both Status and Alarm class filters use **`v-select` dropdowns**
- Status filter options: All / Incoming / Outgoing
- Alarm class filter: populated from distinct `alarmClassName` values in the currently loaded dataset (dynamic, not hardcoded)
- Filtering is **client-side** — no re-fetch on filter change
- Status column: colored `v-chip` — red chip for "Coming", green chip for "Going"
- Acknowledge column: icon — `mdi-check` green for `ackState: true`, `mdi-close` red for `ackState: false`
- Auto-refresh: `setInterval` at 5000ms, cleared in `onUnmounted`
- Fetches the same full-200-record query on each tick
- `AlarmsViewerPage.vue` must NOT be modified

### Claude's Discretion
- Exact backend endpoint path and implementation (new route in `server_realtime_auth` or direct MongoDB via existing `/Invoke/auth/` pattern)
- Whether to require auth token for the new endpoint (follow existing auth patterns)
- Exact i18n key names beyond the Dashboard label
- Column widths and `v-data-table` density setting
- How Date and Time are split from the `timestamp` field (ISO string parsing)

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| VIEW-01 | User can navigate to a dedicated S7Plus Alarms Viewer page in AdminUI, separate from the existing tag-based alarm viewer | Router registration pattern confirmed; existing `/alarms-viewer` route unchanged |
| VIEW-02 | Alarm viewer displays TIA Portal-equivalent columns — Source, Date, Time, Status, Acknowledge, Alarm class name, Event text, ID, Additional texts 1–3 | MongoDB document schema fully confirmed from `AlarmThread.cs`; all columns map to real fields |
| VIEW-04 | Alarm viewer automatically refreshes to show new alarms without manual page reload | `setInterval` / `onUnmounted` clearInterval pattern confirmed; 5000ms interval locked |
| VIEW-05 | User can filter displayed alarms by status (Incoming / Outgoing / All) | Client-side filter on `alarmState` field ("Coming"/"Going") confirmed |
| VIEW-06 | User can filter displayed alarms by alarm class | Client-side filter on `alarmClassName` field; distinct values derived from loaded dataset |
</phase_requirements>

---

## Summary

Phase 3 adds a fully independent S7Plus Alarms Viewer page to AdminUI. The work spans three layers: (1) a new backend GET endpoint that queries the `s7plusAlarmEvents` MongoDB collection, (2) a new Vue 3 / Vuetify 3 component `S7PlusAlarmsViewerPage.vue` that fetches, auto-refreshes, filters, and displays alarm data, and (3) routing/navigation wiring that adds the page to AdminUI without touching existing files other than three specific integration points.

The critical architectural finding is that the existing `auth.controller.js` uses **Mongoose models** — which have no model registered for the raw `s7plusAlarmEvents` collection. The native MongoDB `db` handle (from the Node.js `mongodb` driver) is scoped to `index.js` and is not passed to auth.routes/auth.controller. The recommended approach for the new endpoint is to register it **directly in `index.js`** immediately after the existing auth routes block, using `[authJwt.isAdmin]` middleware inline. This gives the handler access to `db` in closure without any structural changes to the route/controller separation.

The frontend implementation is a near-direct composition of established AdminUI patterns: `v-data-table` + `ref([])` + `onMounted/fetch` + `setInterval/onUnmounted`. All UI components (`v-chip`, `v-select`, `v-icon`) are confirmed present and used elsewhere in AdminUI. The MongoDB document schema is completely defined in `AlarmThread.cs` and confirmed from Phase 2.

**Primary recommendation:** Register the new backend endpoint inline in `index.js` (not in auth.controller) to access the native MongoDB `db` handle. Model the Vue component after `ProtocolConnectionsTab.vue`'s fetch pattern and `DashboardPage.vue`'s structure.

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Vue 3 (Composition API) | ^3.4.31 | Component reactivity and lifecycle | Already in use throughout AdminUI |
| Vuetify | 3.10 | `v-data-table`, `v-select`, `v-chip`, `v-icon` | All confirmed present and used in existing components |
| vue-router (hash mode) | existing | Route registration | `createWebHashHistory` confirmed in `router/index.js` |
| vue-i18n | existing | `$t()` translation keys | Confirmed in `en.json` and all existing components |
| lucide-vue-next | existing | Dashboard tile icons | `DashboardPage.vue` imports all tile icons from this package |
| Node.js `mongodb` driver | existing | Raw `db.collection()` query in index.js | The `s7plusAlarmEvents` collection uses native driver, not Mongoose |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| jsonwebtoken (authJwt.isAdmin) | existing | Protect new endpoint | Required — all admin endpoints use `[authJwt.isAdmin]` middleware |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Inline endpoint in index.js | Pass `db` to auth.routes/controller | Passing db adds a parameter to auth.routes function signature — more invasive, but more consistent with controller pattern. Inline is less invasive but adds to index.js length. |
| Inline endpoint in index.js | Create Mongoose model for s7plusAlarmEvents | Mongoose model requires defining a schema; the collection uses raw BsonDocument in C# — schema drift risk. Not recommended. |
| Client-side filtering | Server-side filtering with query params | Server-side would require re-fetch on filter change; client-side is simpler and sufficient for 200 records. Locked decision. |

**Installation:** No new packages required.

---

## Architecture Patterns

### Recommended Project Structure

```
AdminUI/src/
├── components/
│   └── S7PlusAlarmsViewerPage.vue   ← NEW: full viewer component
├── router/
│   └── index.js                      ← ADD: import + route entry
├── locales/
│   ├── en.json                        ← ADD: dashboard.s7plusAlarms key
│   └── [ar,de,es,fa,fr,it,ja,ps,pt,ru,uk,zh].json  ← MIRROR same key

server_realtime_auth/
└── index.js                           ← ADD: listS7PlusAlarms endpoint
```

### Pattern 1: Component Lifecycle (fetch + auto-refresh)

**What:** `onMounted` triggers initial fetch; `setInterval` repeats at 5000ms; `onUnmounted` clears interval.
**When to use:** Any viewer component needing periodic auto-refresh without user interaction.

```javascript
// Source: ProtocolConnectionsTab.vue (confirmed in codebase)
import { ref, computed, onMounted, onUnmounted } from 'vue'

const alarms = ref([])
const LIMIT = 200
let refreshTimer = null

onMounted(async () => {
  document.documentElement.style.overflowY = 'scroll'
  await fetchAlarms()
  refreshTimer = setInterval(fetchAlarms, 5000)
})

onUnmounted(() => {
  document.documentElement.style.overflowY = 'hidden'
  if (refreshTimer) clearInterval(refreshTimer)
})

const fetchAlarms = async () => {
  try {
    const response = await fetch('/Invoke/auth/listS7PlusAlarms')
    const json = await response.json()
    alarms.value = json
  } catch (err) {
    console.warn(err)
  }
}
```

### Pattern 2: Client-side Filtering with computed()

**What:** Filter state held in `ref`; filtered array is a `computed` derived from raw data.
**When to use:** When data is already fetched and filtering should not re-fetch.

```javascript
// Source: Pattern derived from existing AdminUI computed usage
const statusFilter = ref('All')          // 'All' | 'Incoming' | 'Outgoing'
const alarmClassFilter = ref('All')      // 'All' | any alarmClassName string

const alarmClassOptions = computed(() => {
  const classes = [...new Set(alarms.value.map(a => a.alarmClassName))]
  return ['All', ...classes.sort()]
})

const filteredAlarms = computed(() => {
  return alarms.value.filter(alarm => {
    const stateMatch = statusFilter.value === 'All'
      || (statusFilter.value === 'Incoming' && alarm.alarmState === 'Coming')
      || (statusFilter.value === 'Outgoing' && alarm.alarmState === 'Going')
    const classMatch = alarmClassFilter.value === 'All'
      || alarm.alarmClassName === alarmClassFilter.value
    return stateMatch && classMatch
  })
})
```

### Pattern 3: v-data-table with Custom Cell Templates

**What:** Vuetify 3 `v-data-table` with `#[item.colKey]` slot templates for status chip and ack icon.
**When to use:** Whenever a column needs a non-text render (chip, icon, custom formatting).

```html
<!-- Source: ProtocolConnectionsTab.vue confirmed template syntax -->
<v-data-table
  :headers="headers"
  :items="filteredAlarms"
  density="compact"
  class="mt-4 elevation-1"
  :items-per-page-text="$t('common.itemsPerPageText')"
>
  <template #[`item.alarmState`]="{ item }">
    <v-chip
      :color="item.alarmState === 'Coming' ? 'red' : 'green'"
      size="small"
    >{{ item.alarmState }}</v-chip>
  </template>

  <template #[`item.ackState`]="{ item }">
    <v-icon :color="item.ackState ? 'green' : 'red'">
      {{ item.ackState ? 'mdi-check' : 'mdi-close' }}
    </v-icon>
  </template>
</v-data-table>
```

### Pattern 4: Backend Endpoint Registration (inline in index.js)

**What:** Register a GET endpoint inside the `if (AUTHENTICATION)` block in `index.js`, after the auth.routes() call. Uses `db` (in closure) and `authJwt.isAdmin` middleware.
**When to use:** When a new endpoint needs the native MongoDB `db` handle that is not available in auth.controller.

```javascript
// Source: Pattern derived from existing app.use() + [authJwt.isAdmin] in index.js
app.use(
  OPCAPI_AP.replace('/Invoke/', '/Invoke/auth/listS7PlusAlarms'),  // = '/Invoke/auth/listS7PlusAlarms'
  [authJwt.isAdmin],
  async (req, res) => {
    try {
      if (!db) return res.status(200).send({ error: 'DB not connected' })
      const docs = await db
        .collection('s7plusAlarmEvents')
        .find({})
        .sort({ createdAt: -1 })
        .limit(200)
        .toArray()
      res.status(200).send(docs)
    } catch (err) {
      Log.log(err)
      res.status(200).send({ error: err.message })
    }
  }
)
```

Note: The cleaner literal path is `'/Invoke/auth/listS7PlusAlarms'` since `OPCAPI_AP = '/Invoke'` and the auth prefix is a convention, not a string constant.

### Pattern 5: Router Registration

**What:** Import new component + add route entry.
**When to use:** Every new top-level page in AdminUI.

```javascript
// Source: router/index.js confirmed pattern
import S7PlusAlarmsViewerPage from '../components/S7PlusAlarmsViewerPage.vue'

// In routes array:
{ path: '/s7plus-alarms', component: S7PlusAlarmsViewerPage }
```

### Pattern 6: Dashboard Tile Registration

**What:** Add entry to `shortcuts` ref array in `DashboardPage.vue`.
**When to use:** Any new page that should appear on the dashboard.

```javascript
// Source: DashboardPage.vue confirmed shortcuts array shape
// Icon must be imported from 'lucide-vue-next' at top of <script setup>
import { AlertTriangle } from 'lucide-vue-next'  // or another suitable icon

// In shortcuts array:
{
  titleKey: 'dashboard.s7plusAlarms',
  icon: AlertTriangle,   // lucide-vue-next icon component
  color: 'primary',
  route: '/s7plus-alarms',
  // No 'page' key — this is an internal Vue route, not an external URL
}
```

**Critical: No `page` or `target` key for internal routes.** The existing `alarmsViewer` entry has a `page` key because it opens an external iframe URL. The new S7Plus tile routes internally via Vue Router only — omit `page` to avoid the external-link button appearing.

### Pattern 7: i18n Key Addition

**What:** Add `dashboard.s7plusAlarms` key to all 13 locale files.
**When to use:** Required for any user-visible string in AdminUI.

```json
// Source: en.json confirmed structure
// Location: src/locales/en.json, in the "dashboard" object
"dashboard": {
  "s7plusAlarms": "S7Plus Alarms"
}
```

Mirror in all 13 locale files: `ar.json`, `de.json`, `es.json`, `fa.json`, `fr.json`, `it.json`, `ja.json`, `ps.json`, `pt.json`, `ru.json`, `uk.json`, `zh.json` — use the English string as fallback value in all non-English files (translators can update later).

### Pattern 8: Timestamp Splitting (Date / Time columns)

**What:** The MongoDB `timestamp` field is a BSON Date, serialized as ISO 8601 string in JSON responses. Split into Date and Time display columns.
**When to use:** Required for VIEW-02 column requirements.

```javascript
// Source: Claude's Discretion area — standard JavaScript Date parsing
const formatDate = (isoStr) => {
  if (!isoStr) return ''
  return new Date(isoStr).toLocaleDateString()
}
const formatTime = (isoStr) => {
  if (!isoStr) return ''
  return new Date(isoStr).toLocaleTimeString()
}

// In v-data-table headers:
{ title: 'Date', key: 'date', value: item => formatDate(item.timestamp) },
{ title: 'Time', key: 'time', value: item => formatTime(item.timestamp) },
```

### Anti-Patterns to Avoid

- **Modifying AlarmsViewerPage.vue:** Out of scope — locked. The existing page is an iframe wrapper and must stay exactly as is.
- **Using Mongoose to query s7plusAlarmEvents:** No Mongoose model exists for this collection; the C# driver writes raw BSON documents. Adding a Mongoose model risks schema drift and is unnecessary.
- **Re-fetching on filter change:** Client-side filtering is locked. Do not trigger new backend calls when `statusFilter` or `alarmClassFilter` changes.
- **Adding a `page` key to the dashboard shortcut:** The new tile is an internal SPA route. Adding a `page` key causes a redundant external-link button to appear in the card.
- **Hardcoding alarm class options:** The dropdown must be derived from the loaded dataset's distinct `alarmClassName` values — not a static list.
- **Not clearing setInterval in onUnmounted:** Will cause fetch calls after component is destroyed, leading to Vue warnings and potential memory leaks.

---

## MongoDB Document Schema (s7plusAlarmEvents)

Confirmed from `AlarmThread.cs` `BuildAlarmDocument()`:

| Field | Type | Description | Viewer Column |
|-------|------|-------------|---------------|
| `cpuAlarmId` | Long | PLC-assigned alarm ID | ID |
| `alarmState` | String | "Coming" or "Going" | Status (v-chip) |
| `alarmText` | String | Primary alarm text from TIA Portal | Event text |
| `infoText` | String | Info text from TIA Portal | (not in VIEW-02 columns) |
| `additionalTexts` | Array[9] | Index 0=AdditionalText1…8 | Additional text 1–3 (indices 0,1,2) |
| `associatedValues` | Array[10] | SD_1…SD_10 typed values | (not in VIEW-02 columns) |
| `timestamp` | Date | PLC-side event timestamp | Date + Time columns |
| `ackState` | Boolean | true=acknowledged, false=unacknowledged | Acknowledge (icon) |
| `connectionId` | Int | protocolConnectionNumber | Source column |
| `createdAt` | Date | Server insertion time (used for sort) | (internal — sort field) |
| `priority` | Int | HmiInfo.Priority | (not in VIEW-02 columns) |
| `alarmClass` | Int | Numeric alarm class ID | (not displayed; used for className lookup) |
| `alarmClassName` | String | Human-readable class name (e.g., "Acknowledgment required", "Unknown (33)") | Alarm class name |
| `groupId` | Int | HmiInfo.GroupId | (not in VIEW-02 columns) |
| `allStatesInfo` | Int | AllStatesInfo bitmask | (not in VIEW-02 columns) |

**Column mapping for VIEW-02:**
- Source → `connectionId` (the protocolConnectionNumber integer)
- Date → `timestamp` (date portion)
- Time → `timestamp` (time portion)
- Status → `alarmState` ("Coming"/"Going") rendered as v-chip
- Acknowledge → `ackState` (boolean) rendered as mdi-check/mdi-close icon
- Alarm class name → `alarmClassName`
- Event text → `alarmText`
- ID → `cpuAlarmId`
- Additional text 1 → `additionalTexts[0]`
- Additional text 2 → `additionalTexts[1]`
- Additional text 3 → `additionalTexts[2]`

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Table with sort/pagination | Custom HTML table | `v-data-table` (Vuetify 3) | Built-in sort, pagination, density, slot templates — already used in ProtocolConnectionsTab.vue, TagsTab.vue |
| Dropdown filter | Custom select element | `v-select` (Vuetify 3) | Consistent with AdminUI conventions; confirmed used throughout |
| Status colored badge | Custom CSS chip | `v-chip` with `:color` | Available and used in AdminUI; correct semantic component |
| Boolean icon column | Custom icon logic | `v-icon` with mdi-check/mdi-close | Exact pattern already used in `ProtocolConnectionsTab.vue` enabled column |
| Auth middleware | Custom token parsing | `[authJwt.isAdmin]` | Existing middleware handles JWT verification consistently |
| Route protection | Custom guards | Hash-history router + existing auth cookie | AdminUI already handles auth state via cookie + router |

---

## Common Pitfalls

### Pitfall 1: Native db Handle Not Available in auth.controller

**What goes wrong:** Developer follows the instruction to add `listS7PlusAlarms` to `auth.controller.js` and uses `db.collection('s7plusAlarmEvents')` — but `db` is not imported or available in auth.controller. It will throw `ReferenceError: db is not defined`.

**Why it happens:** auth.controller uses Mongoose models via `require('../models')`. The native MongoDB `db` handle is a local variable in `index.js`'s async IIFE closure. It is not exported, not attached to `app`, and not passed to auth.routes.

**How to avoid:** Register the endpoint inline in `index.js` inside the `if (AUTHENTICATION)` block after the `auth.routes()` call, where `db` is in scope. Alternatively, pass `db` as a third argument to `require('./app/routes/auth.routes')(app, OPCAPI_AP, db)` and thread it through to the controller.

**Warning signs:** Any attempt to `require` db in auth.controller or reference it without import.

### Pitfall 2: Missing clearInterval in onUnmounted

**What goes wrong:** The `setInterval` keeps firing after the component is unmounted (user navigates away). Causes Vue reactive system warnings ("component is already unmounted") and unnecessary network requests.

**Why it happens:** Developers add `onMounted` with `setInterval` but forget the paired `onUnmounted` cleanup.

**How to avoid:** Always capture the timer ID and clear it:
```javascript
let refreshTimer = null
onMounted(() => { refreshTimer = setInterval(fetchAlarms, 5000) })
onUnmounted(() => { if (refreshTimer) clearInterval(refreshTimer) })
```

### Pitfall 3: Dashboard Tile Gets Unintended External Link Button

**What goes wrong:** The S7Plus Alarms tile shows a small external-link icon button in the card footer, navigating to an undefined or wrong URL.

**Why it happens:** The `shortcuts` array entries with a `page` property render a `v-btn href={page}` button. Existing tiles like `alarmsViewer` have `page: '/tabular.html?...'` because they open external HTML apps. The new tile should NOT have a `page` key.

**How to avoid:** Omit the `page` key entirely from the shortcut entry. The card click handler falls through to `navigateTo(shortcut.route)`.

### Pitfall 4: additionalTexts Array Indexing

**What goes wrong:** Developer displays `item.additionalTexts[1]`, `item.additionalTexts[2]`, `item.additionalTexts[3]` (1-based) instead of `[0]`, `[1]`, `[2]` (0-based).

**Why it happens:** TIA Portal labels them "Additional text 1", "Additional text 2", "Additional text 3" — natural 1-based naming. The array in MongoDB is 0-based JavaScript.

**How to avoid:** Confirm from `BuildAlarmDocument` in `AlarmThread.cs`: `additionalTexts` BsonArray index 0 = AdditionalText1, index 1 = AdditionalText2, index 2 = AdditionalText3.

### Pitfall 5: Alarm Class Filter Shows Empty String Options

**What goes wrong:** Dropdown shows blank options because some alarms have empty `alarmClassName` strings.

**Why it happens:** Early alarm documents may lack `alarmClassName` or have empty strings.

**How to avoid:** Filter out falsy values when building the options list:
```javascript
const alarmClassOptions = computed(() => {
  const classes = [...new Set(
    alarms.value.map(a => a.alarmClassName).filter(Boolean)
  )]
  return ['All', ...classes.sort()]
})
```

### Pitfall 6: Hash Router — Route Must Be in routes Array

**What goes wrong:** Navigating to `/#/s7plus-alarms` shows the default/404 view.

**Why it happens:** AdminUI uses `createWebHashHistory()`. Routes are matched by hash segment. The route must be explicitly added to the `routes` array in `router/index.js`.

**How to avoid:** Follow the exact import + route entry pattern from existing routes. No lazy loading needed — all existing routes use eager imports.

### Pitfall 7: i18n Key Missing in Non-English Locales

**What goes wrong:** Dashboard tile shows the raw key string `dashboard.s7plusAlarms` in non-English locales.

**Why it happens:** Only `en.json` is updated; the other 12 locale files are not updated.

**How to avoid:** Add the key to all 13 locale files. Use English as fallback text in non-English files.

---

## Code Examples

### Verified Backend Pattern — listProtocolConnections (reference)

```javascript
// Source: auth.controller.js:892 (confirmed in codebase)
exports.listProtocolConnections = async (req, res) => {
  Log.log('listProtocolConnections')
  try {
    let protocolConnections = await ProtocolConnection.find({}).exec()
    res.status(200).send(protocolConnections)
  } catch (err) {
    Log.log(err)
    res.status(200).send({ error: err })
  }
}
```

### Verified Frontend Fetch Pattern — fetchProtocolConnections (reference)

```javascript
// Source: ProtocolConnectionsTab.vue:3813 (confirmed in codebase)
const fetchProtocolConnections = async () => {
  try {
    const response = await fetch('/Invoke/auth/listProtocolConnections')
    const json = await response.json()
    for (let i = 0; i < json.length; i++) {
      json[i].id = i + 1
    }
    protocolConnections.value = json
  } catch (err) {
    console.warn(err)
  }
}
```

### Verified Boolean Icon Pattern (reference)

```html
<!-- Source: ProtocolConnectionsTab.vue:43-46 (confirmed in codebase) -->
<template #[`item.enabled`]="{ item }">
  <v-icon v-if="item.enabled" color="green">mdi-check</v-icon>
  <v-icon v-else color="red">mdi-close</v-icon>
</template>
```

### Verified Auth Route Registration Pattern (reference)

```javascript
// Source: auth.routes.js:56-60 (confirmed in codebase)
app.use(
  accessPoint + 'auth/listProtocolConnections',
  [authJwt.isAdmin],
  controller.listProtocolConnections
)
```

### Verified Dashboard Shortcut Shape (reference)

```javascript
// Source: DashboardPage.vue:78-85 (confirmed in codebase)
{
  titleKey: 'dashboard.alarmsViewer',
  icon: Bell,
  color: 'primary',
  route: '/alarms-viewer',
  page: '/tabular.html?SELMODULO=ALARMS_VIEWER',
  target: '_blank',
},
// New entry shape (no page/target for internal route):
{
  titleKey: 'dashboard.s7plusAlarms',
  icon: AlertTriangle,   // example — planner chooses appropriate lucide icon
  color: 'primary',
  route: '/s7plus-alarms',
}
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Options API (`data()`, `methods:`) | Composition API (`<script setup>`, `ref`, `computed`) | Vue 3 (already in project) | All new components use `<script setup>` — confirmed by DashboardPage.vue, ProtocolConnectionsTab.vue |
| Vuetify 2 `v-data-table` slot syntax | Vuetify 3 `#[item.key]` dynamic slot names | Vuetify 3 migration | Confirmed in ProtocolConnectionsTab.vue — use `#[\`item.colKey\`]` syntax |

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | None configured (AdminUI: no vitest/jest; server_realtime_auth: no test runner) |
| Config file | None — see Wave 0 |
| Quick run command | N/A — no automated test infrastructure |
| Full suite command | N/A — no automated test infrastructure |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| VIEW-01 | Navigate to `/s7plus-alarms` → S7Plus viewer loads; `/alarms-viewer` unchanged | smoke (manual) | Manual browser check | N/A — manual only |
| VIEW-02 | Alarm table shows all 9 required columns with correct data | smoke (manual) | Manual browser check | N/A — manual only |
| VIEW-04 | New alarm appears within one 5-second cycle without reload | smoke (manual) | Manual: trigger PLC alarm, wait 5s | N/A — manual only |
| VIEW-05 | Status filter shows only matching rows | smoke (manual) | Manual: set filter, verify rows | N/A — manual only |
| VIEW-06 | Alarm class filter populates from data, filters correctly | smoke (manual) | Manual: set filter, verify rows | N/A — manual only |

### Sampling Rate

- **Per task commit:** Manual browser smoke test against running AdminUI
- **Per wave merge:** Full manual acceptance checklist (all 5 requirements above)
- **Phase gate:** All manual checks pass before `/gsd:verify-work`

### Wave 0 Gaps

No automated test infrastructure exists for AdminUI or server_realtime_auth. All phase requirements are verified manually against a running stack. No Wave 0 test file creation needed — the verification approach is human-driven smoke testing.

---

## Open Questions

1. **Which lucide-vue-next icon to use for the S7Plus Alarms dashboard tile**
   - What we know: `DashboardPage.vue` imports `Bell` for the existing Alarms Viewer
   - What's unclear: The planner should pick an appropriate icon; `AlertTriangle` or `BellRing` are reasonable choices from lucide-vue-next
   - Recommendation: Planner decides; both are available in lucide-vue-next

2. **Whether the new backend endpoint should use `app.use` or `app.get`**
   - What we know: Existing auth endpoints mix `app.use()` and `app.get()`. `app.use()` matches any HTTP method; `app.get()` restricts to GET
   - What's unclear: Which is more consistent for a read-only list endpoint
   - Recommendation: Use `app.use()` to match the `listProtocolConnections` pattern exactly; this is a list endpoint and `app.use()` is the dominant pattern for list routes in auth.routes.js

3. **MongoDB `_id` field serialization in JSON response**
   - What we know: `toArray()` returns documents with BSON `_id` (ObjectId). The MongoDB Node.js driver serializes ObjectId as `{ $oid: "..." }` in some versions or as a string
   - What's unclear: Whether the frontend needs to handle the `_id` field specially
   - Recommendation: Add `_id: 0` projection to exclude it, or ignore it in the frontend. Safe to project it out since no field in VIEW-02 columns requires `_id`.

---

## Sources

### Primary (HIGH confidence)

- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — complete `BuildAlarmDocument()` schema, all field names and types confirmed
- `json-scada/src/AdminUI/src/router/index.js` — route registration pattern confirmed
- `json-scada/src/AdminUI/src/components/DashboardPage.vue` — shortcuts array shape, lucide icon import pattern confirmed
- `json-scada/src/server_realtime_auth/app/routes/auth.routes.js` — route registration pattern, `[authJwt.isAdmin]` middleware usage confirmed
- `json-scada/src/server_realtime_auth/index.js` — native `db` scope confirmed (local variable, not exported), inline route registration pattern confirmed
- `json-scada/src/AdminUI/src/components/ProtocolConnectionsTab.vue` — fetch pattern, boolean icon template, v-data-table slot syntax confirmed
- `json-scada/src/AdminUI/src/locales/en.json` — dashboard key structure and 13 locale files confirmed
- `json-scada/src/AdminUI/src/components/AlarmsViewerPage.vue` — iframe wrapper pattern confirmed; must not be modified

### Secondary (MEDIUM confidence)

- Vuetify 3.10 `v-data-table` slot syntax — confirmed from existing component usage in codebase (not from external docs, from direct code inspection)

---

## Metadata

**Confidence breakdown:**
- MongoDB document schema: HIGH — directly read from `AlarmThread.cs` `BuildAlarmDocument()`
- Backend endpoint approach (inline in index.js): HIGH — confirmed by tracing `db` variable scope in index.js
- Frontend component patterns: HIGH — all patterns confirmed from existing AdminUI component code
- Vuetify component API: HIGH — confirmed from existing usage in ProtocolConnectionsTab.vue
- i18n: HIGH — structure confirmed from en.json; 13 locale files confirmed by directory listing
- Pitfalls: HIGH — each pitfall derived from direct code inspection (not speculation)

**Research date:** 2026-03-18
**Valid until:** 2026-04-18 (stable stack — no fast-moving dependencies)

# Phase 14: DatablockBrowser - Research

**Researched:** 2026-03-26
**Domain:** Vue 3 + Vuetify 3 frontend page — AdminUI component, routing, i18n, dashboard integration
**Confidence:** HIGH

## Summary

Phase 14 is a pure frontend addition: a new `DatablockBrowserPage.vue` Vue 3 component added to the AdminUI. All backend API endpoints needed by this phase were completed in Phase 13. The implementation pattern is fully defined by `S7PlusAlarmsViewerPage.vue` — the data table layout, filter controls, fetch pattern, and per-row action buttons are all reproduced from that reference.

The open question from CONTEXT.md (how to populate the connection selector) is resolved: `GET /Invoke/auth/listProtocolConnections` exists and is already used by `ProtocolConnectionsTab.vue`. The response includes `protocolConnectionNumber` and `name` fields, which can populate a dropdown mapping human-readable names to connection numbers. This is the preferred approach over deriving connection numbers from datablocks — it avoids the chicken-and-egg problem of requiring a fetch before knowing valid selections.

The router uses `createWebHashHistory`, so new-tab URLs must be constructed as `/#/s7plus-tag-tree?...` rather than `/s7plus-tag-tree?...`. The decisions in CONTEXT.md already correctly reflect this — `window.open` with a hash-based URL works without any router involvement.

**Primary recommendation:** Copy the S7PlusAlarmsViewerPage structure exactly. Fetch connection list from `listProtocolConnections` on mount for the dropdown. Fetch datablocks from `listS7PlusDatablocks?connectionNumber=N` when dropdown selection changes. Use `window.open('/#/s7plus-tag-tree?db=...&connectionNumber=N', '_blank')` per D-04.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Start with placeholder/empty state — no connection pre-selected on load. Operator must explicitly choose a connection before datablocks are fetched.
- **D-02:** TagTreeBrowserPage route path is `/s7plus-tag-tree`.
- **D-03:** Query params use `encodeURIComponent(db_name)` for `db`: `/#/s7plus-tag-tree?db=${encodeURIComponent(row.db_name)}&connectionNumber=${connectionNumber}`.
- **D-04:** Open in new browser tab via `window.open(url, '_blank')`.
- **D-05:** Explicit "Browse Tags" button in a dedicated Actions column per row — no whole-row click handler.
- **D-06:** Follow S7PlusAlarmsViewerPage pattern throughout: `v-data-table` with `density="compact"` and `class="elevation-1"`, `v-select` with `density="compact"` for connection filter, `fetch('/Invoke/auth/listS7PlusDatablocks?connectionNumber=N')`, no auto-refresh, cookie-based auth.

### Claude's Discretion
- How to populate connection dropdown options (researcher to investigate `listProtocolConnections` endpoint).

### Deferred Ideas (OUT OF SCOPE)
- None — discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DBBROWSER-01 | Operator can see all datablocks on a connected PLC (showing `db_name` and `db_number`) via a new `DatablockBrowserPage` added to the AdminUI navigation menu | Backend API exists (`listS7PlusDatablocks`); DashboardPage shortcut pattern confirmed; router pattern confirmed |
| DBBROWSER-02 | Operator can filter the datablock list to a specific PLC using a connection selector dropdown | `listProtocolConnections` endpoint exists and returns `protocolConnectionNumber` + `name`; pattern in ProtocolConnectionsTab confirmed |
| DBBROWSER-03 | Operator can open `TagTreeBrowserPage` for a specific datablock by clicking a row; tag browser opens in new browser tab | `window.open` with hash URL pattern confirmed; Vue Router uses `createWebHashHistory` so `/#/s7plus-tag-tree` is the correct URL form |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Vue 3 | ^3.4.31 | Composition API, `<script setup>` SFCs | Project standard; all existing pages use it |
| Vuetify | 3.10 | `v-data-table`, `v-select`, `v-btn`, `v-container` | Project standard; locked at 3.10 in package.json |
| vue-router | ^4.4.0 | `createWebHashHistory` routing (not used directly in this component — `window.open` used instead) | Project standard |
| vue-i18n | ^11.1.2 | `$t()` for all user-facing strings | Project standard |
| lucide-vue-next | ^0.441.0 | Dashboard shortcut icon (`Database` icon) | Used by DashboardPage; `Database` icon already imported there |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| @mdi/font | 7.4.47 | MDI icon strings (e.g., `mdi-database`) | Used in v-icon within Vuetify components — not needed for lucide |

**No additional dependencies to install.** All libraries are already present in `package.json`.

## Architecture Patterns

### Project Structure
```
json-scada/src/AdminUI/src/
├── components/
│   ├── DatablockBrowserPage.vue   ← NEW (follow S7PlusAlarmsViewerPage pattern)
│   └── S7PlusAlarmsViewerPage.vue ← MODEL
├── router/
│   └── index.js                   ← ADD import + route entry
├── locales/
│   └── en.json                    ← ADD "datablockBrowser" key under "dashboard"
└── components/
    └── DashboardPage.vue          ← ADD shortcut entry
```

### Pattern 1: Connection Dropdown Population

**What:** On `onMounted`, fetch `GET /Invoke/auth/listProtocolConnections` to build the connection selector options. The endpoint returns an array of objects with `protocolConnectionNumber` (numeric) and `name` (string). Build `v-select` items as `{ title: conn.name, value: conn.protocolConnectionNumber }`.

**When to use:** Always on mount — this is the correct approach, not deriving connections from datablock responses.

**Example:**
```javascript
// Source: ProtocolConnectionsTab.vue line 3813, auth.controller.js line 892
const connectionOptions = ref([])
const selectedConnection = ref(null)  // null = placeholder/empty state (D-01)

onMounted(async () => {
  document.documentElement.style.overflowY = 'scroll'
  try {
    const res = await fetch('/Invoke/auth/listProtocolConnections')
    const json = await res.json()
    if (Array.isArray(json)) {
      connectionOptions.value = json.map(c => ({
        title: c.name,
        value: c.protocolConnectionNumber
      }))
    }
  } catch (err) {
    console.warn('Failed to fetch connections:', err)
  }
})
```

### Pattern 2: Datablock Fetch on Filter Change

**What:** Watch `selectedConnection` ref. When it changes to a non-null value, fetch `listS7PlusDatablocks?connectionNumber=N`. Reset datablocks to `[]` when connection is cleared.

**Example:**
```javascript
// Source: adapted from S7PlusAlarmsViewerPage.vue fetch pattern
import { ref, watch, onMounted, onUnmounted } from 'vue'

const datablocks = ref([])
const selectedConnection = ref(null)

const fetchDatablocks = async (connectionNumber) => {
  try {
    const res = await fetch(
      `/Invoke/auth/listS7PlusDatablocks?connectionNumber=${connectionNumber}`
    )
    const json = await res.json()
    if (Array.isArray(json)) {
      datablocks.value = json
    }
  } catch (err) {
    console.warn('Failed to fetch datablocks:', err)
  }
}

watch(selectedConnection, (newVal) => {
  if (newVal !== null && newVal !== undefined) {
    fetchDatablocks(newVal)
  } else {
    datablocks.value = []
  }
})
```

### Pattern 3: New-Tab Navigation URL Construction

**What:** Hash-history router means all SPA routes live under `/#/path`. `window.open` must use the full hash URL.

**Why it matters:** If the URL is constructed as `/s7plus-tag-tree?...` (without `#`), the browser navigates to a non-SPA path, the server serves the HTML shell, and Vue Router routes to `/login` (the catch-all redirect). The hash prefix is mandatory.

**Example:**
```javascript
// Source: CONTEXT.md D-03; router/index.js createWebHashHistory confirmed
const browseDatablock = (row) => {
  const url = `/#/s7plus-tag-tree?db=${encodeURIComponent(row.db_name)}&connectionNumber=${selectedConnection.value}`
  window.open(url, '_blank')
}
```

### Pattern 4: Router Registration

**What:** Add import and route to `router/index.js` following the exact existing pattern.

**Example:**
```javascript
// Source: router/index.js — matches all existing page registrations
import DatablockBrowserPage from '../components/DatablockBrowserPage.vue'

// Inside routes array:
{ path: '/s7plus-datablocks', component: DatablockBrowserPage },
```

### Pattern 5: Dashboard Shortcut Registration

**What:** Add one entry to the `shortcuts` ref array in `DashboardPage.vue`. The `Database` icon is already imported (used for Metabase entry). Use `route` property (not `page`) so clicking navigates within the SPA.

**Example:**
```javascript
// Source: DashboardPage.vue — confirmed Database icon import at line 53
{
  titleKey: 'dashboard.datablockBrowser',
  icon: Database,
  color: 'primary',
  route: '/s7plus-datablocks',
},
```

### Pattern 6: i18n Key Addition

**What:** Add exactly one key to `en.json` under `dashboard`. The pattern is flat camelCase — `"datablockBrowser": "Datablock Browser"`.

**Note on other locales:** Only `en.json` is strictly required. If other locale files exist and are missing the key, Vue-i18n falls back to English silently — not a runtime error. The planner may optionally add to other locale files but it is not a blocker.

### Anti-Patterns to Avoid
- **Whole-row click handler for navigation:** D-05 is explicit — use a dedicated "Browse Tags" button per row.
- **Auto-refresh timer:** Unlike S7PlusAlarmsViewerPage, datablocks are static. No `setInterval` needed.
- **Pre-selecting a connection on load:** D-01 prohibits this. `selectedConnection` must start as `null`.
- **Using `/s7plus-tag-tree?...` without `/#` prefix:** The router uses `createWebHashHistory`. Missing the hash causes navigation outside the SPA.
- **Double-encoding db_name:** CONTEXT.md D-03 notes Vue Router automatically decodes `route.query.db` — only one `encodeURIComponent` call in Phase 14, no decode needed here.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Connection list population | Custom fetch to datablocks + distinct() | `GET /Invoke/auth/listProtocolConnections` | Endpoint already exists; returns name + number together; used by ProtocolConnectionsTab.vue already |
| i18n string lookup | Direct string literals | `$t('dashboard.datablockBrowser')` | All user-facing strings go through vue-i18n per project convention |
| Table with sorting/pagination | Custom table | `v-data-table` from Vuetify | Already in use; `density="compact"` + `class="elevation-1"` is the project standard |

**Key insight:** Every infrastructure concern in this phase is already solved — the only new code is wiring together existing patterns.

## Connection Selector: Open Question Resolved

The CONTEXT.md flagged "how to populate connection selector options" as an open question for the researcher. The answer:

**Use `GET /Invoke/auth/listProtocolConnections`.**

Evidence:
- `auth.routes.js` line 58: endpoint is registered and admin-guarded
- `auth.controller.js` line 892–901: returns `ProtocolConnection.find({})` — all connections with `protocolConnectionNumber` and `name` fields
- `ProtocolConnectionsTab.vue` line 3813–3815: existing frontend fetch call pattern confirmed
- `protocolConnection.model.js` lines 13–21: model confirms `protocolConnectionNumber` (Double) and `name` (String) fields

This is strictly better than option (a) from CONTEXT.md (calling `listS7PlusDatablocks` with no filter) because:
1. `listS7PlusDatablocks` **requires** `connectionNumber` — it returns 400 if the param is missing or NaN
2. Even if option (a) worked, it would only show connections that have datablocks stored — newly configured connections wouldn't appear until the driver runs

## Common Pitfalls

### Pitfall 1: Hash-History URL Missing `/#` Prefix
**What goes wrong:** `window.open('/s7plus-tag-tree?db=X&connectionNumber=1', '_blank')` opens a tab that gets served by the backend as an API 404 or default page, not the SPA.
**Why it happens:** `createWebHashHistory` routes all SPA navigation through `index.html#/path`. The hash prefix tells the browser this is a client-side route.
**How to avoid:** Always construct new-tab URLs as `/#/s7plus-tag-tree?...`.
**Warning signs:** New tab shows login page or blank page instead of TagTreeBrowserPage.

### Pitfall 2: Calling `listS7PlusDatablocks` Without a `connectionNumber`
**What goes wrong:** The endpoint returns HTTP 400 `{ error: 'Missing or invalid connectionNumber query parameter' }`.
**Why it happens:** The endpoint does `parseInt(req.query.connectionNumber, 10)` and checks `isNaN`.
**How to avoid:** Guard the fetch call — only call when `selectedConnection.value` is not null. Confirmed by `watch(selectedConnection, ...)` pattern above.
**Warning signs:** Console shows 400 errors on page load or when dropdown is cleared.

### Pitfall 3: `listProtocolConnections` Returns All Connections, Including Non-S7Plus
**What goes wrong:** The dropdown shows IEC 104, OPC-UA, etc. connections that have no S7Plus datablocks.
**Why it happens:** `listProtocolConnections` is not filtered by driver type.
**How to avoid:** This is acceptable for the PoC scope. If the operator selects a non-S7Plus connection, `listS7PlusDatablocks` will return an empty array — the table shows "No data" which is the correct empty state. A driver-type filter is listed as a future requirement.
**Warning signs:** None — the empty table is correct UX.

### Pitfall 4: `Database` Icon Name Collision with Metabase Entry
**What goes wrong:** Using `Database` icon for DatablockBrowser while Metabase also uses `Database` creates visual ambiguity, not a code error.
**Why it happens:** Both features are "database-related." CONTEXT.md explicitly noted `Database` icon is "already imported" and available.
**How to avoid:** The planner should use `Database` as specified — DashboardPage already imports it. If visual differentiation is desired, that is a future concern outside this phase's scope.

### Pitfall 5: `v-select` `items` Prop Shape
**What goes wrong:** Passing raw connection number integers to `v-select :items` shows numbers, not names.
**Why it happens:** `v-select` with scalar items shows the value as the label.
**How to avoid:** Use `{ title: conn.name, value: conn.protocolConnectionNumber }` object format. Vuetify 3 `v-select` uses `item-title` and `item-value` props (defaults to `title`/`value`) when items are objects.

## Code Examples

### DatablockBrowserPage.vue — Complete Structure Sketch
```vue
<!-- Source: Modeled on S7PlusAlarmsViewerPage.vue; decisions from 14-CONTEXT.md -->
<template>
  <v-container fluid>
    <h2 class="mb-4">Datablock Browser</h2>
    <v-row class="mb-2">
      <v-col cols="12" sm="4" md="3">
        <v-select
          label="Connection"
          :items="connectionOptions"
          item-title="title"
          item-value="value"
          v-model="selectedConnection"
          density="compact"
          placeholder="Select a connection..."
          clearable
        />
      </v-col>
    </v-row>
    <v-data-table
      :headers="headers"
      :items="datablocks"
      density="compact"
      class="elevation-1"
      :items-per-page="50"
    >
      <template #[`item.actions`]="{ item }">
        <v-btn size="x-small" variant="tonal" @click="browseDatablock(item)">
          Browse Tags
        </v-btn>
      </template>
    </v-data-table>
  </v-container>
</template>

<script setup>
import { ref, watch, onMounted, onUnmounted } from 'vue'

const connectionOptions = ref([])
const selectedConnection = ref(null)
const datablocks = ref([])

const headers = [
  { title: 'DB Name', key: 'db_name', sortable: true },
  { title: 'DB Number', key: 'db_number', sortable: true },
  { title: 'Actions', key: 'actions', sortable: false },
]

onMounted(async () => {
  document.documentElement.style.overflowY = 'scroll'
  try {
    const res = await fetch('/Invoke/auth/listProtocolConnections')
    const json = await res.json()
    if (Array.isArray(json)) {
      connectionOptions.value = json.map(c => ({
        title: c.name,
        value: c.protocolConnectionNumber,
      }))
    }
  } catch (err) {
    console.warn('Failed to fetch connections:', err)
  }
})

onUnmounted(() => {
  document.documentElement.style.overflowY = 'hidden'
})

watch(selectedConnection, async (newVal) => {
  if (newVal !== null && newVal !== undefined) {
    try {
      const res = await fetch(
        `/Invoke/auth/listS7PlusDatablocks?connectionNumber=${newVal}`
      )
      const json = await res.json()
      datablocks.value = Array.isArray(json) ? json : []
    } catch (err) {
      console.warn('Failed to fetch datablocks:', err)
      datablocks.value = []
    }
  } else {
    datablocks.value = []
  }
})

const browseDatablock = (row) => {
  const url = `/#/s7plus-tag-tree?db=${encodeURIComponent(row.db_name)}&connectionNumber=${selectedConnection.value}`
  window.open(url, '_blank')
}
</script>
```

### router/index.js Addition
```javascript
// Source: router/index.js — follows existing import + route entry pattern
import DatablockBrowserPage from '../components/DatablockBrowserPage.vue'

// In routes array (after s7plus-alarms entry):
{ path: '/s7plus-datablocks', component: DatablockBrowserPage },
```

### en.json Addition
```json
// Source: locales/en.json — under "dashboard" key after "s7plusAlarms" entry
"datablockBrowser": "Datablock Browser"
```

### DashboardPage.vue Shortcut Addition
```javascript
// Source: DashboardPage.vue shortcuts array — after s7plusAlarms entry
{
  titleKey: 'dashboard.datablockBrowser',
  icon: Database,
  color: 'primary',
  route: '/s7plus-datablocks',
},
```

## Environment Availability

Step 2.6: SKIPPED — Phase 14 is code-only changes (new Vue component + router/i18n/dashboard wiring). No external dependencies beyond the already-installed AdminUI node_modules.

## Validation Architecture

nyquist_validation is enabled (config.json has `"nyquist_validation": true`).

### Test Framework
| Property | Value |
|----------|-------|
| Framework | None — no test framework installed in AdminUI |
| Config file | None |
| Quick run command | `npm run build` (build verification only) |
| Full suite command | `npm run lint` then `npm run build` |

No test framework (Jest, Vitest, Playwright) is installed in `json-scada/src/AdminUI`. The `package.json` has no `test` script and no test dependencies. All existing phases verify correctness by manual browser testing + build success.

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DBBROWSER-01 | DatablockBrowserPage appears in nav/dashboard and renders table | manual | `npm run build` (smoke) | ❌ No test framework |
| DBBROWSER-02 | Connection dropdown populated from listProtocolConnections; selecting connection fetches datablocks | manual | `npm run build` (smoke) | ❌ No test framework |
| DBBROWSER-03 | "Browse Tags" button opens new tab at correct hash URL | manual | `npm run build` (smoke) | ❌ No test framework |

### Sampling Rate
- **Per task commit:** `cd json-scada/src/AdminUI && npm run lint`
- **Per wave merge:** `cd json-scada/src/AdminUI && npm run build`
- **Phase gate:** Build succeeds (zero errors) + manual browser verification of all three requirements before `/gsd:verify-work`

### Wave 0 Gaps
None — no test infrastructure is needed for this phase. The project has no automated test suite for AdminUI and all previous phases verified manually. Build success (`npm run build`) is the automated gate.

## Sources

### Primary (HIGH confidence)
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — complete implementation model, fetch pattern, v-data-table pattern, per-row button pattern
- `json-scada/src/AdminUI/src/components/DashboardPage.vue` — shortcut registration pattern, Database icon already imported
- `json-scada/src/AdminUI/src/router/index.js` — route registration pattern, `createWebHashHistory` confirmed
- `json-scada/src/AdminUI/src/locales/en.json` — i18n key format under `dashboard`
- `json-scada/src/server_realtime_auth/index.js` lines 463–484 — `listS7PlusDatablocks` endpoint: requires `connectionNumber`, returns array sorted by `db_name`
- `json-scada/src/server_realtime_auth/app/routes/auth.routes.js` line 58 — `listProtocolConnections` endpoint registered
- `json-scada/src/server_realtime_auth/app/controllers/auth.controller.js` lines 892–901 — `listProtocolConnections` returns all connections
- `json-scada/src/server_realtime_auth/app/models/protocolConnection.model.js` — `protocolConnectionNumber` and `name` field shapes
- `json-scada/src/AdminUI/package.json` — confirmed no test framework, Vuetify 3.10, lucide-vue-next present

### Secondary (MEDIUM confidence)
- `json-scada/src/AdminUI/src/components/ProtocolConnectionsTab.vue` line 3813 — confirms `listProtocolConnections` fetch pattern from frontend

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libraries confirmed in package.json, all versions verified from source
- Architecture: HIGH — all patterns sourced directly from existing project files
- Connection selector answer: HIGH — endpoint existence and field shapes confirmed from three source files
- Hash-URL pitfall: HIGH — `createWebHashHistory` confirmed in router/index.js
- Pitfalls: HIGH — sourced from reading actual endpoint implementation

**Research date:** 2026-03-26
**Valid until:** 2026-04-25 (stable — no external dependencies, all findings from local project files)

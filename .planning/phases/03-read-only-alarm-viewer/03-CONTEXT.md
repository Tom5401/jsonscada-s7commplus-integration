# Phase 3: Read-Only Alarm Viewer - Context

**Gathered:** 2026-03-18
**Status:** Ready for planning

<domain>
## Phase Boundary

A dedicated S7Plus Alarms Viewer page in AdminUI (Vue 3 + Vuetify) that queries the `s7plusAlarmEvents` MongoDB collection, displays TIA Portal-equivalent columns, auto-refreshes every 5 seconds, and supports filtering by Status and alarm class. Fully independent of the existing `AlarmsViewerPage.vue` — the existing page must not be modified. Acknowledge capability is out of scope for this phase (Phase 4).

</domain>

<decisions>
## Implementation Decisions

### Navigation
- Add a Dashboard tile in `DashboardPage.vue` for the new viewer alongside the existing "Alarms Viewer" tile
- Label: **"S7Plus Alarms"** (i18n key: `dashboard.s7plusAlarms`)
- Route: `/s7plus-alarms` (as specified in roadmap)
- New component: `S7PlusAlarmsViewerPage.vue`

### Alarm list scope
- Load the **last 200 most recent** alarm events (sort by `createdAt` descending, limit 200)
- Default sort: **newest first** in the displayed table
- 200 is a compile-time constant in the component — no configuration UI needed

### Filter layout
- Filters in a **toolbar row above the table** (`v-row` or `v-toolbar`-style layout)
- Both Status and Alarm class filters use **`v-select` dropdowns** (consistent with AdminUI conventions)
- Status filter options: All / Incoming / Outgoing
- Alarm class filter: populated from the distinct `alarmClassName` values in the currently loaded dataset (dynamic, not hardcoded)
- Filtering is **client-side** (filter the already-fetched 200 records in-component) — no re-fetch on filter change

### Status column presentation
- Render as **colored `v-chip`**: red chip for "Coming" (Incoming), green chip for "Going" (Outgoing)
- Chip text mirrors the MongoDB `alarmState` value: "Coming" / "Going"

### Acknowledge column presentation
- Render as **icon**: `mdi-check` in green for acknowledged (`ackState: true`), `mdi-close` in red for unacknowledged (`ackState: false`)
- Consistent with how AdminUI boolean fields are displayed in other admin tables

### Auto-refresh
- `setInterval` at 5000ms, cleared in `onUnmounted`
- Fetches the same query on each tick — full reload of the 200-record window

### Claude's Discretion
- Exact backend endpoint path and implementation (new route in `server_realtime_auth` or direct MongoDB via existing `/Invoke/auth/` pattern)
- Whether to require auth token for the new endpoint (follow existing auth patterns)
- Exact i18n key names beyond the Dashboard label
- Column widths and `v-data-table` density setting
- How Date and Time are split from the `timestamp` field (ISO string parsing)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements
- `.planning/REQUIREMENTS.md` — VIEW-01, VIEW-02, VIEW-04, VIEW-05, VIEW-06 definitions and acceptance criteria

### Existing viewer to preserve (DO NOT MODIFY)
- `json-scada/src/AdminUI/src/components/AlarmsViewerPage.vue` — existing tag-based alarm viewer wrapped in iframe; must remain at `/alarms-viewer` unchanged

### Routing and navigation integration points
- `json-scada/src/AdminUI/src/router/index.js` — where new route `/s7plus-alarms` is registered
- `json-scada/src/AdminUI/src/components/DashboardPage.vue` — where new Dashboard tile is added (shortcuts array)

### Backend API pattern to follow
- `json-scada/src/server_realtime_auth/app/routes/auth.routes.js` — pattern for registering new `/Invoke/auth/` endpoints
- `json-scada/src/server_realtime_auth/app/controllers/auth.controller.js` — controller implementation pattern

### Data fetching pattern reference
- `json-scada/src/AdminUI/src/components/ProtocolConnectionsTab.vue` — `fetch('/Invoke/auth/...')` + `onMounted` + `ref([])` pattern to mirror

### i18n
- `json-scada/src/AdminUI/src/locales/en.json` — add `dashboard.s7plusAlarms` key here (and mirror in other locale files)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `v-data-table` (Vuetify): Established table component used in `ProtocolConnectionsTab.vue`, `TagsTab.vue`, etc. — use for alarm table
- `v-select` (Vuetify): Used for dropdowns throughout AdminUI — use for both filters
- `v-chip` (Vuetify): Available and used in AdminUI — use for Status column coloring
- `v-icon` (mdi): Used in AdminUI for boolean fields (`mdi-check` green / `mdi-close` red) — use for Ack column
- `fetch('/Invoke/auth/...')` pattern: Established data-fetching method for all admin-side queries

### Established Patterns
- Components use `ref([])` for reactive data + `onMounted(async () => { await fetchXxx() })` for initial load
- All pages are registered in `router/index.js` with explicit import and `{ path, component }` entry
- Dashboard shortcuts are entries in the `shortcuts` ref array with `{ titleKey, icon, color, route }` shape
- Auth-gated routes follow `app.use(accessPoint + 'auth/routeName', [authJwt.isAdmin], controller.handler)` pattern

### Integration Points
- `router/index.js`: Import `S7PlusAlarmsViewerPage` and add `{ path: '/s7plus-alarms', component: S7PlusAlarmsViewerPage }`
- `DashboardPage.vue` `shortcuts` array: Add entry with `titleKey: 'dashboard.s7plusAlarms'`, an appropriate lucide icon, `color: 'primary'`, `route: '/s7plus-alarms'`
- `server_realtime_auth/index.js`: The `db` MongoDB handle is available — new endpoint queries `db.collection('s7plusAlarmEvents').find().sort({createdAt: -1}).limit(200).toArray()`
- `auth.routes.js` + `auth.controller.js`: Add `listS7PlusAlarms` route + controller function following the `listProtocolConnections` shape

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard Vuetify approaches

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 03-read-only-alarm-viewer*
*Context gathered: 2026-03-18*

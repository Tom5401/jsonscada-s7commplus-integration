# Phase 14: DatablockBrowser - Context

**Gathered:** 2026-03-26
**Status:** Ready for planning

<domain>
## Phase Boundary

Add a new `DatablockBrowserPage` Vue component to AdminUI that:
- Lists all PLC datablocks for the selected connection (`db_name` + `db_number` columns)
- Provides a connection selector dropdown that starts empty — operator must explicitly choose a connection
- Has a per-row "Browse Tags" button that opens `TagTreeBrowserPage` in a new browser tab at `/#/s7plus-tag-tree?db=encodeURIComponent(db_name)&connectionNumber=N`
- Is registered as a route `/s7plus-datablocks` and added to the DashboardPage shortcuts

No driver changes, no backend changes, no TagTreeBrowserPage implementation in this phase.
</domain>

<decisions>
## Implementation Decisions

### Connection Selector
- **D-01:** Start with placeholder/empty state — no connection pre-selected on load. Operator must explicitly choose a connection from the dropdown before datablocks are fetched. This avoids silently showing data for the wrong PLC.

### TagTreeBrowser URL
- **D-02:** TagTreeBrowserPage route path is `/s7plus-tag-tree` — follows the `/s7plus-alarms` naming convention (`/s7plus-<content>`).
- **D-03:** Query params constructed with `encodeURIComponent(db_name)` for the `db` param: `/#/s7plus-tag-tree?db=${encodeURIComponent(row.db_name)}&connectionNumber=${connectionNumber}`. Vue Router automatically decodes `route.query.db` in Phase 15 — no double-decode issue.
- **D-04:** Open in new browser tab via `window.open(url, '_blank')`.

### Row Navigation UX
- **D-05:** Explicit "Browse Tags" button in a dedicated Actions column per row — consistent with S7PlusAlarmsViewerPage per-row button pattern. No whole-row click handler needed.

### Pattern Consistency
- **D-06:** Follow the established S7PlusAlarmsViewerPage pattern throughout:
  - `v-data-table` with `density="compact"` and `class="elevation-1"`
  - `v-select` with `density="compact"` for the connection filter
  - `fetch('/Invoke/auth/listS7PlusDatablocks?connectionNumber=N')` for data fetching
  - No auto-refresh (datablocks are static after driver startup)
  - Authentication: use the existing cookie-based auth — no additional middleware in frontend

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing S7Plus Frontend Page (implementation model)
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — v-data-table, v-select filter, fetch pattern, per-row button pattern, density="compact" style
- `json-scada/src/AdminUI/src/components/DashboardPage.vue` — shortcut card registration pattern (titleKey + icon + route)
- `json-scada/src/AdminUI/src/router/index.js` — route registration pattern (import + routes array entry)
- `json-scada/src/AdminUI/src/locales/en.json` — i18n key pattern (e.g., `dashboard.s7plusAlarms`)

### Backend API (already implemented in Phase 13)
- `json-scada/src/server_realtime_auth/index.js` lines 362–461+ — `listS7PlusDatablocks` endpoint already exists; accepts `?connectionNumber=N`, returns array sorted by `db_name`

### Phase Requirements
- `.planning/REQUIREMENTS.md` §DatablockBrowser — DBBROWSER-01, DBBROWSER-02, DBBROWSER-03

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `S7PlusAlarmsViewerPage.vue`: Full template model for v-data-table + filter row + per-row actions — copy the structural pattern (fetch on filter change, headers array, filteredItems computed, per-row button with `@click`)
- `DashboardPage.vue` shortcuts array: add one entry `{ titleKey: 'dashboard.datablockBrowser', icon: Database, color: 'primary', route: '/s7plus-datablocks' }` — `Database` icon already imported in that file (used for Metabase)
- `router/index.js`: add `import DatablockBrowserPage` + route entry `{ path: '/s7plus-datablocks', component: DatablockBrowserPage }`
- `locales/en.json`: add `"datablockBrowser": "Datablock Browser"` under `dashboard` key

### Established Patterns
- Pages live in `src/components/` as `PascalCase.vue` (convention confirmed by existing pages)
- `fetch` calls use `/Invoke/auth/...` path; cookie-based auth is implicit (no Authorization header needed from frontend)
- `v-select` items for connection dropdown: fetch from same `listS7PlusDatablocks` or derive unique `connectionNumber` values from results; alternatively hardcode from a `listConnections`-style approach — **researcher should check if a connection list endpoint exists**

### Integration Points
- New component file: `json-scada/src/AdminUI/src/components/DatablockBrowserPage.vue`
- Router: `json-scada/src/AdminUI/src/router/index.js`
- Dashboard: `json-scada/src/AdminUI/src/components/DashboardPage.vue`
- i18n: `json-scada/src/AdminUI/src/locales/en.json` (and optionally other locale files)

### Open Question for Researcher
- How to populate the connection selector options? Options: (a) call `listS7PlusDatablocks` with no filter first to get all unique connectionNumbers, (b) check if there's a `listConnections` or similar endpoint already in `server_realtime_auth/index.js`

</code_context>

<specifics>
## Specific Ideas

No specific visual references — follow the S7PlusAlarmsViewerPage look and feel exactly.
</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.
</deferred>

---

*Phase: 14-datablockbrowser*
*Context gathered: 2026-03-26*

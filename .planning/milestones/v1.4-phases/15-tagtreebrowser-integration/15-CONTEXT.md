# Phase 15: TagTreeBrowser & Integration - Context

**Gathered:** 2026-03-26
**Status:** Ready for planning

<domain>
## Phase Boundary

Build `TagTreeBrowserPage.vue` — a new AdminUI page that:
- Reads `?db=<name>&connectionNumber=<N>` URL params on load (route `/s7plus-tag-tree`, established in Phase 14)
- Fetches all tags for the datablock via `listS7PlusTagsForDb`, parses `protocolSourceBrowsePath` into a client-side tree
- Displays the tag hierarchy as an expandable tree; auto-expands the first level on load
- Shows live `value` + `type` + `protocolSourceObjectAddress` for leaf tag nodes, auto-refreshing every 5 seconds
- Calls `touchS7PlusActiveTagRequests` when nodes are expanded (and on each 5-second refresh cycle for currently expanded leaf tags, to prevent TTL expiry)
- Registers the route and adds a nav entry consistent with DatablockBrowserPage

Additionally, wire `S7PlusAlarmsViewerPage`:
- `originDbName` cells render as clickable links when non-empty — opens `TagTreeBrowserPage` in a new tab using `item.connectionId` as `connectionNumber`
- `originDbName === ""` renders as plain text (no link, no special chip)

No driver changes, no new backend endpoints in this phase.
</domain>

<decisions>
## Implementation Decisions

### Initial Tree State
- **D-01:** Auto-expand the first level on load — direct children of the DB (top-level structs and tags) are immediately visible without any user click.

### Struct / Intermediate Node Display
- **D-02:** Intermediate struct nodes (inferred from child paths, no own realtimeData doc) show folder icon + name only. No value, no type, no count badge. Clean separation: containers are folders, leaf tags show data.

### Leaf Tag Display
- **D-03:** Each leaf tag node shows three pieces of information:
  1. `type` — from the `type` field on the realtimeData doc (e.g., "analog", "digital")
  2. `value` — raw value from `realtimeData.value` (no stateTextTrue/stateTextFalse formatting)
  3. `protocolSourceObjectAddress` — the full address string (e.g., `"DBName".SubStruct.Tag`) for copy-paste into TIA Portal

### Alarms Viewer Integration: Empty originDbName
- **D-04:** When `originDbName === ""`, render as plain text (dash or blank) — no link, no chip. Only non-empty `originDbName` values become clickable links.

### Live Value Refresh
- **D-05:** Re-fetch `listS7PlusTagsForDb` every 5 seconds (same periodic pattern as alarm viewer). Update only the `value` field on leaf nodes in-place to preserve expand/collapse state across refreshes. Also call `touchS7PlusActiveTagRequests` for all currently expanded leaf tags on each refresh cycle to prevent TTL expiry.

### Pattern Consistency (carried from Phase 14 D-06)
- **D-06:** Follow S7PlusAlarmsViewerPage conventions: `v-container fluid`, `density="compact"`, `class="elevation-1"`, cookie-based auth (`/Invoke/auth/...`).

### Claude's Discretion
- Tree component choice: use `v-treeview` (Vuetify 3.10, already available) if it fits the lazy expand + in-place value update requirements cleanly; otherwise a recursive component is acceptable. Researcher/planner to decide based on v-treeview's open-state API.
- Path parsing: use `protocolSourceBrowsePath` (e.g., `DBName.SubStruct.Tag`) for building the hierarchy — strip the leading DB name segment, then split on `.` for remaining depth. TAGTREE-01 describes `protocolSourceObjectAddress` format for illustration, but `protocolSourceBrowsePath` is the cleaner field (no quote stripping needed).

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing Frontend Pages (implementation model)
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — fetch pattern, 5-second auto-refresh, per-row actions, filter row, `density="compact"` style
- `json-scada/src/AdminUI/src/components/DatablockBrowserPage.vue` — Phase 14 output; source of truth for route `/s7plus-tag-tree` URL construction, `window.open(..., '_blank')` pattern, connection selector
- `json-scada/src/AdminUI/src/router/index.js` — route registration pattern
- `json-scada/src/AdminUI/src/components/DashboardPage.vue` — shortcut card registration pattern

### Backend API (already implemented)
- `json-scada/src/server_realtime_auth/index.js` lines ~488–515 — `listS7PlusTagsForDb` endpoint; accepts `?connectionNumber=N&dbName=X`, filters by `protocolSourceBrowsePath` prefix, returns full realtimeData docs
- `json-scada/src/server_realtime_auth/index.js` lines ~517–555 — `touchS7PlusActiveTagRequests` endpoint; accepts array of `{connectionNumber, protocolSourceObjectAddress}`

### Data Shape
- `json-scada/src/S7CommPlusClient/TagsCreation.cs` — `rtData` class definition; relevant fields: `protocolSourceBrowsePath`, `protocolSourceObjectAddress`, `protocolSourceConnectionNumber`, `type`, `value`, `protocolSourceASDU`

### Phase Requirements
- `.planning/REQUIREMENTS.md` §TagTreeBrowser — TAGTREE-01, TAGTREE-02, TAGTREE-03, TAGTREE-04
- `.planning/REQUIREMENTS.md` §Integration — INTEGRATION-01

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `S7PlusAlarmsViewerPage.vue`: 5-second `setInterval` refresh pattern with `onUnmounted` cleanup — copy verbatim for TagTreeBrowserPage auto-refresh
- `DatablockBrowserPage.vue`: `onMounted` → `fetch('/Invoke/auth/listProtocolConnections')` pattern for connection list; `window.open(url, '_blank')` for new-tab navigation
- `useRoute()` from Vue Router for reading `route.query.db` and `route.query.connectionNumber` on mount
- Vuetify 3.10: `v-treeview` available; also `mdi-folder` / `mdi-tag` icons via mdi icon set already loaded

### Established Patterns
- Pages in `src/components/` as `PascalCase.vue`
- `fetch` calls use `/Invoke/auth/...`; cookie auth is implicit
- `document.documentElement.style.overflowY` toggled in `onMounted`/`onUnmounted` (established by DatablockBrowserPage)
- `<script setup>` with `ref`, `watch`, `onMounted`, `onUnmounted` — all pages use Composition API

### Integration Points
- New file: `json-scada/src/AdminUI/src/components/TagTreeBrowserPage.vue`
- Modify: `json-scada/src/AdminUI/src/router/index.js` — add `/s7plus-tag-tree` route
- Modify: `json-scada/src/AdminUI/src/components/DashboardPage.vue` — add shortcut card (if desired; researcher to confirm whether Phase 14 already added this or left it for Phase 15)
- Modify: `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — `originDbName` column template: link when non-empty, plain text when empty

</code_context>

<specifics>
## Specific Ideas

- Leaf tag address (`protocolSourceObjectAddress`) should be copy-paste friendly — render in a monospace style or small chip so operators can easily grab it for TIA Portal cross-reference.
- The `connectionId` field on alarm documents IS the connection number — use directly as `connectionNumber` in the TagTreeBrowser URL without any mapping.
</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.
</deferred>

---

*Phase: 15-tagtreebrowser-integration*
*Context gathered: 2026-03-26*

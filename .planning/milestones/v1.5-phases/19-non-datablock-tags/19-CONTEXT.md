# Phase 19: Non-Datablock Tags - Context

**Gathered:** 2026-03-30
**Status:** Ready for planning

<domain>
## Phase Boundary

Add 5 virtual area rows (IArea, QArea, MArea, S7Timers, S7Counters) to `DatablockBrowserPage.vue`. The rows appear at the **top of the first page**, above the real datablock list, preceded by a non-data section divider row labelled "Memory Areas". Clicking "Browse Tags" on any virtual row opens `TagTreeBrowserPage` with `?db=<areaName>&connectionNumber=<N>` — the same URL pattern already used for real datablocks. No backend changes required; `listS7PlusChildNodes` already handles area paths without special-casing.

</domain>

<decisions>
## Implementation Decisions

### Area row position and separation (D-01)
- **D-01:** The 5 virtual area rows appear at the **top** of the table, before the real datablock list, not appended at the end. A non-data section divider row labelled **"Memory Areas"** precedes them (implemented as a styled row item in the `datablocks` array or via a `v-data-table` group/prepend slot — agent's discretion on mechanism). Real datablocks follow after, with no explicit divider needed (natural separation by position).

### db_number column for area rows (D-02)
- **D-02:** The `db_number` cell for all 5 virtual area rows displays `—` (em-dash). These areas have no DB number in the S7CommPlus addressing model.

### Agent's Discretion
- **D-03 (discretion):** The 5 area names to inject are exactly: `IArea`, `QArea`, `MArea`, `S7Timers`, `S7Counters` — matching the ROADMAP specification and the names used in `protocolSourceBrowsePath` values in `realtimeData`.
- **D-04 (discretion):** The divider row mechanism — whether a static header item prepended to the computed `datablocks` array or a dedicated `<template #top>` / `<template #prepend>` slot — is the agent's choice. Prefer whatever requires the fewest template changes and remains compatible with `v-data-table`'s `:items` binding.
- **D-05 (discretion):** The "Browse Tags" button on area rows uses the same `browseDatablock(item)` function and `window.open` pattern already in place. No separate handler needed.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Frontend — target file
- `json-scada/src/AdminUI/src/components/DatablockBrowserPage.vue` — the only file modified in this phase; contains the `datablocks` ref, `watch(selectedConnection)` fetch, `browseDatablock()` function, and `v-data-table` template

### Frontend — context (read-only, do not modify)
- `json-scada/src/AdminUI/src/components/TagTreeBrowserPage.vue` — confirms that `?db=<name>` is read via `route.query.db` and passed directly to `listS7PlusChildNodes` as the `path` parameter; no area-specific handling needed

### Backend — context (read-only, do not modify)
- `json-scada/src/server_realtime_auth/index.js` §`listS7PlusChildNodes` (~L532) — confirms the endpoint queries `protocolSourceBrowsePath: path` with no area/datablock distinction; area paths work identically to DB paths

### Requirements
- `.planning/REQUIREMENTS.md` — AREA-01 defines acceptance criteria for this phase

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `browseDatablock(item)` in `DatablockBrowserPage.vue`: constructs URL as `` `/#/s7plus-tag-tree?db=${encodeURIComponent(row.db_name)}&connectionNumber=${selectedConnection.value}` `` and calls `window.open(url, '_blank')` — virtual area rows use this unchanged
- `v-data-table` `:items="datablocks"` binding — virtual rows can be prepended to the same array; no new table needed

### Established Patterns
- All rows use `db_name` and `db_number` fields — virtual area rows must conform to the same shape (`{ db_name: 'IArea', db_number: '—' }`) for the table to render them
- Connection dropdown empty-start pattern: `datablocks` is only populated after `selectedConnection` is set; virtual rows must also only appear after a connection is selected (they're per-connection in intent, even though they're static)

### Integration Points
- `watch(selectedConnection)` — the right place to inject virtual rows: after the API response populates real datablocks, prepend the virtual area rows + divider to `datablocks.value`

</code_context>

<specifics>
## Specific Ideas

- Area names are: `IArea`, `QArea`, `MArea`, `S7Timers`, `S7Counters` (exact casing to match `protocolSourceBrowsePath` values in MongoDB)
- Section divider label: **"Memory Areas"**
- db_number for area rows: `—`
- Area rows at top; real datablocks below — natural visual grouping without a second divider

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 19-non-datablock-tags*
*Context gathered: 2026-03-30*

---
phase: 17-lazy-tree-loading-boolean-display
plan: "01"
subsystem: ui
tags: [vue, vuetify, v-treeview, load-children, lazy-loading, s7plus]

requires:
  - phase: 16-backend-listS7PlusChildNodes-endpoint-index
    provides: "GET /Invoke/auth/listS7PlusChildNodes endpoint with hasChildren boolean per doc"

provides:
  - "Lazy one-level-at-a-time tag tree loading via Vuetify :load-children callback"
  - "Scoped 5-second value refresh — parallel listS7PlusChildNodes only for expanded parent paths"
  - "Boolean tag display: digital type shows TRUE/FALSE instead of 0/1"
  - "mapDocToNode() helper: children=[] for folders, children=undefined for leaves"

affects: [18-value-write-dialog, 19-non-datablock-tags]

tech-stack:
  added: []
  patterns:
    - "Vuetify v-treeview :load-children callback for lazy fetch-one-level expand"
    - "children=[] means expandable folder; children=undefined means leaf (no expand arrow)"
    - "Scoped refresh: getExpandedParentPaths() + Promise.all listS7PlusChildNodes per path"
    - "In-place value patching via buildNodeMap() — preserves expand state across refresh cycles"

key-files:
  created: []
  modified:
    - json-scada/src/AdminUI/src/components/TagTreeBrowserPage.vue

key-decisions:
  - "Scoped refresh uses listS7PlusChildNodes (not a separate values endpoint) since each refresh re-fetches child docs including updated value/valueString fields — consistent with the lazy load pattern"
  - "Digital type check uses doc.type === 'digital' (MongoDB field name), not protocolSourceDataType (ROADMAP referenced the wrong field)"
  - "buildNodeMap() builds a flat id→node map by walking the tree for O(1) lookup during value patch; does not replace children arrays (preserves expand state)"
  - "Root node is a synthetic folder; listS7PlusChildNodes path param for root uses dbName (the DB name string)"

patterns-established:
  - "mapDocToNode(doc): translate endpoint response doc to v-treeview node with correct children shape"
  - "getExpandedParentPaths(treeNodes, openSet): only returns paths that are expanded AND have loaded children (not the initial empty [] placeholder)"
  - "getExpandedLeafTags(treeNodes, openSet): walk only under open parent paths — guards against empty array before touchS7PlusActiveTagRequests call"

requirements-completed: [PERF-03, PERF-04, DISP-01]

duration: 25min
completed: 2026-03-30
---

# Phase 17: Lazy Tree Loading + Boolean Display Summary

**TagTreeBrowserPage.vue rewritten to fetch one tree level per expand via the Phase 16 endpoint, restricting value refresh to expanded nodes and displaying digital tags as TRUE/FALSE.**

## Performance

- **Duration:** ~25 min
- **Started:** 2026-03-30T11:09:00Z
- **Completed:** 2026-03-30T11:34:00Z
- **Tasks:** 1/1
- **Files modified:** 1

## Accomplishments

- Lazy loading: `loadRootChildren()` on mount + `onLoadChildren(item)` callback replaces the eager `loadTree()` / `buildTree()` full-subtree pattern — Vuetify caches after first expand so each node fetches exactly once
- Scoped refresh: `getExpandedParentPaths()` + `Promise.all` issues parallel `listS7PlusChildNodes` calls only for expanded parent nodes; leaf values patched in-place via `buildNodeMap()` lookup
- Boolean display: `formatLeafValue(item)` returns `'TRUE'` or `'FALSE'` for `item.type === 'digital'`, used in template append slot

## Task Commits

1. **Task 1: Rewrite TagTreeBrowserPage.vue with lazy load-children, scoped refresh, and boolean display** — `20c1dc26` (feat)

## Files Created/Modified

- `json-scada/src/AdminUI/src/components/TagTreeBrowserPage.vue` — Full rewrite: removed `buildTree`, `patchLeafValues`, `loadTree`, `listS7PlusTagsForDb`. Added `mapDocToNode`, `onLoadChildren`, `loadRootChildren`, `formatLeafValue`, `buildNodeMap`, `getExpandedParentPaths`, scoped `refreshValues`.

## Decisions Made

- **`type` field not `protocolSourceDataType`**: The plan mentioned `protocolSourceDataType` in the success criteria but the actual MongoDB field returned by the endpoint is `type` (set by the C# driver `GetTypeFromAsdu`). Used `item.type === 'digital'` accordingly.
- **Refresh via re-fetch**: The scoped refresh re-fetches child docs from `listS7PlusChildNodes` (which returns current values) rather than a separate lightweight values endpoint. This keeps the code simple and consistent while still being scoped to visible nodes.

## Deviations from Plan

### Auto-fixed Issues

**1. Field name correction: `protocolSourceDataType` → `type`**
- **Found during:** Task 1 (implementation review of plan spec vs actual endpoint response)
- **Issue:** ROADMAP success criterion mentioned `protocolSourceDataType` for boolean check but the endpoint response (Phase 16) returns `type` field (`"digital"`, `"analog"`, `"string"`, `"json"`)
- **Fix:** Used `item.type === 'digital'` in `formatLeafValue()`; plan's `<action>` section already noted this correction ("Note: the `type` field in realtimeData is `'digital'`...")
- **Files modified:** TagTreeBrowserPage.vue
- **Verification:** `Select-String -Pattern "digital"` confirms correct field check

## Self-Check

- [x] `:load-children="onLoadChildren"` present on v-treeview
- [x] `listS7PlusTagsForDb` removed — `listS7PlusChildNodes` used throughout
- [x] `buildTree` and `patchLeafValues` removed
- [x] `formatLeafValue(item)` in template; `item.type === 'digital'` branch returns `'TRUE'`/`'FALSE'`
- [x] `Promise.all` in `refreshValues` for parallel scoped fetches
- [x] `children: []` for expandable nodes, `children: undefined` for leaves
- [x] `openedNodes` ref and `v-model:opened` preserved
- [x] Empty tag guard in `touchExpandedLeafTags` preserved

---
phase: 15-tagtreebrowser-integration
plan: 01
subsystem: frontend
tags: [vue3, vuetify, tagtree, treeview, integration]
dependency_graph:
  requires:
    - Phase 13 backend APIs: listS7PlusTagsForDb, touchS7PlusActiveTagRequests
    - Phase 14 DatablockBrowserPage: established /s7plus-tag-tree URL pattern
  provides:
    - TagTreeBrowserPage.vue at /s7plus-tag-tree
    - originDbName clickable link in S7PlusAlarmsViewerPage
  affects:
    - S7PlusAlarmsViewerPage.vue (originDbName column slot added)
    - router/index.js (two new routes)
tech_stack:
  added: []
  patterns:
    - v-treeview with v-model:opened for expand-state control
    - patchLeafValues in-place mutation to preserve tree expand state across 5s refresh
    - getExpandedLeafTags with openedIds Set to collect visible leaf tags for touch calls
key_files:
  created:
    - json-scada/src/AdminUI/src/components/TagTreeBrowserPage.vue
    - json-scada/src/AdminUI/src/components/DatablockBrowserPage.vue (Phase 14 catch-up)
  modified:
    - json-scada/src/AdminUI/src/router/index.js
    - json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue
    - json-scada/src/AdminUI/src/components/DashboardPage.vue (Phase 14 catch-up)
    - json-scada/src/AdminUI/src/locales/en.json (Phase 14 catch-up)
decisions:
  - buildTree uses protocolSourceBrowsePath for path splitting (no quote stripping needed vs protocolSourceObjectAddress)
  - Leaf nodes have children: undefined (not children: []) to prevent spurious expand arrows in v-treeview
  - patchLeafValues mutates leaf .value in-place without replacing treeItems array reference — preserves v-model:opened state
  - Empty tag array guard before touchS7PlusActiveTagRequests fetch — backend rejects empty array with 400
  - originDbName link uses item.connectionId as connectionNumber — CONTEXT.md confirmed connectionId IS the connection number on alarm documents
metrics:
  duration: ~10 minutes
  completed_date: "2026-03-26"
  tasks_completed: 2
  files_created: 1
  files_modified: 3
---

# Phase 15 Plan 01: TagTreeBrowser & Integration Summary

**One-liner:** v-treeview tag hierarchy from protocolSourceBrowsePath parsing with 5s in-place value refresh and alarms viewer originDbName clickable link integration.

## Tasks Completed

| Task | Name | Commit | Key Files |
|------|------|--------|-----------|
| 1 | Create TagTreeBrowserPage.vue and register route | 6cb1751 (parent) / b83dec79 (submodule) | TagTreeBrowserPage.vue, router/index.js, DatablockBrowserPage.vue (catch-up) |
| 2 | Wire alarms viewer originDbName clickable link | 6cb1751 (parent) / ad98a18d (submodule) | S7PlusAlarmsViewerPage.vue |

## What Was Built

### TagTreeBrowserPage.vue (TAGTREE-01 through 04)

A new Vue 3 SFC at `/s7plus-tag-tree` that:

1. **Reads URL params on mount** (`?db=<name>&connectionNumber=<N>`) via `useRoute()` — TAGTREE-04
2. **Builds tree on load** by fetching `listS7PlusTagsForDb` and parsing `protocolSourceBrowsePath` strings (splitting on `.`, stripping the leading DB-name segment) into a nested node structure — TAGTREE-01
3. **Auto-expands first level** by initializing `openedNodes` with `[root.id]` — D-01
4. **Shows leaf tag data** in the `#append` slot: `type` chip, `value` code element, `protocolSourceObjectAddress` monospace span — TAGTREE-03, D-03
5. **Folder/tag icons** via `#prepend` slot — D-02
6. **Live value refresh every 5 seconds** via `setInterval` — TAGTREE-02, D-05
   - `patchLeafValues()` mutates only leaf `.value` fields in-place; treeItems array reference is never replaced so expand state is preserved
   - After patching, calls `touchExpandedLeafTags()` to keep TTL alive for all currently open leaf tags
7. **touch on expand** via `watch(openedNodes)` — TAGTREE-02
8. **Empty tag array guard** before `touchS7PlusActiveTagRequests` POST to prevent 400 error from backend

### S7PlusAlarmsViewerPage.vue (INTEGRATION-01)

Added `#[item.originDbName]` template slot:
- Non-empty `originDbName`: renders as `<a>` tag with `target="_blank"` pointing to `/#/s7plus-tag-tree?db=...&connectionNumber=...` — opens TagTreeBrowserPage in new tab
- Empty/falsy `originDbName`: renders as plain `<span>-</span>` — D-04
- Uses `item.connectionId` as connectionNumber (confirmed: connectionId field IS the connection number on alarm documents)

### router/index.js

Added two routes:
- `{ path: '/s7plus-datablocks', component: DatablockBrowserPage }` (Phase 14 catch-up)
- `{ path: '/s7plus-tag-tree', component: TagTreeBrowserPage }` (Phase 15 deliverable)

## Deviations from Plan

### Phase 14 Catch-Up (Rule 3 - Blocking Issue)

**Found during:** Task 1 setup

**Issue:** This worktree's json-scada submodule was at an older commit (pre-Phase 14 `30840511`) while master tracked `f3f9e552` which includes DatablockBrowserPage.vue, dashboard shortcut card, and i18n key. Since Phase 15 builds on Phase 14's route convention, these missing files were a prerequisite.

**Fix:** Copied DatablockBrowserPage.vue, DashboardPage.vue, and en.json from the main dev directory (`/c/Users/tnielen/Documents/Levvel_PoC/dev/json-scada/`) to bring the worktree up to the Phase 14 state before adding Phase 15 changes.

**Files added:** `DatablockBrowserPage.vue`, `DashboardPage.vue` (updated), `en.json` (updated)

**Commit:** included in b83dec79 (submodule Task 1 commit)

## Known Stubs

None — TagTreeBrowserPage fetches live data from the backend; no hardcoded empty values flow to UI rendering. The `openedNodes` starts with root node id so the first level is visible immediately.

## Self-Check: PASSED

| Check | Result |
|-------|--------|
| TagTreeBrowserPage.vue exists | FOUND |
| DatablockBrowserPage.vue exists | FOUND |
| S7PlusAlarmsViewerPage.vue exists | FOUND |
| 15-01-SUMMARY.md exists | FOUND |
| Commit 6cb1751 exists | FOUND |
| npm run build exit 0 | PASSED |

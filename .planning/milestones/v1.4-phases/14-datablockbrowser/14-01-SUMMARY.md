---
phase: 14-datablockbrowser
plan: 01
subsystem: AdminUI (Vue 3 frontend)
tags: [vue3, vuetify, adminui, datablocks, frontend]
dependency_graph:
  requires:
    - Phase 13 backend API: listS7PlusDatablocks + listProtocolConnections endpoints
  provides:
    - DatablockBrowserPage.vue: operator-facing datablock list page at /s7plus-datablocks
    - Route /s7plus-datablocks registered in Vue Router
    - Dashboard shortcut "Datablock Browser" added
  affects:
    - DashboardPage.vue: new shortcut card added
    - router/index.js: new route registered
    - en.json: new i18n key added
tech_stack:
  added: []
  patterns:
    - Vue 3 SFC with script setup
    - v-data-table density=compact + class=elevation-1 (project standard)
    - fetch on mount for connection list; watch for datablock fetch on selection change
    - window.open with hash-URL for new-tab navigation (createWebHashHistory)
key_files:
  created:
    - json-scada/src/AdminUI/src/components/DatablockBrowserPage.vue
  modified:
    - json-scada/src/AdminUI/src/router/index.js
    - json-scada/src/AdminUI/src/components/DashboardPage.vue
    - json-scada/src/AdminUI/src/locales/en.json
decisions:
  - selectedConnection initialized to null (D-01: no pre-selection on load)
  - window.open with /#/s7plus-tag-tree hash URL prefix (D-04: createWebHashHistory router)
  - encodeURIComponent on db_name in URL construction (D-03)
  - Database icon reused from existing DashboardPage import (no new import needed)
  - No setInterval auto-refresh (datablocks are static after driver startup)
metrics:
  duration: ~2min
  completed_date: 2026-03-26
  tasks_completed: 2
  tasks_total: 2
  files_changed: 4
---

# Phase 14 Plan 01: DatablockBrowserPage Summary

**One-liner:** Vue 3 DatablockBrowserPage with connection dropdown populating datablocks table and per-row "Browse Tags" new-tab navigation to TagTreeBrowserPage.

## What Was Built

A new `DatablockBrowserPage.vue` component added to the json-scada AdminUI. The page provides:

1. A connection selector dropdown populated from `GET /Invoke/auth/listProtocolConnections` on mount — starts empty per D-01
2. A Vuetify `v-data-table` showing `db_name` and `db_number` columns, populated by `GET /Invoke/auth/listS7PlusDatablocks?connectionNumber=N` when the operator selects a connection
3. Per-row "Browse Tags" button that opens a new browser tab at `/#/s7plus-tag-tree?db=...&connectionNumber=N` using `window.open` with `encodeURIComponent` on the db_name

The page is wired into the router at `/s7plus-datablocks`, visible on the dashboard as a "Datablock Browser" card using the existing `Database` lucide icon, and has an i18n key registered in `en.json`.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create DatablockBrowserPage.vue component | 3f905631 | DatablockBrowserPage.vue (created, 90 lines) |
| 2 | Wire router, dashboard, and i18n | c31def4d | router/index.js, DashboardPage.vue, en.json |

## Verification Results

- `npm run build` exits with code 0 (both tasks)
- `grep -c 'DatablockBrowserPage' router/index.js` returns 2 (import + route)
- `grep -c 'datablockBrowser' en.json` returns 1
- `grep -c 'datablockBrowser' DashboardPage.vue` returns 1
- `window.open` with `s7plus-tag-tree` confirmed in DatablockBrowserPage.vue
- `listProtocolConnections` fetch confirmed in DatablockBrowserPage.vue

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all data flows are wired to real backend endpoints (Phase 13 APIs). The page renders an empty table when no connection is selected, which is the correct empty state per D-01.

## Self-Check: PASSED

Files created/modified:
- FOUND: json-scada/src/AdminUI/src/components/DatablockBrowserPage.vue
- FOUND: json-scada/src/AdminUI/src/router/index.js (contains DatablockBrowserPage import + route)
- FOUND: json-scada/src/AdminUI/src/components/DashboardPage.vue (contains datablockBrowser entry)
- FOUND: json-scada/src/AdminUI/src/locales/en.json (contains datablockBrowser key)

Commits:
- FOUND: 3f905631 feat(14-01): create DatablockBrowserPage.vue component
- FOUND: c31def4d feat(14-01): wire DatablockBrowserPage into router, dashboard, and i18n

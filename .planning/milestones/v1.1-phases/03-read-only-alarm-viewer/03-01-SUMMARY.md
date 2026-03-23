---
phase: 03-read-only-alarm-viewer
plan: 01
subsystem: ui, api
tags: [vue3, vuetify, mongodb, express, s7plus, alarms]

# Dependency graph
requires:
  - phase: 02-driver-fixes
    provides: s7plusAlarmEvents MongoDB collection populated by driver with ackState and alarmClassName fields
provides:
  - GET /Invoke/auth/listS7PlusAlarms endpoint returning up to 200 alarm docs sorted newest-first
  - S7PlusAlarmsViewerPage.vue complete alarm viewer component with auto-refresh and client-side filtering
affects: [03-02-navigation-wiring]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "app.use() with [authJwt.isAdmin] middleware for admin-only endpoints in server_realtime_auth/index.js"
    - "Vue 3 Composition API (script setup) with setInterval/clearInterval lifecycle pattern for polling"
    - "v-data-table custom slot templates using #[`item.key`] dynamic slot syntax"

key-files:
  created:
    - json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue
  modified:
    - json-scada/src/server_realtime_auth/index.js

key-decisions:
  - "Endpoint inserted inline in index.js AUTHENTICATION block (not in auth.routes.js) to use module-level db handle directly — avoids passing db handle to routes file"
  - "projection: { _id: 0 } prevents BSON ObjectId serialization issues when returning MongoDB docs via JSON"
  - "Array.isArray(json) guard in fetchAlarms prevents error response objects from overwriting alarms array"

patterns-established:
  - "Alarm viewer fetch pattern: fetch endpoint -> Array.isArray guard -> ref assignment"
  - "overflowY scroll/hidden pattern on mount/unmount matching DashboardPage.vue"

requirements-completed: [VIEW-01, VIEW-02, VIEW-04, VIEW-05, VIEW-06]

# Metrics
duration: 2min
completed: 2026-03-19
---

# Phase 3 Plan 01: S7Plus Alarms Viewer — Backend Endpoint and Vue Component Summary

**REST endpoint at /Invoke/auth/listS7PlusAlarms querying s7plusAlarmEvents collection and Vue 3 viewer component with 11-column v-data-table, 5s auto-refresh, and client-side status/alarm-class filtering**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-19T06:52:33Z
- **Completed:** 2026-03-19T06:53:59Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments

- Backend: registered `GET /Invoke/auth/listS7PlusAlarms` in `index.js` with `authJwt.isAdmin` guard, querying `s7plusAlarmEvents` collection sorted by `createdAt` descending, limited to 200 docs, excluding `_id` from projection
- Frontend: created `S7PlusAlarmsViewerPage.vue` with all 11 VIEW-02 columns (Source, Date, Time, Status, Acknowledge, Alarm class name, Event text, ID, Additional text 1-3)
- Auto-refresh every 5 seconds via `setInterval` in `onMounted`, cleared via `clearInterval` in `onUnmounted` (VIEW-04)
- Client-side filtering: Status filter (All/Incoming/Outgoing mapped to Coming/Going) and dynamic Alarm Class filter computed from live data (VIEW-05, VIEW-06)

## Task Commits

Each task was committed atomically (inside json-scada submodule):

1. **Task 1: Add listS7PlusAlarms endpoint to index.js** - `04685753` (feat)
2. **Task 2: Create S7PlusAlarmsViewerPage.vue component** - `d0e1a478` (feat)

## Files Created/Modified

- `json-scada/src/server_realtime_auth/index.js` - Added listS7PlusAlarms app.use block inside AUTHENTICATION block between auth.routes and user.routes requires
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` - New full alarm viewer component (140 lines)

## Decisions Made

- Endpoint inserted directly in `index.js` AUTHENTICATION block (not delegated to `auth.routes.js`) so the module-level `db` native MongoDB handle is in scope without threading it through route files
- `projection: { _id: 0 }` avoids BSON ObjectId serialization issues when sending MongoDB docs as JSON
- `Array.isArray(json)` guard in `fetchAlarms` ensures error response objects never overwrite the alarms array on network or server errors

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

- `git add` from the project root failed because `json-scada` is a git submodule. Resolved by committing from within the `json-scada/` subdirectory. No file changes required.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- Plan 02 (03-02) can now wire `S7PlusAlarmsViewerPage.vue` into AdminUI navigation — both the component and its backend endpoint are complete
- No blockers for Plan 02

---
*Phase: 03-read-only-alarm-viewer*
*Completed: 2026-03-19*

## Self-Check: PASSED

- FOUND: json-scada/src/server_realtime_auth/index.js
- FOUND: json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue
- FOUND: .planning/phases/03-read-only-alarm-viewer/03-01-SUMMARY.md
- FOUND commit 04685753: feat(03-01): add listS7PlusAlarms endpoint to index.js
- FOUND commit d0e1a478: feat(03-01): create S7PlusAlarmsViewerPage.vue component

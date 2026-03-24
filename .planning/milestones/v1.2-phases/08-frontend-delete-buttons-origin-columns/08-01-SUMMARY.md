---
phase: 08-frontend-delete-buttons-origin-columns
plan: 01
subsystem: frontend
tags: [vue, alarm-viewer, delete, origin-columns, ack-button]

provides:
  - Origin DB Name column in S7Plus alarm viewer table
  - DB Number column in S7Plus alarm viewer table
  - Per-row delete button (red trash icon) with optimistic removal
  - Coming+unacked warning dialog before delete
  - Bulk "Delete Filtered (N)" button in filter toolbar
  - Confirmation dialog for bulk delete
  - Ack button restored (cherry-pick of Phase 4 branch into main)

affects:
  - json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue
  - json-scada/src/server_realtime_auth/index.js
  - json-scada/src/S7CommPlusClient/MongoCommands.cs
  - json-scada/src/S7CommPlusClient/AlarmThread.cs
  - json-scada/src/S7CommPlusClient/Common.cs

key-files:
  created: []
  modified:
    - json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue
    - json-scada/src/server_realtime_auth/index.js
    - json-scada/src/S7CommPlusClient/MongoCommands.cs

key-decisions:
  - "ORIGIN-08 (RelationId column) intentionally skipped per locked decision D-02 — raw 64-bit integer adds no operator value"
  - "Phase 4 ack branch was stranded on unmerged branch since Phase 3 — cherry-picked onto Phase 8 tip to restore Ack button"
  - "MongoCommands.cs uses queued AlarmThread approach (ae59194e) over direct SendAlarmAck — dedicated alarmConn avoids polling contention"
  - "NativeDoubleSerializer removed from rtCommand (ae59194e) — standard BSON double deserializer sufficient"
  - "Both ackS7PlusAlarm and deleteS7PlusAlarms endpoints coexist in index.js"

duration: ~1h (Task 1 automated ~6min; checkpoint + ack branch recovery ~50min)
completed: 2026-03-24
---

# Phase 8 Plan 01: Frontend Delete Buttons & Origin Columns Summary

**S7PlusAlarmsViewerPage.vue updated with origin columns, delete functionality, and Ack button restored via cherry-pick.**

## Performance

- **Duration:** ~1h total (automated task + human verification + branch recovery)
- **Started:** 2026-03-24
- **Completed:** 2026-03-24
- **Tasks:** 2/2 complete (Task 1 automated, Task 2 human-verified — approved)
- **Files modified:** 5 (Vue + index.js + MongoCommands.cs + AlarmThread.cs + Common.cs)

## Accomplishments

- Added `Origin DB Name` and `DB Number` columns to alarm viewer (ORIGIN-06, ORIGIN-07)
- Added per-row delete column with red trash icon (`mdi-delete`) immediately after Acknowledge column (DELETE-04)
- Per-row delete: immediate optimistic removal for normal rows; warning dialog for Coming+unacked rows (DELETE-06)
- Added "Delete Filtered (N)" button in filter toolbar, disabled at 0, bulk delete with confirmation dialog (DELETE-05)
- Both delete paths POST to `/Invoke/auth/deleteS7PlusAlarms` with `{ ids: [...] }`, no Authorization header (D-04, D-09)
- `confirmState` ref manages both dialog types (row-active, bulk)
- RelationId column intentionally omitted per D-02

**Branch recovery:** Phase 4 ack button commits (`be400938`, `0fbc1ed7`, `ae59194e`) were stranded on an unmerged branch since Phase 3. Cherry-picked onto Phase 8 tip with conflict resolution:
- `index.js`: both `ackS7PlusAlarm` and `deleteS7PlusAlarms` endpoints preserved
- `MongoCommands.cs`: queued AlarmThread approach (ae59194e) kept over direct SendAlarmAck
- `Common.cs`: NativeDoubleSerializer removed per ae59194e

## Task Commits

1. **Task 1: Vue changes** — `003a959d` in json-scada, `e217631` in superproject worktree
2. **Branch recovery** — `be89156d`, `600c6dc7`, `a58d693f` in json-scada; `33e57ee` in superproject

## Deviations from Plan

### Intentional
- **ORIGIN-08 skipped:** RelationId column not added per locked decision D-02

### Discovery
- **Phase 4 ack branch not merged:** Phases 5–8 were built on Phase 3 tip. Phase 4's Ack button (be400938, 0fbc1ed7) and full ack implementation (ae59194e) existed on a separate branch that diverged at Phase 3 and was never merged. Cherry-picked during Phase 8 human verification.

## Self-Check: PASSED

- FOUND: json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue (contains originDbName, handleDeleteRow, confirmState, executeDeleteFiltered, mdi-delete, Delete Filtered, pendingAcks, ackAlarm)
- FOUND: json-scada/src/server_realtime_auth/index.js (contains ackS7PlusAlarm, deleteS7PlusAlarms)
- FOUND: json-scada/src/S7CommPlusClient/MongoCommands.cs (contains s7plus-alarm-ack branch)
- FOUND commit: 003a959d (Task 1 — Vue changes)
- FOUND commit: a58d693f (Phase 4 ack branch cherry-pick)
- Human verification: APPROVED by user 2026-03-24

---
phase: 11-vue-ui-enhancements
plan: "02"
subsystem: ui
tags: [vue, vuetify, alarm-viewer, frontend, bulk-action, filter]

requires:
  - phase: 11-vue-ui-enhancements
    plan: "01"
    provides: Updated S7PlusAlarmsViewerPage.vue with timestamp, priority, ack indicator, page preservation

provides:
  - Source PLC filter dropdown (connectionName-based, same pattern as Status/AlarmClass)
  - Ack All bulk action with count label and disabled-when-zero guard
  - Confirmation dialog extended for ack-all type with Acknowledge button

affects:
  - Future plans sharing S7PlusAlarmsViewerPage.vue

tech-stack:
  added: []
  patterns:
    - "Bulk action with confirmation dialog: handleX sets confirmState with type; dialog dispatches to executeX; count shown pre-confirmation"
    - "connectionOptions computed from alarms.value.map(a => a.connectionName) — same Set/sort/prepend-All pattern as alarmClassOptions"
    - "executeAckAll: sequential await loop over filtered targets; each ack independent (failure does not block rest)"

key-files:
  created: []
  modified:
    - json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue

key-decisions:
  - "connectionFilter follows exact same pattern as alarmClassFilter — computed options from alarms.value, ref defaulting to All, condition in filteredAlarms"
  - "ackAllCount computed scoped to filteredAlarms (not all alarms) — count changes as filters change, button disabled when 0"
  - "executeAckAll uses alarm.connectionId (not connectionNumber) consistent with existing per-row ackAlarm call pattern"
  - "Confirmation dialog extended with v-else-if for ack-all; confirm button color switches orange/red by type"

patterns-established:
  - "Multi-type confirmation dialog: extend confirmState.type with v-else-if in card-title, card-text, and confirm button; ternary dispatch chain in @click"

requirements-completed: [VIEWER-04, VIEWER-05]

duration: 5min
completed: 2026-03-25
---

# Phase 11 Plan 02: Vue UI Enhancements (Source Filter + Ack All) Summary

**Source PLC filter dropdown and Ack All bulk action added to S7Plus Alarms Viewer — operators can narrow alarms by connection and bulk-acknowledge all unacked+acknowledgeable alarms matching active filters.**

## Performance

- **Duration:** ~5 min
- **Started:** 2026-03-25T17:58:52Z
- **Completed:** 2026-03-25T18:03:00Z
- **Tasks:** 1/2 (Task 2 is human-verify checkpoint — awaiting visual verification)
- **Files modified:** 1

## Accomplishments

- Added `connectionFilter` ref and `connectionOptions` computed (distinct `connectionName` values from alarm data, sorted, with 'All' prepended)
- Extended `filteredAlarms` computed with `connectionMatch` condition alongside existing `stateMatch` and `classMatch`
- Added `ackAllCount` computed: count of `filteredAlarms` where `!ackState && isAcknowledgeable === true`
- Added `handleAckAll` / `executeAckAll` functions: confirmation dialog dispatch and sequential `await ackAlarm()` loop
- Added Source `v-select` in filter row (after Alarm Class, before Delete Filtered)
- Added "Ack All (N)" button in orange with `:disabled="ackAllCount === 0"` (next to Delete Filtered)
- Extended confirmation dialog: `v-else-if="confirmState.type === 'ack-all'"` in card-title, card-text, and confirm button; confirm button color driven by type (orange for ack-all, red for delete)

## Task Commits

1. **Task 1: Add source filter dropdown, Ack All button, and confirmation dialog** - `30840511` (feat)

**Plan metadata commit:** (pending — awaiting human-verify checkpoint at Task 2)

## Files Created/Modified

- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — Source filter, Ack All button, ackAllCount computed, handleAckAll/executeAckAll functions, extended confirmation dialog

## Decisions Made

- `alarm.connectionId` passed as second arg to `ackAlarm()` in `executeAckAll` — consistent with existing per-row call `ackAlarm(item.cpuAlarmId, item.connectionId)` (not `connectionNumber` which is the function parameter name, not the property name)
- `ackAllCount` computed is scoped to `filteredAlarms` so it reflects the current combined filter state — Ack All count changes as Status/AlarmClass/Source filters change
- Sequential `await` loop in `executeAckAll` per RESEARCH.md recommendation — each ack independent, one failure does not block rest

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all data flows from `alarms.value` through `filteredAlarms` computed. No placeholder values.

## Self-Check: PASSED

- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — FOUND (modified, committed as 30840511)
- Commit 30840511 in json-scada submodule — FOUND
- grep: connectionFilter (4), connectionOptions (2), connectionMatch (2), ackAllCount (4), handleAckAll (2), executeAckAll (2), ack-all (6), Ack All (1) — all present

## Next Phase Readiness

- Task 2 (human-verify checkpoint) requires visual verification in browser
- After approval: state updates and final metadata commit complete this plan and phase 11

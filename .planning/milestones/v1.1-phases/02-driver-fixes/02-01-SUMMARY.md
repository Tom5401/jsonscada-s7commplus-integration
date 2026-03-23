---
phase: 02-driver-fixes
plan: 01
subsystem: driver
tags: [csharp, dotnet, s7commplus, plcsim, mongodb, alarms]

# Dependency graph
requires:
  - phase: 01-alarm-pipeline
    provides: AlarmThread.cs with BuildAlarmDocument() writing to MongoDB
provides:
  - BuildAlarmDocument() with correct ackState sentinel (DateTime.UnixEpoch)
  - Confirmed PLCSIM trace data: AckTimestamp sentinel, AlarmClass IDs, AllStatesInfo values
affects:
  - 02-02 (alarmClassName fix — same AlarmThread.cs file)
  - 04-alarm-ack (AllStatesInfo byte values useful for ack state tracking)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "AckTimestamp sentinel: DateTime.UnixEpoch (protocol sends raw uint64 = 0 for unacknowledged)"
    - "Single comment + single code line pattern for sentinel documentation at BSON field site"

key-files:
  created: []
  modified:
    - json-scada/src/S7CommPlusClient/AlarmThread.cs

key-decisions:
  - "Use DateTime.UnixEpoch as ackState sentinel (Case A) — confirmed by PLCSIM trace showing 01/01/1970 00:00:00 for unacknowledged alarms"
  - "Remove trace log line from production code after sentinel confirmed — no debug artifacts in ship build"
  - "Single-file surgical fix only: no changes to AlarmsAsCgs.cs or any other file"

patterns-established:
  - "Confirm sentinel values via live PLCSIM trace before committing fixes — do not rely on static analysis alone"

requirements-completed: [DRVR-01]

# Metrics
duration: ~45min (spanning two sessions including PLCSIM trace checkpoint)
completed: 2026-03-18
---

# Phase 2 Plan 01: ackState Sentinel Fix Summary

**ackState always-true bug fixed: replaced DateTime.MinValue with DateTime.UnixEpoch in BuildAlarmDocument(), sentinel confirmed via PLCSIM trace showing 01/01/1970 00:00:00 for unacknowledged alarms**

## Performance

- **Duration:** ~45 min (two sessions including human-verify checkpoint)
- **Started:** 2026-03-18
- **Completed:** 2026-03-18
- **Tasks:** 3 (Task 1 auto, Task 2 human-verify checkpoint, Task 3 auto)
- **Files modified:** 1 (AlarmThread.cs inside json-scada submodule)

## Accomplishments

- Identified root cause: `DtFromValueTimestamp(0)` returns Unix epoch (1970-01-01), not `DateTime.MinValue` (0001-01-01), so the old `!= DateTime.MinValue` expression always evaluated to `true`
- Confirmed sentinel via live PLCSIM trace: unacknowledged AckTimestamp = `01/01/1970 00:00:00`
- Applied surgical fix: `dai.AsCgs.AckTimestamp != DateTime.UnixEpoch` with confirming comment
- Removed diagnostic trace log line — no debug artifacts in production code
- Build passes: 0 errors, 0 warnings

## PLCSIM Trace Findings (for future phase reference)

**Unacknowledged alarm (SubtypeId = Coming / 2673):**
- AckTimestamp: `01/01/1970 00:00:00` (Unix epoch — protocol sends raw uint64 = 0)
- AllStatesInfo: `133`
- AlarmClass numeric: `33`
- AlarmClass name in TIA Portal: "Acknowledgment"

**Acknowledged alarm:**
- Could not capture — driver logs "unknown PDU type, skipping" when ack PDU arrives
- Ack PDU type is not yet handled by S7CommPlusDriver (Phase 4 scope)
- Fix focuses on Coming/Going events using AckTimestamp sentinel (correct approach)

**MongoDB confirmation:** All documents showed `ackState: true` before fix — bug confirmed.

## Task Commits

Each task was committed atomically inside the json-scada submodule and the parent repo pointer updated:

1. **Task 1: Add PLCSIM trace logging to BuildAlarmDocument()** - `d115582` (feat)
2. **Task 2: PLCSIM trace — observe AckTimestamp sentinel** - human-verify checkpoint (no code change)
3. **Task 3: Apply surgical ackState fix with confirmed sentinel** - `4636de9` (fix)

## Files Created/Modified

- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — ackState line replaced with UnixEpoch sentinel + confirming comment; trace log line removed

## Decisions Made

- **Case A applied (DateTime.UnixEpoch):** PLCSIM trace confirmed unacknowledged AckTimestamp = 01/01/1970 00:00:00. Case B (MinValue) was ruled out — static analysis had predicted MinValue but the library initialises to epoch, not .NET's MinValue.
- **Sentinel comment placed inline:** Comment is on the line immediately above the BSON field entry, following the plan's "one comment + one code line" rule.
- **Trace line removed:** Diagnostic log added in Task 1 removed in Task 3 per plan's CRITICAL rules.

## Deviations from Plan

None — plan executed exactly as written. PLCSIM trace produced a clear sentinel value (Case A), no ambiguity requiring AllStatesInfo fallback (Case C).

## Issues Encountered

- Acknowledged alarm trace could not be captured: the driver catches ack PDU as "unknown PDU type" (NotImplementedException) and skips it. This is a pre-existing limitation documented in the continuation state. The ackState fix is correct regardless — it operates on Coming/Going event AckTimestamp values, not on ack-specific PDUs.

## User Setup Required

None — no external service configuration required.

## Next Phase Readiness

- Phase 2 Plan 02 (alarmClassName fix) can proceed immediately — same file, independent change
- AllStatesInfo = 133 for unacknowledged Coming alarms noted for Phase 4 reference
- Ack PDU handling gap documented in Phase 4 scope (Wireshark capture required before implementation)

---
*Phase: 02-driver-fixes*
*Completed: 2026-03-18*

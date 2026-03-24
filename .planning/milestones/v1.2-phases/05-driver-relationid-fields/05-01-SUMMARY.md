---
phase: 05-driver-relationid-fields
plan: 01
subsystem: driver
tags: [csharp, mongodb, bson, s7commplus, alarm]

# Dependency graph
requires:
  - phase: 04-ack-write-back
    provides: AlarmThread.cs with BuildAlarmDocument() as base to extend
provides:
  - relationId (BsonInt64) field in every new alarm event document in s7plusAlarmEvents
  - dbNumber (BsonInt32) field in every new alarm event document in s7plusAlarmEvents
affects:
  - 06-driver-startup-db-name-map (depends on relationId per-document for name resolution)
  - 07-backend-delete-endpoint (listS7PlusAlarms should expose these fields)
  - 08-frontend-delete-buttons (Origin columns display relationId, dbNumber, originDbName)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "uint bit-extraction from ulong: (uint)(dai.CpuAlarmId >> 32) for upper 32 bits"
    - "dbNumber from lower 16 bits of relationId: relationId & 0xFFFF"
    - "BsonInt64 for uint values that may exceed Int32.MaxValue (e.g. PLC relation IDs)"

key-files:
  created: []
  modified:
    - json-scada/src/S7CommPlusClient/AlarmThread.cs

key-decisions:
  - "Store relationId as BsonInt64 not BsonInt32 — PLC values like 0x8a0e0005 exceed Int32.MaxValue and would silently corrupt if stored as Int32"
  - "dbNumber uses lower 16 bits of relationId (& 0xFFFF) not upper bits (>> 16) — confirmed by S7CommPlusConnection.cs:1247"
  - "Do not add originDbName in this phase — Phase 6 scope; keeping Phase 5 minimal and focused"

patterns-established:
  - "Bit-extraction pattern for CpuAlarmId: upper 32 bits = relationId, lower 16 bits of that = dbNumber"

requirements-completed: [ORIGIN-01, ORIGIN-02]

# Metrics
duration: 10min
completed: 2026-03-23
---

# Phase 5 Plan 01: Driver RelationId Fields Summary

**Added relationId (BsonInt64, from upper 32 bits of CpuAlarmId) and dbNumber (BsonInt32, lower 16 bits of relationId) to every alarm event document written by BuildAlarmDocument() in AlarmThread.cs — enabling Phase 6 DB name resolution.**

## Performance

- **Duration:** ~10 min
- **Started:** 2026-03-23T13:01:00Z
- **Completed:** 2026-03-23T13:11:00Z
- **Tasks:** 1
- **Files modified:** 1

## Accomplishments

- Added `uint relationId = (uint)(dai.CpuAlarmId >> 32)` — exact inverse of BrowseAlarms.cs:414 packing
- Added `uint dbNumber = relationId & 0xFFFF` — confirmed by S7CommPlusConnection.cs:1247
- Stored `relationId` as `BsonInt64` to prevent silent value corruption for PLCs with IDs > Int32.MaxValue
- Project builds with 0 errors after the change

## Task Commits

Each task was committed atomically:

1. **Task 1: Add relationId and dbNumber fields to BuildAlarmDocument** - `51b44e8` (feat)
   - json-scada submodule commit: `bf9a91d1`

**Plan metadata:** (docs commit — see below)

## Files Created/Modified

- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — Added two local variables and two BsonDocument fields to BuildAlarmDocument()

## Decisions Made

- Used `BsonInt64` for `relationId` because PLC values like `0x8a0e0005` (2,317,140,997) exceed `Int32.MaxValue`. Storing as Int32 would silently wrap to negative, corrupting Phase 6 lookups.
- Used `& 0xFFFF` (not `>> 16`) for `dbNumber` extraction — lower 16 bits of relationId, confirmed by `S7CommPlusConnection.cs:1247`.
- Did not add `originDbName` — Phase 6 scope per D-05 in the plan.

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

None. The submodules were not initialized in the worktree at start; `git submodule update --init --recursive` was run to set them up before any code changes.

## User Setup Required

None — no external service configuration required.

## Next Phase Readiness

- Every new alarm document now carries `relationId` and `dbNumber` — Phase 6 can use these to build a DB name map at startup and resolve `originDbName` per alarm at write time.
- Phase 6 spike recommended before full implementation: confirm `srv.connection` state at browse point, verify RelationId-to-name mapping against a live PLCSIM instance.

---
*Phase: 05-driver-relationid-fields*
*Completed: 2026-03-23*

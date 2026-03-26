---
phase: 12-driver-datablock-persistence
plan: 01
subsystem: database
tags: [csharp, mongodb, s7commplus, datablock, persistence, upsert]

requires:
  - phase: 06-driver-startup-db-name-map
    provides: GetListOfDatablocks browse and RelationIdNameMap pattern established

provides:
  - s7plusDatablocks MongoDB collection with one document per browseable PLC datablock
  - DatablocksCollectionName constant ("s7plusDatablocks") for Phase 13 API use
  - EnsureDatablockIndexes startup method (unique compound index on connectionNumber+db_name, secondary on connectionNumber)
  - UpsertDatablocks helper for bulk upsert of datablock list on successful PLC connect

affects:
  - 13-api-datablock-endpoints
  - 14-frontend-datablock-browser

tech-stack:
  added: []
  patterns:
    - BulkWriteAsync with ReplaceOneModel IsUpsert=true for idempotent startup persistence
    - UInt32 cast to (BsonInt32)(int) for db_number/db_block_relid/db_block_ti_relid (consistent with Phase 5 alarm event pattern)
    - MongoDatabase null guard before collection access (same as GetActiveAddressesForConnection)

key-files:
  created: []
  modified:
    - json-scada/src/S7CommPlusClient/Common.cs
    - json-scada/src/S7CommPlusClient/Program.cs

key-decisions:
  - "Upsert keyed on {connectionNumber, db_name} — same compound key as unique index — ensures no duplicates on driver restart"
  - "UpsertDatablocks called only on successful GetListOfDatablocks (browseRes == 0) — stale data from previous run preserved on browse failure"
  - "BulkWriteAsync IsOrdered=false — single round-trip, order-independent, consistent with Phase 6 pattern recommendation"
  - "GetAwaiter().GetResult() sync-over-async — consistent with AlarmThread.cs pattern throughout the driver"

patterns-established:
  - "EnsureXxxIndexes pattern: static method accepting IMongoDatabase, called at Main startup after EnsureActiveTagRequestIndexes"
  - "UpsertXxx pattern: static helper with null-guard, BulkWriteAsync ReplaceOne IsUpsert, exception caught at LogLevelBasic"

requirements-completed: [DRIVER-03]

duration: 10min
completed: 2026-03-26
---

# Phase 12 Plan 01: Driver — Datablock Persistence Summary

**Bulk-upsert of full PLC datablock list to MongoDB s7plusDatablocks collection at driver startup, with unique compound index and idempotent restart behavior**

## Performance

- **Duration:** ~10 min
- **Started:** 2026-03-26T08:00:00Z
- **Completed:** 2026-03-26T08:10:00Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments

- Added `DatablocksCollectionName = "s7plusDatablocks"` constant and `EnsureDatablockIndexes` method to Common.cs
- Added `UpsertDatablocks` static helper in Program.cs using `BulkWriteAsync` with `ReplaceOneModel IsUpsert=true`
- Wired `UpsertDatablocks` into ConnectionThread (after successful RelationIdNameMap build) and `EnsureDatablockIndexes` into Main startup
- Build succeeds with 0 errors, 0 warnings

## Task Commits

Each task was committed atomically in the json-scada submodule:

1. **Task 1: Add DatablocksCollectionName constant and EnsureDatablockIndexes to Common.cs** - `315eee26` (feat)
2. **Task 2: Add UpsertDatablocks helper and wire into ConnectionThread and Main** - `69ec0ba9` (feat)

## Files Created/Modified

- `json-scada/src/S7CommPlusClient/Common.cs` — Added `DatablocksCollectionName` constant and `EnsureDatablockIndexes` method with two indexes
- `json-scada/src/S7CommPlusClient/Program.cs` — Added `UpsertDatablocks` helper method, called from ConnectionThread and Main

## Decisions Made

- Upsert keyed on `{connectionNumber, db_name}` matches unique index — prevents duplicate documents on driver restart
- `UpsertDatablocks` only called on `browseRes == 0` (success path) — stale data from prior run is preserved when browse fails; correct behavior
- `(BsonInt32)(int)` cast for UInt32 fields (db_number, db_block_relid, db_block_ti_relid) consistent with dbNumber pattern from Phase 5 alarm events
- `BulkWriteAsync` with `IsOrdered=false` — single round-trip, no ordering needed for upserts
- `GetAwaiter().GetResult()` sync-over-async — consistent with all other sync-thread MongoDB operations in the driver

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

- The source files are in a git submodule (`json-scada`), not directly in the planning repo. Commits were made to the `json-scada` submodule on its `master` branch. The parent repo points to the updated submodule commit.

## Next Phase Readiness

- `s7plusDatablocks` collection will be populated on next driver startup connecting to a PLC
- Phase 13 (API endpoints) can query `s7plusDatablocks` by `connectionNumber` using the `conn_number` index
- Phase 14 (DatablockBrowser UI) can consume Phase 13 API to display the datablock list
- No blockers

---
*Phase: 12-driver-datablock-persistence*
*Completed: 2026-03-26*

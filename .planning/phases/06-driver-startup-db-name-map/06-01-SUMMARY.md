---
phase: 06-driver-startup-db-name-map
plan: "01"
subsystem: S7CommPlusClient (C# driver)
tags: [driver, mongodb, alarm, startup, browse, csharp]
dependency_graph:
  requires: [Phase 05 — relationId and dbNumber fields in alarm documents]
  provides: [RelationIdNameMap populated at driver startup, originDbName field on every alarm document]
  affects: [AlarmThread MongoDB writes, Phase 7 alarm list API, Phase 8 alarm viewer]
tech_stack:
  added: []
  patterns: [TryGetValue with empty-string fallback, scoped local variable block for startup browse]
key_files:
  created: []
  modified:
    - json-scada/src/S7CommPlusClient/Common.cs
    - json-scada/src/S7CommPlusClient/Program.cs
    - json-scada/src/S7CommPlusClient/AlarmThread.cs
decisions:
  - "Browse on tag connection (srv.connection) not alarm connection — alarm connection does not exist yet at this point"
  - "Empty-string fallback for originDbName (not null) — consistent schema for Phase 7/8 API consumers"
  - "Scoped block {} for local variables (dbInfoList, browseRes) — keeps ConnectionThread scope clean"
metrics:
  duration: "~2 min"
  completed: "2026-03-23"
  tasks_completed: 2
  files_modified: 3
---

# Phase 06 Plan 01: Driver Startup DB Name Map Summary

## One-Liner

Driver builds a `Dictionary<uint, string>` RelationId-to-datablock-name map at PLC connect time via `GetListOfDatablocks`, and writes the resolved TIA Portal DB name as `originDbName` on every alarm document.

## What Was Done

### Task 1: Add RelationIdNameMap field and browse call (commits: 7a7c9104)

**Common.cs** — Added `[BsonIgnore] public Dictionary<uint, string> RelationIdNameMap` to `S7CP_connection`, initialized to empty (not null) so downstream `TryGetValue` is always safe without null checks.

**Program.cs** — Inserted a browse block in `ConnectionThread` between "connected" and `alarmThread.Start()`. The block calls `srv.connection.GetListOfDatablocks(out dbInfoList)`:
- Success: builds map from `db_block_relid` → `db_name`, assigns to `srv.RelationIdNameMap`, logs count at `LogLevelDetailed`
- Failure (non-zero return): logs warning at `LogLevelBasic` (visible at default log level), resets map to empty, continues to alarm thread start without interruption

### Task 2: Add originDbName field to BuildAlarmDocument (commit: 5d43a6aa)

**AlarmThread.cs** — Added `originDbName` as the last field in the `BsonDocument` returned by `BuildAlarmDocument()`. Uses `srv.RelationIdNameMap.TryGetValue(relationId, out var dbName) ? dbName : ""` — matching the pattern already used for `AlarmClassNames.TryGetValue`. Field is placed after `dbNumber` to group the three origin fields: `relationId`, `dbNumber`, `originDbName`.

## Verification

- `dotnet build` exits 0 with 0 warnings, 0 errors
- `[BsonIgnore]` on line immediately before `RelationIdNameMap` in Common.cs (line 94)
- `GetListOfDatablocks` at line 288, `alarmThread.Start()` at line 307 — correct order confirmed
- `srv.RelationIdNameMap.TryGetValue(relationId, out var dbName) ? dbName : ""` in AlarmThread.cs

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — `originDbName` is wired end-to-end: map populated at startup from live PLC browse, value resolved at alarm write time. No hardcoded empty values flow to output (empty string only appears when DB is genuinely not in map).

## Self-Check: PASSED

Files exist:
- json-scada/src/S7CommPlusClient/Common.cs — FOUND, RelationIdNameMap field present
- json-scada/src/S7CommPlusClient/Program.cs — FOUND, GetListOfDatablocks call present
- json-scada/src/S7CommPlusClient/AlarmThread.cs — FOUND, originDbName field present

Commits in json-scada submodule:
- 7a7c9104 — feat(06-01): add RelationIdNameMap field and GetListOfDatablocks browse call
- 5d43a6aa — feat(06-01): add originDbName field to BuildAlarmDocument

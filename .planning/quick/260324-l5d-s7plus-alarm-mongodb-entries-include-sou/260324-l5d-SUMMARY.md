---
phase: quick-260324-l5d
plan: 01
subsystem: s7plus-alarms
tags: [mongodb, alarm-documents, connectionName, vue, cleanup]
dependency_graph:
  requires: []
  provides: [connectionName-in-mongo, viewer-reads-connectionName, cleanup-fetchConnectionNames]
  affects: [json-scada/src/S7CommPlusClient/AlarmThread.cs, json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue]
tech_stack:
  added: []
  patterns: [self-contained alarm documents, direct field read from MongoDB document]
key_files:
  created: []
  modified:
    - json-scada/src/S7CommPlusClient/AlarmThread.cs
    - json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue
decisions:
  - connectionName written at alarm write-time from srv.name ?? "" — no extra lookup needed
  - Viewer reads field directly from alarm doc — eliminates secondary listProtocolConnections API call
  - Old records without connectionName fall back to raw connectionId display (defensive)
metrics:
  duration: ~10 min
  completed: 2026-03-24
  tasks_completed: 2
  files_modified: 2
---

# Quick Task 260324-l5d: S7Plus Alarm MongoDB entries include connectionName — Summary

**One-liner:** Added `connectionName` (from `srv.name`) to every alarm BsonDocument at write time and removed the secondary `listProtocolConnections` API call from the Alarms Viewer.

## Tasks Completed

| # | Task | Commit | Files |
|---|------|--------|-------|
| 1 | Add connectionName field to BuildAlarmDocument in AlarmThread.cs | `4893f435` (submodule), `718529c` (parent) | AlarmThread.cs |
| 2 | Simplify Alarms Viewer to read connectionName from alarm document | `e6a43a13` (submodule), `32cd1b0` (parent) | S7PlusAlarmsViewerPage.vue |

## Changes Made

### Task 1 — AlarmThread.cs

Added one line to `BuildAlarmDocument()` immediately after the `connectionId` field:

```csharp
{ "connectionName",    srv.name ?? "" },
```

`srv` is the `S7CP_connection` already in scope at the call site. The null-coalescing `?? ""` guards against any edge case where `srv.name` is null.

### Task 2 — S7PlusAlarmsViewerPage.vue

Removed:
- `const connectionNameMap = ref({})` — ref holding the id-to-name map
- `fetchConnectionNames()` async function (17 lines) — secondary API call to `listProtocolConnections`
- `const connectionName = (id) => ...` — lookup helper

Updated:
- Template slot for `item.connectionId` column: now renders `{{ item.connectionName || item.connectionId }}` — direct read with numeric fallback
- `onMounted`: changed `await Promise.all([fetchAlarms(), fetchConnectionNames()])` to `await fetchAlarms()`

## Verification

```
1. grep "connectionName" AlarmThread.cs
   → { "connectionName",    srv.name ?? "" },   ✓

2. grep -c "fetchConnectionNames" S7PlusAlarmsViewerPage.vue
   → 0                                           ✓

3. grep -c "connectionNameMap" S7PlusAlarmsViewerPage.vue
   → 0                                           ✓

4. grep "item.connectionName" S7PlusAlarmsViewerPage.vue
   → {{ item.connectionName || item.connectionId }}  ✓
```

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — alarm documents are written with the live `srv.name` value at runtime. The viewer reads the stored field directly with no placeholder data.

## Self-Check: PASSED

- `json-scada/src/S7CommPlusClient/AlarmThread.cs` modified and committed — 4893f435 / 718529c
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` modified and committed — e6a43a13 / 32cd1b0
- Summary file created at `.planning/quick/260324-l5d-s7plus-alarm-mongodb-entries-include-sou/260324-l5d-SUMMARY.md`

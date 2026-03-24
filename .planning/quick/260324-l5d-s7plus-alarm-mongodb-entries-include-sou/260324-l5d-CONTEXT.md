---
name: Quick Task 260324-l5d Context
description: Decisions for adding connectionName to S7Plus alarm MongoDB documents
type: project
---

# Quick Task 260324-l5d: S7Plus Alarm MongoDB connectionName — Context

**Gathered:** 2026-03-24
**Status:** Ready for planning

<domain>
## Task Boundary

Add `connectionName` (and `connectionId` already exists) to every S7Plus alarm document written to MongoDB. Update the Alarms Viewer Source column to read `connectionName` directly from the alarm doc.

</domain>

<decisions>
## Implementation Decisions

### Write point
- Add `connectionName` in `BuildAlarmDocument()` in `AlarmThread.cs` (C# driver)
- `srv.name` is already available at that call site — no extra lookup needed
- Field name: `connectionName`

### Existing alarm records
- Old records will be cleared by the user — no migration needed
- Viewer should still fall back to displaying raw `connectionId` number if `connectionName` is absent (defensive, for any edge case)

### Frontend fetchConnectionNames cleanup
- Remove `fetchConnectionNames()` and `connectionNameMap` entirely from `S7PlusAlarmsViewerPage.vue`
- Source column reads `item.connectionName` directly, falling back to `item.connectionId` if absent
- No secondary API call on page load

</decisions>

<specifics>
## Specific Ideas

- `AlarmThread.cs` `BuildAlarmDocument()` already receives `S7CP_connection srv` — add `{ "connectionName", srv.name }` alongside the existing `{ "connectionId", srv.protocolConnectionNumber }` line
- Vue template slot for `item.connectionId` column should become: `{{ item.connectionName || item.connectionId }}`
- The header key stays `connectionId` for sorting continuity — only display changes

</specifics>

<canonical_refs>
## Canonical References

- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — BuildAlarmDocument(), line ~199–260
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — connectionNameMap, fetchConnectionNames, template slot
</canonical_refs>

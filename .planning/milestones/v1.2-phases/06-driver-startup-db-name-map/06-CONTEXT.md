# Phase 6: Driver — Startup DB Name Map - Context

**Gathered:** 2026-03-23
**Status:** Ready for planning

<domain>
## Phase Boundary

Build a `RelationIdNameMap` (`Dictionary<uint, string>`) on `S7CP_connection` by calling `GetListOfDatablocks` on `srv.connection` before `AlarmThread.Start()`. Use the map in `BuildAlarmDocument()` to populate an `originDbName` field (string) on every new alarm event document. This is a pure C# driver change — no UI changes, no API changes, no new connections. Phase 7 exposes `originDbName` via the API; Phase 8 displays it in the viewer.

</domain>

<decisions>
## Implementation Decisions

### Map Lifecycle
- **D-01:** Add `RelationIdNameMap` as `Dictionary<uint, string>` to `S7CP_connection` (in `Common.cs`), decorated with `[BsonIgnore]`. Initialized as `new Dictionary<uint, string>()` (empty, not null).
- **D-02:** The map **refreshes on every reconnect** — the browse call lives in the "just connected" block of `Program.cs` before `alarmThread.Start()`, and the reconnect loop naturally re-runs it. No guard flag needed. This picks up renamed or newly added datablocks after a PLC program change.

### Browse Call Placement
- **D-03:** Call `GetListOfDatablocks` on `srv.connection` immediately after `srv.isConnected = true`, **before** `srv.alarmThread.Start()`. The call must complete synchronously before the alarm thread starts. Existing `autoCreateTags` browse (which happens after `alarmThread.Start()`) is unaffected.
- **D-04:** Browse runs on `srv.connection` (the tag connection), not on `alarmConn` (the alarm-only connection). No PDU interleaving with alarm subscription.

### Browse Failure Handling
- **D-05:** On browse failure (non-zero return from `GetListOfDatablocks`), log at **Warning** level so operators know origin names won't appear, then continue with an empty map. Alarm subscription starts normally regardless. No retry — failure on this path should be immediately visible.

### `originDbName` Field
- **D-06:** `originDbName` is **always written** to every alarm document, even when the map is empty or lookup fails. Value is the DB name string from the map keyed by `relationId`; falls back to `""` (empty string) when the key is not found. Consistent schema for Phase 7/8 — no null checks needed downstream.

### Claude's Discretion
- Map key type: researcher to confirm whether `DatablockInfo.db_block_relid` exactly matches the `uint relationId` from Phase 5 (`(uint)(dai.CpuAlarmId >> 32)`). Almost certainly the same value — the driver library's `RelationId` field is the source for both.
- Field position of `originDbName` in the `BsonDocument` literal — after `dbNumber` to keep origin fields grouped.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Driver Implementation (S7CommPlusClient)
- `json-scada/src/S7CommPlusClient/Common.cs` — `S7CP_connection` class (lines 44–98); add `RelationIdNameMap` here following the `[BsonIgnore]` pattern of `AddressCache` and `SoftdatatypeCache`
- `json-scada/src/S7CommPlusClient/Program.cs` — lines 282–296: the "just connected" block where the browse call must be inserted before `alarmThread.Start()`
- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — `BuildAlarmDocument()` (lines 173–228); add `originDbName` lookup here using `srv.RelationIdNameMap`

### Driver Library (S7CommPlusDriver)
- `S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs` — `GetListOfDatablocks` (line 1191) and `DatablockInfo` class (line 1183): `db_block_relid` (full `UInt32` RelationId) → `db_name` (TIA Portal datablock name string)
- `S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs` — lines 1244–1254: confirms `relid = ob.RelationId`; `num = relid & 0xffff` (db number); the full `relid` is the map key

### Phase 5 Context
- `.planning/phases/05-driver-relationid-fields/05-CONTEXT.md` — D-01/D-03: `uint relationId = (uint)(dai.CpuAlarmId >> 32)` is the value to look up in `RelationIdNameMap`

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `GetListOfDatablocks(out List<DatablockInfo> dbInfoList)` on `S7CommPlusConnection`: already fully implemented, returns `DatablockInfo` records with `db_block_relid` and `db_name`
- `[BsonIgnore]` pattern for runtime-only fields on `S7CP_connection`: established in `Common.cs` (lines 80–97) — `AddressCache`, `SoftdatatypeCache` are direct analogues
- `srv.connection` field (line 81 of `Common.cs`): is of type `S7CommPlusDriver.S7CommPlusConnection` — exactly the type that has `GetListOfDatablocks`

### Established Patterns
- `BsonDocument` string field: existing fields like `alarmText`, `alarmClassName` use plain `string` values in the literal — same for `originDbName`
- Warning log pattern: `Log(srv.name + " - message", ...)` — check existing log calls for format consistency

### Integration Points
- Browse insert point: `Program.cs` line 284 (after `srv.isConnected = true`) through line 288 (before `srv.alarmThread.Start()`)
- `BuildAlarmDocument` receives `srv` as second parameter — `RelationIdNameMap` is directly accessible as `srv.RelationIdNameMap`

</code_context>

<specifics>
## Specific Ideas

No specific requirements beyond the decisions above.

</specifics>

<deferred>
## Deferred Ideas

- FB type name column (requires `GetTypeInformation` browse per DB; high complexity) — Future Requirements
- Auto-refresh of map on a timer (separate from reconnect) — not requested; reconnect refresh is sufficient

</deferred>

---

*Phase: 06-driver-startup-db-name-map*
*Context gathered: 2026-03-23*

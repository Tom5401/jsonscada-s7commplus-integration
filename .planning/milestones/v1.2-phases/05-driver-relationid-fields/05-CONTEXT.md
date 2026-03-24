# Phase 5: Driver — RelationId Fields - Context

**Gathered:** 2026-03-23
**Status:** Ready for planning

<domain>
## Phase Boundary

Add `relationId` (BsonInt64) and `dbNumber` (uint stored as BsonInt32 or BsonInt64) to every new alarm event document written by `BuildAlarmDocument()` in `AlarmThread.cs`. This is a pure C# driver change — no PLC browse calls, no new connections, no UI changes. Phase 6 will populate `originDbName` using a startup lookup map; Phase 5 only stores the raw numeric identifiers.

</domain>

<decisions>
## Implementation Decisions

### RelationId Extraction
- **D-01:** Extract `relationId` by deriving from `dai.CpuAlarmId >> 32` inside `BuildAlarmDocument`. No changes to the call site signature — `BuildAlarmDocument(AlarmsDai dai, S7CP_connection srv)` stays as-is. This keeps the function self-contained.
  - Formula: `uint relationId = (uint)(dai.CpuAlarmId >> 32);`

### dbNumber Formula (Roadmap Correction)
- **D-02:** `dbNumber` is extracted from the **lower 16 bits** of `relationId`: `uint dbNumber = relationId & 0xFFFF;`
  - **Correction from roadmap:** ORIGIN-02 states `relationId >> 16 & 0xFFFF` but that extracts the area code (e.g., `0x8a0e`), not the DB number. Confirmed in `S7CommPlusConnection.cs:1247` — `UInt32 num = relid & 0xffff`.

### BSON Storage Types
- **D-03:** Store `relationId` as `BsonInt64` (not Int32). RelationId values like `0x8a0e0005 = 2,317,140,997` exceed Int32 max and would be silently truncated. Use `new BsonInt64((long)relationId)`.
- **D-04:** Store `dbNumber` as `BsonInt32` — DB numbers fit within Int32 range (max 65535).

### Phase Scope
- **D-05:** Phase 5 adds `relationId` and `dbNumber` to `BuildAlarmDocument` only. The `RelationIdNameMap` field on `S7CP_connection` and the `originDbName` field are deferred to Phase 6. Phase 5 does NOT add `originDbName: ""` as a placeholder — Phase 6 adds it when it has meaning.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Driver Implementation
- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — `BuildAlarmDocument()` function (lines 173-228); this is where the two new fields are added
- `json-scada/src/S7CommPlusClient/Common.cs` — `S7CP_connection` class with `[BsonIgnore]` pattern; `BsonInt64` usage example in `SdValueToBson`
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsDai.cs` — `CpuAlarmId` field (line 24, read from `DAI_CPUAlarmID` attribute)
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/BrowseAlarms.cs` — line 414: `CpuAlarmId = ((ulong)(RelationId) << 32) | ((ulong)(Alid) << 16)` — confirms `relationId = (uint)(CpuAlarmId >> 32)`
- `S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs` — lines 1245-1254: `GetListOfDatablocks` logic confirming `db_number = relid & 0xffff`

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `BsonInt64` usage: already used in `SdValueToBson()` (`AlarmThread.cs:271`) — same pattern applies for `relationId`
- `BuildAlarmDocument()` pattern: simple `BsonDocument` constructor with named fields; add two more entries to the existing list

### Established Patterns
- BSON field ordering: new fields should be added after existing numeric fields (`priority`, `alarmClass`, `alarmClassName`, `groupId`, `allStatesInfo`) to keep logical grouping
- All existing integer fields use `(int)` cast to `BsonInt32` except when overflow is possible (then BsonInt64 is used — see `SdValueToBson`)

### Integration Points
- `AlarmThread()` loop calls `BuildAlarmDocument(dai, srv)` then `InsertOneAsync` — no change to the call site
- No changes to `S7CP_connection`, `AlarmThread()` control flow, or MongoDB collection schema (additive only)

</code_context>

<specifics>
## Specific Ideas

No specific requirements beyond the decisions above — open to standard approaches for field positioning within the BsonDocument.

</specifics>

<deferred>
## Deferred Ideas

- `RelationIdNameMap Dictionary<uint, string>` field on `S7CP_connection` — Phase 6
- `originDbName` field in alarm documents — Phase 6
- Phase scope boundary discussion (adding empty map field in Phase 5 vs 6) — user did not select this for discussion; planner may decide

</deferred>

---

*Phase: 05-driver-relationid-fields*
*Context gathered: 2026-03-23*

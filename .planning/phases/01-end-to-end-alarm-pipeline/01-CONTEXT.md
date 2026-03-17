# Phase 1: End-to-End Alarm Pipeline - Context

**Gathered:** 2026-03-17
**Status:** Ready for planning

<domain>
## Phase Boundary

Fix all 5 known protocol bugs in S7CommPlusDriver, build the alarm subscription lifecycle (create, credit-limit replenishment, delete), stand up a dedicated alarm receive thread with a thread-safe queue to MongoDB writer, and write alarm event documents into the `s7plusAlarmEvents` MongoDB collection — while tag polling in the existing connection thread continues unaffected. The PoC is complete when a real PLC alarm appears in `s7plusAlarmEvents` with correct metadata.

</domain>

<decisions>
## Implementation Decisions

### Bug fix placement
- All 5 bugs (BUG-01 to BUG-05) are fixed **directly in S7CommPlusDriver** (the local fork of thomas-v2's library). S7CommPlusClient is not used to work around library bugs.
- Fixes are **surgical** — change only the wrong values, no surrounding refactor.
- **Log lines are added** around alarm lifecycle calls (create, credit-limit replenish, delete) to aid field debugging. This is the one intentional deviation from pure surgical.
- BUG-03 (NotImplementedException from unknown PDU types): exception is **caught at the alarm receive loop level in S7CommPlusClient** — catch, log, continue. The library itself is not changed for this bug.
- BUG-04 (null dai from ack-only notifications): **log + skip** — log a debug-level message ("ack-only notification, skipping") and continue the loop. No MongoDB write for ack-only PDUs.
- BUG-05 (RelationId collision): alarm subscription uses `0x7fffc002` as a hardcoded distinct constant (tag subscription uses `0x7fffc001`).

### Alarm thread design
- **Dedicated alarm thread per connection**, separate from the existing `connectionThread` that handles tag read/write. Mirrors the existing pattern (`thrRedundancy`, `thrMongoCmd`, `connectionThread` each being their own `Thread`).
- Thread-safe queue (`ConcurrentQueue`) passes `AlarmsDai` + `Notification` metadata from alarm receiver to MongoDB writer.
- MongoDB writer for alarms runs on the **alarm thread itself** (not the shared `ProcessMongo` task) — keeps alarm writes independent of tag bulk-write performance.

### MongoDB document schema
- Collection name: `s7plusAlarmEvents`
- Mandatory v1 fields: `cpuAlarmId`, `alarmState` ("Coming" / "Going"), `alarmText`, `timestamp` (PLC-side, UTC), `ackState`, `connectionId`, `createdAt`
- **Also include HmiInfo fields** — `priority`, `alarmClass`, `groupId` — since they are already parsed by the library at zero extra protocol cost. Gives more diagnostic value for the PoC.
- `allStatesInfo` stored as raw byte value (semantics partially unknown — stored but not interpreted).
- Write concern: **WriteConcern.W1** (acknowledged, as per MONGO-02).

### Alarm text language (LCID)
- **Hardcoded English (1033)** for the PoC. No configuration hook needed.

### Claude's Discretion
- Exact MongoDB index definition on `s7plusAlarmEvents` (if any)
- Alarm thread startup/shutdown sequencing relative to the tag connection thread
- Whether the alarm queue drains before shutdown or drops on disconnect

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements
- `.planning/REQUIREMENTS.md` — All 17 v1 requirements (BUG-01 to BUG-05, LIFE-01 to LIFE-03, DATA-01 to DATA-04, THRD-01 to THRD-02, MONGO-01 to MONGO-03). Traceability table maps each to Phase 1.

### Library alarm code (S7CommPlusDriver)
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHandler.cs` — AlarmSubscriptionCreate, TestWaitForAlarmNotifications, AlarmSubscriptionDelete. Contains all 5 bugs. Primary target for BUG-01 through BUG-05 fixes.
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsDai.cs` — AlarmsDai class and FromNotificationObject factory. Returns null for ack-only notifications (BUG-04 site). Fields: CpuAlarmId, AllStatesInfo, AlarmDomain, MessageType, SequenceCounter, AlarmTexts, HmiInfo, AsCgs.
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsAsCgs.cs` — Alarm state struct: SubtypeId (Coming=2673 / Going=2677), Timestamp, AckTimestamp, AssociatedValues.
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHmiInfo.cs` — HmiInfo fields: Priority, AlarmClass, GroupId, Flags, ClientAlarmId.

### Client driver (S7CommPlusClient)
- `json-scada/src/S7CommPlusClient/Program.cs` — Main entry point, ConnectionThread, thread spawning pattern, ConcurrentQueue<S7CPValue> DataQueue, existing thread architecture to mirror.
- `json-scada/src/S7CommPlusClient/MongoUpdate.cs` — Existing MongoDB bulk-write pattern for tag data. Alarm writes should be modelled on this but use InsertOneModel, not UpdateOneModel.
- `json-scada/src/S7CommPlusClient/Common.cs` — S7CP_connection class (connection config), shared constants, Log() helper.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ConcurrentQueue<S7CPValue> DataQueue` (Program.cs): Pattern to copy for alarm event queue — replace S7CPValue with a new AlarmEvent type.
- `Log()` helper (Common.cs): Use for all alarm lifecycle and debug log lines.
- `S7CP_connection` (Common.cs): Connection config object passed to alarm thread — provides connectionId, name, and MongoDB credentials.
- `MongoDatabase` (Program.cs): Shared IMongoDatabase instance — alarm writer gets a handle to `s7plusAlarmEvents` collection from this.

### Established Patterns
- One `Thread` per concern (redundancy, command, connection) — alarm receive thread follows the same pattern, not a Task.
- `connectionThread` spawned per `S7CP_connection` entry in the config list — alarm thread is spawned alongside it, bound to the same connection.
- MongoDB accessed via `MongoDatabase.GetCollection<>()` — alarm collection obtained the same way.
- `BulkWriteAsync` used for tag data writes; alarm events use `InsertOneAsync` or small `InsertManyAsync` (events are infrequent compared to tag polling).

### Integration Points
- `ConnectionThread` in Program.cs: After successful PLC connect + tag subscription, add alarm subscription creation and start alarm thread.
- On disconnect/shutdown: call `AlarmSubscriptionDelete()` before closing the connection, then signal alarm thread to stop.
- `TestWaitForAlarmNotifications` in AlarmsHandler.cs: Replace with a production receive loop in S7CommPlusClient that runs indefinitely until cancelled, rather than a fixed-count loop.

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-end-to-end-alarm-pipeline*
*Context gathered: 2026-03-17*

# Project Research Summary

**Project:** S7CommPlus Alarm Subscription PoC — json-scada C#/.NET driver
**Domain:** Industrial SCADA protocol driver extension — S7CommPlus native alarm subscription to MongoDB
**Researched:** 2026-03-17
**Confidence:** HIGH

## Executive Summary

This project extends an existing, running C#/.NET SCADA protocol driver (`S7CommPlusClient`) to receive native S7CommPlus alarm subscriptions from Siemens S7-1200/S7-1500 PLCs and persist alarm events to MongoDB. The codebase already contains all required dependencies — no new NuGet packages are needed. The S7CommPlusDriver library already implements the full alarm subscription API (`AlarmSubscriptionCreate`, `WaitForNewS7plusReceived`, `AlarmSubscriptionDelete`), and the client already uses MongoDB.Driver 3.4.2 for tag data writes. The integration is additive: two new source files plus minimal extensions to existing files, following patterns that already exist in the codebase (`DataQueue`/`ProcessMongo` for tag data translate directly to `AlarmQueue`/`ProcessMongoAlarms` for alarm events).

The recommended approach is to follow the existing producer/consumer threading model with one critical constraint: the alarm notification receive loop (`WaitForNewS7plusReceived`) is a blocking synchronous call and must run on a dedicated thread separate from the tag read cycle. The subscription is created from `connectionThread` immediately after connect, then a dedicated `alarmSubscriptionThread` takes over notification consumption and enqueues events to a `ConcurrentQueue<S7CPAlarmEvent>`. A separate async task drains the queue and inserts into a new `s7plusAlarmEvents` MongoDB collection using acknowledged writes (`WriteConcern.W1`).

The primary risk is that the S7CommPlusDriver alarming code is explicitly marked by its author as test/experimental scaffolding, not production-ready integration code. Six concrete bugs were found through direct code inspection that would cause silent data loss or crashes in production. All six must be addressed in Phase 1 before any functional testing. The most critical: `SubscriptionSetCreditLimit` sends credit renewal to the wrong PLC object ID, silently stopping alarm delivery after 10 events. This is a one-method fix but must not be overlooked — the failure mode produces no error and is easy to miss in short demos.

## Key Findings

### Recommended Stack

No new dependencies are required. The entire implementation uses types already present in the codebase: `MongoDB.Driver` 3.4.2 (already pinned in `S7CommPlusClient.csproj`), `MongoDB.Bson` 3.4.2, `System.Collections.Concurrent` (BCL), `System.Threading` (BCL), and the S7CommPlusDriver project reference (already wired). The project targets `net8.0` x64, and any AnyCPU or x86 build attempt will fail due to OpenSSL native DLLs in the driver.

**Core technologies:**
- **.NET 8 / C#:** Runtime and language — already used by both projects, no change needed
- **S7CommPlusDriver (project ref):** PLC protocol implementation — the only C# library with S7CommPlus alarm subscription support; no NuGet alternative exists
- **MongoDB.Driver 3.4.2:** Persistence — already in use for all tag data writes; pin at this version for the PoC
- **`ConcurrentQueue<T>` (BCL):** Producer/consumer bridge — same pattern as the existing `DataQueue`; no extra package needed
- **Dedicated `Thread` (BCL):** Alarm notification loop — required because `WaitForNewS7plusReceived` is a synchronous blocking call; must not run on `Task`/`async` or the existing read cycle thread

### Expected Features

The S7CommPlus protocol delivers rich per-alarm data in each notification. The PoC must deliver the seven table-stakes features to demonstrate end-to-end value; the v1.x completeness fields are low-cost additions once the core pipeline is confirmed working.

**Must have (table stakes — PoC v1):**
- Subscription lifecycle (create, credit replenishment loop, delete on disconnect) — without this the subscription silently stops after 10 events
- Alarm state (coming/going) from `SubtypeId` — the fundamental alarm signal
- PLC-side timestamp from `AsCgs.Timestamp` — operators need when the PLC saw the event
- Driver-receive timestamp from `Notification.Add1Timestamp` — audit latency trail
- Alarm identity (`CpuAlarmId`) — correlate coming/going pairs and dedup
- Primary alarm text (inline, `AlarmTexts.AlarmText`) — delivered in each notification when `SendAlarmTexts=true`; no separate browse needed
- MongoDB insert per notification into `s7plusAlarmEvents` collection — the stated deliverable

**Should have (v1.x completeness pass after PoC validated):**
- Acknowledgement state (passive read of `AckTimestamp`) — low cost, high operator value
- Priority and alarm class (`HmiInfo.Priority`, `HmiInfo.AlarmClass`) — enables downstream filtering
- Alarm domain — distinguishes system vs user program alarms
- Sequence counter — gap detection and ordering
- Associated values SD_1..SD_10 — process values at alarm time (requires testing with a real alarm that has associated values)
- Infotext field — zero-cost supplement to primary alarm text

**Defer (v2+):**
- Acknowledgement write-back to PLC — requires separate `SetVariable` command and auth; out of scope per PROJECT.md
- Multi-language text storage — requires multiple subscription objects or language-keyed sub-documents
- Additional texts 1-9 storage — marginal value; bloats every document
- Alarm catalog pre-fetch (`ExploreASAlarms`) — adds startup latency; inline text delivery is sufficient

### Architecture Approach

The integration adds two new files (`AlarmSubscription.cs` and `MongoAlarms.cs`) and minimal extensions to `Common.cs` and `Program.cs`, all as `partial class MainClass` additions matching the existing code structure. The alarm path is a parallel mirror of the existing tag data path: a dedicated thread produces events into a `ConcurrentQueue`, and a separate async `Task` drains the queue and writes to MongoDB. The alarm subscription is created from within `connectionThread` after a successful connect (before the read cycle starts), then transferred to a dedicated `alarmSubscriptionThread` that owns the blocking notification loop. Both threads share the `S7CommPlusConnection` instance but operate on different PDU types; subscription creation must complete before concurrent PDU traffic begins.

**Major components:**
1. `alarmSubscriptionThread` (new, per connection) — manages alarm subscription lifecycle; blocks on `WaitForNewS7plusReceived`; decodes `AlarmsDai`; manages credit limit renewal; enqueues `S7CPAlarmEvent` to `AlarmQueue`
2. `AlarmQueue` (new static field, `ConcurrentQueue<S7CPAlarmEvent>`) — decouples PLC timing from MongoDB latency; prevents credit-limit starvation due to DB slowness
3. `S7CPAlarmEvent` (new DTO) — carries all alarm fields from one notification; defined in `Common.cs`
4. `ProcessMongoAlarms` (new async `Task`, `MongoAlarms.cs`) — drains `AlarmQueue`; inserts to `s7plusAlarmEvents` with `WriteConcern.W1`; mirrors `ProcessMongo` pattern
5. `connectionThread` (existing, unchanged except for alarm startup call) — creates the subscription after connect, starts `alarmSubscriptionThread`, detects disconnect and signals alarm thread to stop

**Build order (from architecture research):**
1. `S7CPAlarmEvent` DTO + `AlarmQueue` field (no PLC needed)
2. `ProcessMongoAlarms` task with `MongoAlarms.cs` (testable with dummy events before PLC work)
3. `AlarmSubscription.cs` — the protocol-dependent piece (requires live PLC)
4. Wire into `Program.cs` — integration glue
5. End-to-end test: trigger alarm on PLC, verify document in `s7plusAlarmEvents`

### Critical Pitfalls

All six pitfalls below are Phase 1 issues — they must be resolved before functional testing, not discovered during it.

1. **`SubscriptionSetCreditLimit` sends renewal to the wrong PLC object ID** — implement a dedicated `AlarmSubscriptionSetCreditLimit` method that uses `m_AlarmSubscriptionObjectId` instead of `m_SubscriptionObjectId`; failure mode is silent: alarm delivery stops after 10 events with no error logged
2. **Blocking receive loop conflicts with the tag read cycle** — run `WaitForNewS7plusReceived` on a dedicated thread; never insert alarm waiting into `connectionThread`'s read loop; these two threads must be strictly separated
3. **`AlarmSubscriptionDelete` leaks PLC subscription slots across restarts** — log `PlcSubscriptionsFree` on connect; verify slot count recovers after each disconnect cycle; PLC has a finite number of subscription slots
4. **`RelationId` collision between tag and alarm subscriptions** — assign a distinct initial `RelationId` value for the alarm subscription (e.g., `0x7fffc002`); the existing code initializes both to the same value, which has never been tested in combined use
5. **`AlarmsDai.FromNotificationObject` returns null for ack-only notifications** — always null-check the return value before accessing any field; the test caller (`TestWaitForAlarmNotifications`) does not null-check and will `NullReferenceException` on any operator acknowledgement event
6. **`Notification.Deserialize` throws `NotImplementedException` for unexpected PDU codes** — wrap all deserialization in try/catch in the alarm loop; log and skip unexpected PDUs rather than letting the exception kill the alarm thread

## Implications for Roadmap

Based on combined research, two phases are sufficient for the PoC. The build order from architecture research directly maps to a phase structure.

### Phase 1: Alarm Subscription Wire-Up and End-to-End Pipeline

**Rationale:** All six critical pitfalls are Phase 1 issues. The alarming code in S7CommPlusDriver is test scaffolding, not integration-ready code. Before a single alarm event can reliably land in MongoDB, the bugs in credit-limit renewal, thread isolation, subscription deletion, RelationId assignment, null guards, and exception handling must all be addressed. These are prerequisites, not polish. The build-order dependency (DTO first, then MongoDB writer, then subscription thread, then wiring) reinforces a single phase with a clear internal sequence.

**Delivers:** A running driver that receives S7CommPlus alarm notifications from a live PLC, processes Coming and Going events, and inserts structured documents into a `s7plusAlarmEvents` MongoDB collection with correct PLC timestamps, alarm text, alarm identity, and alarm state. The driver continues tag reads at normal poll rate while alarms are flowing.

**Addresses (from FEATURES.md):** All seven table-stakes features — subscription lifecycle, alarm state, PLC timestamp, driver timestamp, alarm identity, primary alarm text, MongoDB insert.

**Avoids (from PITFALLS.md):** All six critical pitfalls — credit limit bug, thread isolation, slot leak, RelationId collision, null guard, exception hygiene.

**Stack used:** `ConcurrentQueue<S7CPAlarmEvent>`, dedicated `Thread`, `InsertOneAsync` with `WriteConcern.W1`, MongoDB.Driver 3.4.2.

**Research flag:** No additional research needed. All patterns are well-documented in the existing codebase. Implementation is a code translation exercise from test scaffolding to production patterns, not a design discovery exercise.

### Phase 2: Completeness Pass and Validation

**Rationale:** Once Phase 1 is validated with a live PLC (all 7 table-stakes fields confirmed in MongoDB, tag polling rate unaffected, subscription survives 3 connect/disconnect cycles), the v1.x enrichment fields can be added safely. These are trivial additions — all fields are already parsed by the existing alarm library code; they simply need to be included in the `S7CPAlarmEvent` DTO and written to the document.

**Delivers:** Enriched alarm documents with acknowledgement state, priority, alarm class, alarm domain, sequence counter, infotext, and associated values. MongoDB index recommendations applied. UTC timestamp consistency verified across all stored fields.

**Addresses (from FEATURES.md):** All v1.x should-have features.

**Avoids (from PITFALLS.md):** Server timestamp UTC inconsistency (use `DateTime.UtcNow` not `DateTime.Now`; verify both `plcTimestamp` and `serverTimestamp` are UTC in stored documents).

**Research flag:** No additional research needed. All v1.x fields are already decoded by the library. This phase is purely additive.

### Phase Ordering Rationale

- Phase 1 before Phase 2 because the v1.x fields have zero value if the core pipeline is broken. Confirming end-to-end reliability before enriching is the only sensible order.
- The internal build order within Phase 1 (DTO, MongoDB writer, subscription thread, wiring) is dictated by the dependency graph: you cannot write to MongoDB without the DTO; you cannot test the subscription thread without the MongoDB writer already working with dummy data.
- All pitfalls cluster in Phase 1, which means no known surprises in Phase 2. Phase 2 is low-risk.

### Research Flags

Phases needing deeper research during planning:
- Neither phase requires additional research. All implementation details are grounded in direct source inspection with HIGH confidence.

Phases with standard patterns (skip research-phase):
- **Phase 1:** All patterns mirror existing code in the same codebase. The only novel element is fixing bugs in the experimental alarm library — the fixes are straightforward method-level changes.
- **Phase 2:** Entirely additive field additions. No architectural decisions.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All dependencies confirmed from `.csproj` files and direct code inspection; no external research needed |
| Features | HIGH | Derived directly from S7CommPlusDriver source code; every field listed has been confirmed present in the parsed data model |
| Architecture | HIGH | Based on direct inspection of all four source files that will be modified; existing patterns are clear and well-established |
| Pitfalls | HIGH | All six pitfalls are grounded in specific line numbers in the source code, not conjecture; the author's own comment explicitly states this is test code |

**Overall confidence:** HIGH

### Gaps to Address

- **`AllStatesInfo` bitmask semantics are partially unknown:** Both `AlarmsDai.AllStatesInfo` and `AsCgs.AllStatesInfo` are parsed but their exact bit meanings are marked `TODO` in the source. Store as raw bytes; do not use for logic. This is documented and acceptable for the PoC — `SubtypeId` (Coming/Going) is the reliable state signal.
- **`m_AlarmSubscriptionRelationId` and `m_AlarmSubscriptionRefRelationId` semantics are uncertain:** The source marks these as `TODO Unknown`. The recommended mitigation (assign distinct initial value) is defensive. If `AlarmSubscriptionCreate` fails in testing, these values are the first thing to investigate.
- **Thread safety of `S7CommPlusConnection` for concurrent PDU access:** The architecture assumes that the tag read cycle and the alarm notification consumer can safely share one `S7CommPlusConnection` instance (operating on different PDU opcodes). The `m_ReceivedPDUs` queue uses `m_Mutex` for access, which supports this. However, this has never been tested in combined use — Phase 1 integration testing will be the first real-world validation of this assumption.

## Sources

### Primary (HIGH confidence — direct code inspection)
- `json-scada/src/S7CommPlusClient/Program.cs` — thread model, `DataQueue` pattern, `connectionThread` structure
- `json-scada/src/S7CommPlusClient/MongoUpdate.cs` — `ProcessMongo` async task, `ConcurrentQueue` drain, `BulkWriteAsync`, `WriteConcern.Unacknowledged`
- `json-scada/src/S7CommPlusClient/Common.cs` — `S7CP_connection`, `S7CPValue`, `rtFilt` types
- `json-scada/src/S7CommPlusClient/MongoCommands.cs` — `ProcessMongoCmd` change-stream pattern
- `S7CommPlusDriver/.../Alarming/AlarmsHandler.cs` — alarm subscription API, credit management, author's "this is test code" comment
- `S7CommPlusDriver/.../Alarming/AlarmsDai.cs` — notification data model, null return on ack-only notifications
- `S7CommPlusDriver/.../Alarming/AlarmsAsCgs.cs` — timestamps, ack timestamp, associated values, SubtypeId
- `S7CommPlusDriver/.../Alarming/AlarmsAlarmTexts.cs` — text fields, LCID-keyed sparse array
- `S7CommPlusDriver/.../Alarming/AlarmsHmiInfo.cs` — priority, alarm class, SyntaxId-gated fields
- `S7CommPlusDriver/.../Alarming/AlarmsAssociatedValues.cs` — SD_1..SD_10 typed process values
- `S7CommPlusDriver/.../Subscriptions/Subscription.cs` — `SubscriptionSetCreditLimit` uses `m_SubscriptionObjectId` (pitfall source)
- `S7CommPlusDriver/.../Core/Notification.cs` — `NotImplementedException` on unknown return codes, two-path timestamp parsing
- `S7CommPlusDriver/.../Core/CommRessources.cs` — `PlcSubscriptionsFree`, `PlcSubscriptionsMax`

### Secondary (MEDIUM confidence)
- [NuGet: MongoDB.Driver](https://www.nuget.org/packages/mongodb.driver) — 3.7.0 is latest stable (March 2026); 3.4.2 is project pin
- [MongoDB C# Driver Compatibility](https://www.mongodb.com/docs/drivers/csharp/current/compatibility/) — .NET 8 / MongoDB server version matrix

---
*Research completed: 2026-03-17*
*Ready for roadmap: yes*

# Architecture Research

**Domain:** S7CommPlus alarm subscription integration into json-scada C#/.NET protocol driver
**Researched:** 2026-03-17
**Confidence:** HIGH — based on direct code inspection of all relevant source files

## Standard Architecture

### System Overview

```
  S7-1200 / S7-1500 PLC
        |  (S7CommPlus TCP/IP)
        |
  ┌─────┴─────────────────────────────────────────────────────────┐
  │                    S7CommPlusClient process                    │
  │                                                               │
  │  ┌──────────────────┐    ┌───────────────────────────────┐   │
  │  │  connectionThread │    │  alarmSubscriptionThread      │   │
  │  │  (per connection) │    │  (NEW — per connection)       │   │
  │  │                  │    │                               │   │
  │  │  · Connect()     │    │  · AlarmSubscriptionCreate()  │   │
  │  │  · Browse tags   │    │  · WaitForNotification loop   │   │
  │  │  · PerformRead   │    │  · AlarmSubscriptionDelete()  │   │
  │  │    Cycle()       │    │  · Enqueue to AlarmQueue      │   │
  │  └────────┬─────────┘    └──────────────┬────────────────┘   │
  │           │                             │                     │
  │  ┌────────▼─────────┐    ┌──────────────▼────────────────┐   │
  │  │  DataQueue        │    │  AlarmQueue (NEW)             │   │
  │  │  ConcurrentQueue  │    │  ConcurrentQueue              │   │
  │  │  <S7CPValue>      │    │  <S7CPAlarmEvent>             │   │
  │  └────────┬─────────┘    └──────────────┬────────────────┘   │
  │           │                             │                     │
  │  ┌────────▼─────────┐    ┌──────────────▼────────────────┐   │
  │  │  ProcessMongo     │    │  ProcessMongoAlarms (NEW)     │   │
  │  │  Task (async)     │    │  Task (async)                 │   │
  │  │                  │    │                               │   │
  │  │  bulk writes →   │    │  InsertOne per event →        │   │
  │  │  realtimeData    │    │  s7plusAlarmEvents (NEW)      │   │
  │  └──────────────────┘    └───────────────────────────────┘   │
  │                                                               │
  │  ┌──────────────────┐    ┌───────────────────────────────┐   │
  │  │  ProcessRedun-   │    │  ProcessMongoCmd              │   │
  │  │  dancyMongo      │    │  (change stream, unchanged)   │   │
  │  └──────────────────┘    └───────────────────────────────┘   │
  └───────────────────────────────────────────────────────────────┘
        |                           |
  MongoDB: realtimeData        MongoDB: s7plusAlarmEvents (NEW)
  MongoDB: protocolConnections
  MongoDB: protocolDriverInstances
  MongoDB: commandsQueue
```

### Component Responsibilities

| Component | Responsibility | Where it lives |
|-----------|----------------|----------------|
| `connectionThread` | Maintain PLC TCP connection; run read cycle; owns `S7CommPlusConnection` instance | `Program.cs` — existing, unchanged |
| `alarmSubscriptionThread` (NEW) | Manage alarm subscription lifecycle on a connected PLC; block-wait for notifications; decode `AlarmsDai`; enqueue to `AlarmQueue` | New file: `AlarmSubscription.cs` |
| `DataQueue` | Decouples tag read results from MongoDB bulk writes | `Program.cs` — existing |
| `AlarmQueue` (NEW) | Decouples alarm notifications from MongoDB inserts; same pattern as `DataQueue` | `Program.cs` — add static field |
| `S7CPAlarmEvent` (NEW) | DTO carrying all alarm fields from a single notification | New class in `Common.cs` or `AlarmSubscription.cs` |
| `ProcessMongo` | Bulk-write tag values to `realtimeData` | `MongoUpdate.cs` — existing, unchanged |
| `ProcessMongoAlarms` (NEW) | Drain `AlarmQueue`; insert events into `s7plusAlarmEvents` collection | New file: `MongoAlarms.cs` |
| `ProcessMongoCmd` | Watch `commandsQueue` change stream; write values to PLC | `MongoCommands.cs` — existing, unchanged |
| `ProcessRedundancyMongo` | Primary/standby election | `Program.cs` — existing, unchanged |
| `S7CP_connection` (extend) | Add `alarmSubscriptionThread` field and `AlarmSubscriptionEnabled` config flag | `Common.cs` — minimal extension |
| `S7CommPlusConnection` | Provides `AlarmSubscriptionCreate()`, `WaitForNewS7plusReceived()`, `AlarmSubscriptionDelete()` | External library — read-only |

## Recommended Project Structure

```
json-scada/src/S7CommPlusClient/
├── Program.cs               # Startup, thread orchestration — add AlarmQueue, alarm thread launch
├── Common.cs                # Types — add S7CPAlarmEvent, extend S7CP_connection
├── MongoUpdate.cs           # Existing tag bulk-write task — unchanged
├── MongoCommands.cs         # Existing command change-stream task — unchanged
├── AlarmSubscription.cs     # NEW — alarm subscription thread logic
└── MongoAlarms.cs           # NEW — alarm event MongoDB insert task
```

### Structure Rationale

- **AlarmSubscription.cs** mirrors the pattern of `ConnectionThread` in `Program.cs` — one file per major driver concern. It is a `partial class MainClass` file.
- **MongoAlarms.cs** mirrors the pattern of `MongoUpdate.cs` — one file per MongoDB write concern. Keeps alarm persistence isolated so it can be changed without touching tag persistence.
- No new project or library is needed. All additions are partial class extensions of `MainClass`.

## Architectural Patterns

### Pattern 1: Dedicated Thread per PLC Connection for Alarm Subscription

**What:** Alarm notifications arrive as push messages on the same `S7CommPlusConnection` object used for reads. The existing `connectionThread` owns the connection and drives the read cycle. The alarm subscription thread must use the same connection object but cannot block it. The correct model is: `connectionThread` establishes the connection and sets `isConnected = true`; `alarmSubscriptionThread` spins waiting for `isConnected`, then calls `AlarmSubscriptionCreate()` and enters a blocking `WaitForNewS7plusReceived()` loop.

**When to use:** Always — this is the only viable approach given the S7CommPlusDriver API. `TestWaitForAlarmNotifications()` in `AlarmsHandler.cs` calls `WaitForNewS7plusReceived()` in a loop, which blocks the calling thread indefinitely.

**Trade-offs:** Two threads share one `S7CommPlusConnection` object. Read-cycle writes and alarm subscription receives operate on different PDU types, but thread safety of the underlying transport must be confirmed (see Pitfalls). For the PoC, a simple approach is: alarm subscription thread waits for `isConnected`, then suspends `connectionThread`'s read cycle temporarily while creating the subscription, then both run concurrently. Alternatively, create the subscription from `connectionThread` after the `isConnected` flag is set, then launch the alarm thread.

**Recommended approach for PoC:** Create the subscription from within `connectionThread` immediately after connection is established (before starting the read cycle). Then start a separate `alarmSubscriptionThread` that calls only `WaitForNewS7plusReceived()` in a loop, decoding notifications and enqueuing to `AlarmQueue`. This avoids concurrent subscription setup and keeps thread interaction minimal.

**Example:**
```csharp
// In ConnectionThread, after srv.isConnected = true:
if (srv.alarmSubscriptionEnabled)
{
    int alarmRes = srv.connection.AlarmSubscriptionCreate();
    if (alarmRes == 0)
    {
        srv.alarmSubscriptionThread = new Thread(() => AlarmSubscriptionThread(srv));
        srv.alarmSubscriptionThread.Start();
    }
}
```

### Pattern 2: ConcurrentQueue Decoupling (same as tag data path)

**What:** The alarm subscription thread enqueues `S7CPAlarmEvent` objects to a static `AlarmQueue`. A separate async Task (`ProcessMongoAlarms`) drains the queue and writes to MongoDB. This is identical to how `DataQueue` and `ProcessMongo` work for tag values.

**When to use:** Whenever a producer thread must not block on MongoDB latency. The PLC's credit-limit mechanism (see credit refresh in `TestWaitForAlarmNotifications`) requires the notification consumer to respond quickly; parking work in a queue keeps the consumer responsive.

**Trade-offs:** Adds memory buffering. For PoC this is fine. The existing `DataBufferLimit = 50000` constant can serve as reference for alarm queue depth if needed.

**Example:**
```csharp
public static ConcurrentQueue<S7CPAlarmEvent> AlarmQueue = new ConcurrentQueue<S7CPAlarmEvent>();

// In AlarmSubscriptionThread:
AlarmQueue.Enqueue(new S7CPAlarmEvent { ... });

// In ProcessMongoAlarms:
while (AlarmQueue.TryDequeue(out var ev))
{
    await collection.InsertOneAsync(ev.ToBsonDocument());
}
```

### Pattern 3: Alarm Subscription Lifecycle Tied to Connection Lifecycle

**What:** When `connectionThread` detects a disconnect (exception or `isConnected = false`), it sets `srv.isConnected = false` and nulls `srv.connection`. The `alarmSubscriptionThread` must detect this and exit so it can be restarted cleanly on reconnect.

**When to use:** Always — the subscription object lives on the server side and is destroyed when the TCP connection drops. Attempting to receive notifications on a dead connection will return an error from `WaitForNewS7plusReceived()`.

**Trade-offs:** Requires a lifecycle signal between threads. Simplest PoC approach: `alarmSubscriptionThread` checks `srv.isConnected` in its loop; if false, calls `AlarmSubscriptionDelete()` (best-effort), exits, and lets `connectionThread` restart it after reconnection.

## Data Flow

### Alarm Notification Flow (new path)

```
PLC sends alarm notification (push, no poll)
    |
    v
S7CommPlusConnection.WaitForNewS7plusReceived()
  [blocks alarm subscription thread until PDU arrives]
    |
    v
Notification.DeserializeFromPdu(m_ReceivedPDU)
    |
    v
AlarmsDai.FromNotificationObject(noti.P2Objects[0], languageId)
  [decodes: CpuAlarmId, AlarmDomain, MessageType,
            AsCgs.SubtypeId (Coming/Going),
            AsCgs.Timestamp, AsCgs.AckTimestamp,
            AsCgs.AllStatesInfo,
            AlarmTexts.AlarmText, AlarmTexts.Infotext,
            HmiInfo.Priority, HmiInfo.AlarmClass]
    |
    v
Check noti.NotificationCreditTick >= threshold
  -> if yes: SubscriptionSetCreditLimit(newLimit)
    |
    v
Construct S7CPAlarmEvent DTO
    |
    v
AlarmQueue.Enqueue(event)
    |
    v
ProcessMongoAlarms Task drains AlarmQueue
    |
    v
collection.InsertOneAsync(event.ToBsonDocument())
  -> s7plusAlarmEvents collection in MongoDB
```

### Tag Data Flow (existing, unchanged reference)

```
PLC tag read (cyclic poll)
    -> DataQueue.Enqueue(S7CPValue)
    -> ProcessMongo bulk-writes to realtimeData
```

### Key Data Flows Summary

1. **Subscription setup:** `connectionThread` connects → calls `AlarmSubscriptionCreate()` → stores `m_AlarmSubscriptionObjectId` in connection → starts `alarmSubscriptionThread`.
2. **Notification receive:** `alarmSubscriptionThread` blocks on `WaitForNewS7plusReceived()` → decodes → enqueues → manages credit limit.
3. **MongoDB persistence:** `ProcessMongoAlarms` async task drains queue → inserts to `s7plusAlarmEvents` → logs throughput.
4. **Disconnect/reconnect:** `connectionThread` detects failure → sets `isConnected = false` → `alarmSubscriptionThread` detects → exits → `connectionThread` reconnects → restarts alarm subscription.

## MongoDB Collection Schema: `s7plusAlarmEvents`

One document per alarm notification (Coming or Going event). This is a time-series append-only log — no upserts, no updates.

```javascript
{
  // Identity
  "_id": ObjectId,                        // MongoDB auto-generated
  "connName": "PLC1",                     // S7CP_connection.name
  "connNumber": 1,                        // S7CP_connection.protocolConnectionNumber
  "driverInstanceNumber": 1,              // ProtocolDriverInstanceNumber

  // Alarm identity (from AlarmsDai)
  "cpuAlarmId": NumberLong(12345),        // AlarmsDai.CpuAlarmId (ulong)
  "alarmDomain": 256,                     // AlarmsDai.AlarmDomain (ushort)
  "messageType": 1,                       // AlarmsDai.MessageType (int)
  "sequenceCounter": 42,                  // AlarmsDai.SequenceCounter (uint)
  "clientAlarmId": 7890,                  // AlarmsDai.HmiInfo.ClientAlarmId (uint)

  // Event type
  "eventType": "Coming",                  // "Coming" or "Going" (from AsCgs.SubtypeId)
  "allStatesInfo": 3,                     // AlarmsDai.AsCgs.AllStatesInfo (byte)

  // PLC timestamps (from AsCgs)
  "plcTimestamp": ISODate("2026-03-17T10:00:00Z"),    // AsCgs.Timestamp
  "ackTimestamp": ISODate("2026-03-17T10:00:05Z"),    // AsCgs.AckTimestamp (zero if not acked)

  // Alarm texts (from AlarmsDai.AlarmTexts)
  "alarmText": "Motor overtemperature",   // AlarmTexts.AlarmText
  "infoText": "Check cooling circuit",    // AlarmTexts.Infotext
  "additionalTexts": [                    // AdditionalText1..9, omit if all empty
    "Zone A",
    "Drive 3"
  ],
  "textLanguageId": 1033,                 // LCID used for text decoding (en-US = 1033)

  // HMI classification (from AlarmsHmiInfo)
  "priority": 10,                         // HmiInfo.Priority (byte)
  "alarmClass": 1,                        // HmiInfo.AlarmClass (ushort)
  "groupId": 0,                           // HmiInfo.GroupId (byte)

  // Driver metadata
  "serverTimestamp": ISODate("2026-03-17T10:00:00.123Z"),  // DateTime.UtcNow at receive time
  "notificationSequenceNumber": 7,        // noti.NotificationSequenceNumber
  "creditTick": 9                         // noti.NotificationCreditTick (for diagnostics)
}
```

**Index recommendations:**
- `{ connNumber: 1, plcTimestamp: -1 }` — primary query pattern: "all alarms for this connection, newest first"
- `{ cpuAlarmId: 1, plcTimestamp: -1 }` — look up history for a specific alarm
- TTL index on `serverTimestamp` with configurable expiry — optional, prevents unbounded growth

**Why not realtimeData:** Alarm events are log entries with a PLC-side timestamp. They are not live point state. Inserting them into `realtimeData` would conflict with the existing point document model (one document per tag, updated in place) and pollute the Change Stream used by `cs_data_processor`.

## Suggested Build Order

The components have the following dependencies:

```
Step 1: S7CPAlarmEvent DTO + AlarmQueue field
   (no dependencies — pure data structure addition to Common.cs / Program.cs)
        |
        v
Step 2: ProcessMongoAlarms task (MongoAlarms.cs)
   (depends on AlarmQueue and S7CPAlarmEvent from Step 1)
   (can be written and tested with manually enqueued dummy events before Step 3)
        |
        v
Step 3: AlarmSubscriptionThread (AlarmSubscription.cs)
   (depends on S7CPAlarmEvent, AlarmQueue — both from Step 1)
   (depends on S7CommPlusConnection.AlarmSubscriptionCreate/WaitForNewS7plusReceived)
        |
        v
Step 4: Wire into Program.cs
   (launch AlarmQueue drain task alongside ProcessMongo)
   (start alarmSubscriptionThread from connectionThread after connect)
   (handle lifecycle: stop alarm thread on disconnect, restart on reconnect)
        |
        v
Step 5: Integration test
   (trigger alarm on PLC, verify document appears in s7plusAlarmEvents)
```

**Rationale for this order:**
- Steps 1-2 can be built and partially tested (MongoDB collection, schema, insert path) without any PLC connectivity.
- Step 3 is the protocol-dependent piece that requires a live PLC. Isolating it last means MongoDB plumbing is verified before protocol work begins.
- Step 4 is integration glue — deferred until both sides work independently.

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 1 PLC, PoC | Single `alarmSubscriptionThread` per connection; `InsertOneAsync` per event; no batching needed |
| 5-10 PLCs | Same pattern scales linearly — one thread per connection; consider batch inserts if alarm rate is high |
| High-frequency alarms (>100/sec) | Switch `ProcessMongoAlarms` from InsertOne to BulkWrite (same pattern as `ProcessMongo`) |

## Anti-Patterns

### Anti-Pattern 1: Subscription Creation Inside the Alarm Thread

**What people do:** Start `alarmSubscriptionThread` first and let it call `AlarmSubscriptionCreate()`, then wait for notifications.

**Why it's wrong:** `AlarmSubscriptionCreate()` uses `SendS7plusFunctionObject()` and `WaitForNewS7plusReceived()` — the same transport as the read cycle. If called concurrently with `connectionThread`'s `PerformReadCycle()`, the PDU receive queue will mix read responses with subscription confirmation responses. The library does not appear to multiplex these safely.

**Do this instead:** Call `AlarmSubscriptionCreate()` from `connectionThread` after connect, before starting the read cycle or in a mutex-guarded window. Only start the alarm thread after successful subscription creation.

### Anti-Pattern 2: Ignoring the Credit Limit

**What people do:** Receive notifications forever without refreshing the credit limit.

**Why it's wrong:** The PLC stops sending notifications when `NotificationCreditTick` reaches `CreditLimit`. The subscription goes silent. This is the protocol's flow-control mechanism (documented in `TestWaitForAlarmNotifications` comment: "Set new limit one tick before it expires, to get a constant flow of data").

**Do this instead:** In the notification receive loop, check `noti.NotificationCreditTick >= m_AlarmNextCreditLimit - 1` after every notification and call `SubscriptionSetCreditLimit()`. Port the credit management logic from `TestWaitForAlarmNotifications` verbatim for the PoC.

### Anti-Pattern 3: Blocking MongoDB Inserts in the Alarm Thread

**What people do:** Call `collection.InsertOneAsync()` directly inside the notification receive loop.

**Why it's wrong:** MongoDB latency spikes (network hiccup, replication lag) will cause the receive loop to stall, failing to refresh the credit limit in time and silencing the subscription.

**Do this instead:** Enqueue to `AlarmQueue` and process in a separate async task (Pattern 2 above). The receive loop only does: decode, enqueue, credit-check.

### Anti-Pattern 4: Sharing One Thread for Reads and Alarm Polling

**What people do:** Add alarm `WaitForNewS7plusReceived()` into the existing `connectionThread` read cycle.

**Why it's wrong:** `WaitForNewS7plusReceived()` is a blocking wait. Inserting it into the read cycle would stall tag data acquisition for the duration of the wait timeout on every iteration.

**Do this instead:** Keep the alarm notification receive loop on its own dedicated thread.

## Integration Points

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| `connectionThread` -> `alarmSubscriptionThread` | `srv.isConnected` flag + `srv.connection` reference | Read-only from alarm thread perspective; no locks needed for read but volatile or Interlocked recommended |
| `alarmSubscriptionThread` -> `ProcessMongoAlarms` | `AlarmQueue` (ConcurrentQueue — thread-safe) | Producer/consumer; no additional synchronization needed |
| `connectionThread` -> `ProcessMongo` | `DataQueue` (existing ConcurrentQueue) | Unchanged |
| `alarmSubscriptionThread` -> `S7CommPlusConnection` | Direct method calls (`WaitForNewS7plusReceived`, `SubscriptionSetCreditLimit`) | Must coordinate with `connectionThread` during subscription creation (see Anti-Pattern 1) |

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| S7CommPlusDriver library | Direct method calls on `S7CommPlusConnection` | `AlarmSubscriptionCreate()`, `AlarmSubscriptionDelete()` in `AlarmsHandler.cs` — experimental/test quality; `TODO Unknown` values present for `m_AlarmSubscriptionRelationId` and `m_AlarmSubscriptionRefRelationId` |
| MongoDB `s7plusAlarmEvents` | `InsertOneAsync` per event (or BulkWrite if rate warrants) | New collection; driver creates it on first insert implicitly; add indexes separately |

## Sources

- `json-scada/src/S7CommPlusClient/Program.cs` — direct inspection of `connectionThread`, thread startup, `DataQueue` pattern
- `json-scada/src/S7CommPlusClient/MongoUpdate.cs` — direct inspection of `ProcessMongo` async task and queue drain pattern
- `json-scada/src/S7CommPlusClient/MongoCommands.cs` — direct inspection of `ProcessMongoCmd` change stream pattern
- `json-scada/src/S7CommPlusClient/Common.cs` — direct inspection of `S7CP_connection`, `S7CPValue`, `rtFilt` types
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHandler.cs` — direct inspection of `AlarmSubscriptionCreate()`, `TestWaitForAlarmNotifications()`, credit management
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsDai.cs` — direct inspection of notification data model
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsAlarmTexts.cs` — direct inspection of text fields available per notification
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsAsCgs.cs` — direct inspection of Coming/Going state, timestamps, ack timestamp
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHmiInfo.cs` — direct inspection of priority, alarm class, group fields
- `.planning/codebase/ARCHITECTURE.md` — json-scada system architecture reference

---
*Architecture research for: S7CommPlus alarm subscription integration into json-scada*
*Researched: 2026-03-17*

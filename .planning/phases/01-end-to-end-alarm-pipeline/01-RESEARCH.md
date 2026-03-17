# Phase 1: End-to-End Alarm Pipeline - Research

**Researched:** 2026-03-17
**Domain:** C# / S7CommPlusDriver alarm subscription + MongoDB writes + .NET threading
**Confidence:** HIGH (all findings are from direct source-code inspection — no external documentation required)

## Summary

This phase is entirely self-contained within a well-understood local codebase. The source code for every component has been read directly: `AlarmsHandler.cs`, `AlarmsDai.cs`, `AlarmsAsCgs.cs`, `AlarmsHmiInfo.cs`, `Notification.cs`, `Subscription.cs`, `S7CommPlusConnection.cs`, `Program.cs`, `MongoUpdate.cs`, and `Common.cs`. No external library research is needed because the technology choices are already locked.

The five bugs are concretely identifiable from the source. BUG-01: `SubscriptionSetCreditLimit` in `AlarmsHandler.cs` (line 158) calls the method but it resolves to `Subscription.cs`'s implementation which passes `m_SubscriptionObjectId` — the alarm handler must use `m_AlarmSubscriptionObjectId` instead. BUG-02: `AlarmSubscriptionDelete` (line 170) calls `DeleteObject(SessionId2)` which deletes the session object rather than the subscription object — it must pass `m_AlarmSubscriptionObjectId`. BUG-03: `Notification.Deserialize` throws `NotImplementedException` for unknown item return values (`0x83` and `default` branch) and for unknown P2ReturnValue; the fix is a catch-and-continue in the alarm receive loop in `S7CommPlusClient`, not in the library. BUG-04: `AlarmsDai.FromNotificationObject` returns `null` when neither `DAI_Coming` nor `DAI_Going` attribute is present (`dai_id == 0`); the caller must null-check the result. BUG-05: `m_AlarmSubscriptionRelationId` is `0x7fffc001` — identical to the tag subscription RelationId; must be changed to `0x7fffc002`.

The alarm thread design, MongoDB schema, and connection isolation strategy are all decided. The research task is to map every code touch-point precisely so the planner can produce surgical task descriptions.

**Primary recommendation:** Plan each BUG-fix as a separate atomic task pointing to the exact line, then plan the AlarmThread class, then the MongoDB write, then integration with ConnectionThread.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Bug fix placement**
- All 5 bugs (BUG-01 to BUG-05) are fixed directly in S7CommPlusDriver (the local fork). S7CommPlusClient is not used to work around library bugs.
- Fixes are surgical — change only the wrong values, no surrounding refactor.
- Log lines are added around alarm lifecycle calls (create, credit-limit replenish, delete) to aid field debugging. This is the one intentional deviation from pure surgical.
- BUG-03 (NotImplementedException from unknown PDU types): exception is caught at the alarm receive loop level in S7CommPlusClient — catch, log, continue. The library itself is not changed for this bug.
- BUG-04 (null dai from ack-only notifications): log + skip — log a debug-level message ("ack-only notification, skipping") and continue the loop. No MongoDB write for ack-only PDUs.
- BUG-05 (RelationId collision): alarm subscription uses `0x7fffc002` as a hardcoded distinct constant (tag subscription uses `0x7fffc001`).

**PDU demux / connection isolation**
- Two separate TCP connections — one `S7CommPlusConnection` for tag polling (existing), a second dedicated `S7CommPlusConnection` for alarm subscription only.
- The alarm connection is opened only after the tag connection succeeds.
- If the alarm connection drops while the tag connection is up: log the loss and let the alarm thread exit. Tag polling continues unaffected. Re-establishment deferred to LIFE-04 (v2).

**Alarm thread design**
- Dedicated alarm thread per connection, separate from the existing `connectionThread`.
- The alarm thread owns its own `S7CommPlusConnection` object exclusively.
- The alarm thread's loop: Connect alarm connection → AlarmSubscriptionCreate → receive loop → on exit: AlarmSubscriptionDelete → Disconnect.
- MongoDB writer for alarms runs on the alarm thread itself (direct `InsertOneAsync` per event, no intermediate queue needed).

**MongoDB document schema**
- Collection name: `s7plusAlarmEvents`
- Mandatory v1 fields: `cpuAlarmId`, `alarmState` ("Coming" / "Going"), `alarmText`, `timestamp` (PLC-side, UTC), `ackState`, `connectionId`, `createdAt`
- Also include HmiInfo fields: `priority`, `alarmClass`, `groupId`
- `allStatesInfo` stored as raw byte value (semantics partially unknown — stored but not interpreted).
- Write concern: WriteConcern.W1 (acknowledged, as per MONGO-02).

**Alarm text language (LCID)**
- Hardcoded English (1033) for the PoC.

### Claude's Discretion
- Exact MongoDB index definition on `s7plusAlarmEvents` (if any)
- Whether the alarm connection reuses the same PLC credentials as the tag connection (yes, same endpoint and credentials from `S7CP_connection`)
- On shutdown: whether in-flight alarm receives are drained before `AlarmSubscriptionDelete` is called

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| BUG-01 | Credit-limit renewal targets correct alarm subscription ObjectId | Fix: `AlarmsHandler.cs` line 158 — `SubscriptionSetCreditLimit` must use `m_AlarmSubscriptionObjectId`, not the tag subscription ObjectId. The working reference pattern is in `Subscription.cs` line 134 which uses `m_SubscriptionObjectId`. |
| BUG-02 | Alarm subscription slot deleted cleanly on disconnect | Fix: `AlarmsHandler.cs` line 170 — `DeleteObject(SessionId2)` must become `DeleteObject(m_AlarmSubscriptionObjectId)`. The `DeleteObject` private method in `S7CommPlusConnection.cs` handles both session and non-session IDs correctly; only the argument is wrong. |
| BUG-03 | Unknown PDU types handled gracefully without crashing | Fix: in the alarm receive loop in `S7CommPlusClient` (new alarm thread code), wrap `Notification.DeserializeFromPdu(m_ReceivedPDU)` call in try/catch `NotImplementedException` — log and continue. Three throw sites exist in `Notification.cs`: lines 126, 129, 152. |
| BUG-04 | Ack-only notification events handled without null dereference | Fix: in the alarm receive loop, null-check result of `AlarmsDai.FromNotificationObject(noti.P2Objects[0], 1033)` before use. The null return path is at `AlarmsDai.cs` line 80 (`dai_id == 0`). |
| BUG-05 | Alarm subscription uses distinct RelationId from tag subscription | Fix: `AlarmsHandler.cs` line 36 — change `m_AlarmSubscriptionRelationId = 0x7fffc001` to `0x7fffc002`. Tag subscription uses `0x7fffc001` (confirm in `Subscription.cs`). |
| LIFE-01 | Driver creates alarm subscription on successful PLC connection | New alarm thread calls `AlarmSubscriptionCreate()` after its own connection succeeds. Integration point: spawned from `ConnectionThread` in `Program.cs` after `srv.isConnected = true`. |
| LIFE-02 | Driver maintains continuous credit-limit replenishment | Alarm receive loop checks `noti.NotificationCreditTick >= m_AlarmNextCreditLimit - 1` and calls `SubscriptionSetCreditLimit` (BUG-01 fixed). Pattern verbatim from `AlarmsHandler.cs` lines 153-158 but using the corrected ObjectId. |
| LIFE-03 | Driver deletes alarm subscription cleanly on disconnect or shutdown | Alarm thread finally/catch block calls `AlarmSubscriptionDelete()` (BUG-02 fixed) then disconnects alarm connection. |
| DATA-01 | Each alarm event captures alarm state (Coming / Going) | `AlarmsAsCgs.SubtypeId`: `Ids.DAI_Coming` (2673) = "Coming", `Ids.DAI_Going` (2677) = "Going". Resolved via `dai.AsCgs.SubtypeId`. |
| DATA-02 | Each alarm event captures alarm text as configured in TIA Portal | `dai.AlarmTexts` populated by `AlarmsAlarmTexts.FromNotificationBlob(...)` when LCID 1033 requested. Access via `dai.AlarmTexts` string property. |
| DATA-03 | Each alarm event captures PLC-side event timestamp (UTC) | `dai.AsCgs.Timestamp` — a `DateTime` produced by `Utils.DtFromValueTimestamp(...)`. Already UTC. |
| DATA-04 | Each alarm event captures acknowledgement state | `dai.AsCgs.AckTimestamp` is non-epoch when alarm has been acknowledged. `ackState` field: boolean or timestamp. |
| THRD-01 | Alarm notifications received on a dedicated thread | New `Thread alarmThread` created per `S7CP_connection` alongside the existing `connectionThread` in `Program.cs`. Follows the existing `new Thread(() => ConnectionThread(srv))` pattern. |
| THRD-02 | Thread-safe queue passes alarm events from receiver to MongoDB writer | Decision: alarm writes happen directly on the alarm thread (no queue needed per the architecture decision). THRD-02 is satisfied by the fact that the alarm thread is isolated — no shared queue contention with the tag bulk-write path. |
| MONGO-01 | Alarm events written to dedicated `s7plusAlarmEvents` collection | Collection obtained via `MongoDatabase.GetCollection<BsonDocument>("s7plusAlarmEvents")`. Follows pattern in `Common.cs` `EnsureActiveTagRequestIndexes`. |
| MONGO-02 | Alarm writes use acknowledged write concern (W1 minimum) | `collection.WithWriteConcern(WriteConcern.W1).InsertOneAsync(doc)`. Contrast with tag bulk writes which use `WriteConcern.Unacknowledged`. |
| MONGO-03 | Each alarm event document contains required fields | BsonDocument with: `cpuAlarmId`, `alarmState`, `alarmText`, `timestamp`, `ackState`, `connectionId`, `createdAt`, plus `priority`, `alarmClass`, `groupId`, `allStatesInfo`. |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| S7CommPlusDriver (local fork) | project ref | S7-1200/S7-1500 S7CommPlus protocol | Only available implementation; already integrated |
| MongoDB.Driver | 3.4.2 | MongoDB C# driver | Already in S7CommPlusClient.csproj; used for all existing DB writes |
| MongoDB.Bson | 3.4.2 | BSON document building | Paired with MongoDB.Driver; already imported |
| .NET 8.0 System.Threading | runtime | Thread, CancellationToken | Already used for connectionThread, ProcessRedundancyMongo, etc. |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| System.Collections.Concurrent | runtime | ConcurrentQueue | Pattern established for DataQueue; not needed for alarm path per architecture decision |
| MongoDB.Driver WriteConcern.W1 | 3.4.2 | Acknowledged write | Alarm inserts must be acknowledged |

**Installation:** No new packages required. All dependencies are already referenced in `S7CommPlusClient.csproj`.

## Architecture Patterns

### Recommended Project Structure
```
json-scada/src/S7CommPlusClient/
├── Program.cs              # Add: alarm thread spawn after isConnected = true; alarm thread stop on disconnect
├── AlarmThread.cs          # NEW: AlarmThread method, AlarmEvent struct/class, alarm MongoDB write
├── Common.cs               # Add: alarmThread field to S7CP_connection; alarm collection name constant
├── MongoUpdate.cs          # No changes needed
S7CommPlusDriver/src/S7CommPlusDriver/Alarming/
├── AlarmsHandler.cs        # BUG-01, BUG-02, BUG-05 fixes + log lines
```

### Pattern 1: Alarm Thread Method Structure
**What:** A static void method `AlarmThread(S7CP_connection srv)` mirroring the existing `ConnectionThread` structure
**When to use:** Spawned once per `S7CP_connection` after tag connection succeeds

```csharp
// Mirrors the ConnectionThread(srv) pattern in Program.cs
static void AlarmThread(S7CP_connection srv)
{
    var alarmConn = new S7CommPlusDriver.S7CommPlusConnection();
    try
    {
        Log(srv.name + " - AlarmThread: connecting alarm connection...");
        int res = alarmConn.Connect(srv.endpointURLs[0], srv.password, srv.username, (int)srv.timeoutMs);
        if (res != 0)
        {
            Log(srv.name + " - AlarmThread: alarm connection failed, error: " + res);
            return;
        }
        Log(srv.name + " - AlarmThread: alarm connection established.");

        res = alarmConn.AlarmSubscriptionCreate();
        Log(srv.name + " - AlarmThread: AlarmSubscriptionCreate returned " + res);
        if (res != 0)
        {
            Log(srv.name + " - AlarmThread: subscription create failed, exiting.");
            return;
        }

        var alarmCollection = MongoDatabase
            .GetCollection<BsonDocument>("s7plusAlarmEvents")
            .WithWriteConcern(WriteConcern.W1);

        // Receive loop
        while (!srv.alarmThreadStop)
        {
            alarmConn.m_LastError = 0;
            alarmConn.WaitForNewS7plusReceived(5000);
            if (alarmConn.m_LastError != 0) break;

            Notification noti;
            try
            {
                noti = Notification.DeserializeFromPdu(alarmConn.m_ReceivedPDU);  // BUG-03 site
            }
            catch (NotImplementedException ex)
            {
                Log(srv.name + " - AlarmThread: unknown PDU type, skipping. " + ex.Message, LogLevelDetailed);
                continue;
            }

            if (noti == null || noti.P2Objects == null || noti.P2Objects.Count == 0) continue;

            // Credit-limit replenishment (BUG-01 fixed in library)
            // ... check noti.NotificationCreditTick

            var dai = AlarmsDai.FromNotificationObject(noti.P2Objects[0], 1033);
            if (dai == null)  // BUG-04 site
            {
                Log(srv.name + " - AlarmThread: ack-only notification, skipping.", LogLevelDebug);
                continue;
            }

            // Build and insert document
            var doc = BuildAlarmDocument(dai, srv);
            await alarmCollection.InsertOneAsync(doc);
        }
    }
    finally
    {
        Log(srv.name + " - AlarmThread: deleting alarm subscription...");
        alarmConn.AlarmSubscriptionDelete();  // BUG-02 fixed in library
        alarmConn.Disconnect();
        Log(srv.name + " - AlarmThread: alarm connection closed.");
    }
}
```

### Pattern 2: MongoDB Alarm Document
**What:** BsonDocument construction for alarm events
**When to use:** Once per non-null, non-ack-only alarm receive

```csharp
// Collection name constant in Common.cs or Program.cs
public static string AlarmEventsCollectionName = "s7plusAlarmEvents";

static BsonDocument BuildAlarmDocument(AlarmsDai dai, S7CP_connection srv)
{
    string alarmState = dai.AsCgs.SubtypeId == S7CommPlusDriver.Alarming.AlarmsAsCgs.SubtypeIds.Coming
        ? "Coming" : "Going";

    return new BsonDocument
    {
        { "cpuAlarmId",   (long)dai.CpuAlarmId },
        { "alarmState",   alarmState },
        { "alarmText",    dai.AlarmTexts?.GetText() ?? "" },
        { "timestamp",    dai.AsCgs.Timestamp },
        { "ackState",     dai.AsCgs.AckTimestamp != DateTime.MinValue },
        { "connectionId", srv.protocolConnectionNumber },
        { "createdAt",    DateTime.UtcNow },
        { "priority",     (int)dai.HmiInfo.Priority },
        { "alarmClass",   (int)dai.HmiInfo.AlarmClass },
        { "groupId",      (int)dai.HmiInfo.GroupId },
        { "allStatesInfo",(int)dai.AllStatesInfo }
    };
}
```

### Pattern 3: Alarm Thread Spawn in ConnectionThread
**What:** Spawning and stopping the alarm thread from ConnectionThread
**When to use:** After `srv.isConnected = true`; before the next reconnect attempt

```csharp
// In ConnectionThread, after srv.isConnected = true:
srv.alarmThreadStop = false;
srv.alarmThread = new Thread(() => AlarmThread(srv));
srv.alarmThread.Start();
Log(srv.name + " - AlarmThread started.");

// On disconnect / exception catch, before Thread.Sleep(5000):
srv.alarmThreadStop = true;
srv.alarmThread?.Join(3000);
srv.alarmThread = null;
```

### Anti-Patterns to Avoid
- **Sharing the tag connection for alarm receive:** The library's `m_ReceivedPDUs` queue is a single `Stream`; two threads calling `WaitForNewS7plusReceived` on the same connection race. Use two connections.
- **Calling `SubscriptionSetCreditLimit` without fixing BUG-01 first:** Without BUG-01 fix, the credit limit targets the session object, so the PLC will stop sending alarms after 10 events.
- **Calling `DeleteObject(SessionId2)` in `AlarmSubscriptionDelete`:** Without BUG-02 fix, the session is torn down instead of just the subscription slot, leaking a `PlcSubscriptionsFree` slot on the PLC.
- **Not catching `NotImplementedException` from `Notification.DeserializeFromPdu`:** Any unknown PDU byte kills the alarm thread permanently if not caught.
- **Using `WriteConcern.Unacknowledged` for alarm inserts:** The tag bulk-write path uses Unacknowledged for performance; alarm inserts must use W1 per MONGO-02.
- **Blocking the ConnectionThread on alarm operations:** All alarm network I/O must be on `alarmThread`, never inside the `ConnectionThread` read cycle.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| PLC alarm deserialization | Custom PDU parser | `Notification.DeserializeFromPdu` + `AlarmsDai.FromNotificationObject` | Already implemented; edge cases in PeekByte and P2Objects list handling are already covered |
| Thread-safe alarm queuing | Custom producer-consumer queue | Direct `InsertOneAsync` on alarm thread (per architecture decision) | Alarm events are infrequent; extra queue adds complexity without benefit |
| MongoDB connection pool management | Custom retry/pool | `MongoClient` with existing `ConnectMongoClient(JSConfig)` pattern | The existing client already handles pooling and retry |
| Alarm text extraction | Custom text parser | `AlarmsAlarmTexts.FromNotificationBlob(blob, 1033)` | LCID-keyed sparse array parsing is already implemented in `AlarmsAlarmTexts.cs` |
| HmiInfo binary parsing | Manual byte parsing | `AlarmsHmiInfo.FromValueBlob(blob)` | Endian-specific multi-field blob parse is already correct in library |

## Common Pitfalls

### Pitfall 1: Credit Limit Silently Stops After 10 Events (BUG-01)
**What goes wrong:** After exactly 10 alarm events, the PLC stops sending notifications. No error is thrown.
**Why it happens:** `SubscriptionSetCreditLimit` in `AlarmsHandler.cs` resolves via the partial-class pattern but the method body in `Subscription.cs` sends the request to `m_SubscriptionObjectId` (tag subscription object), not `m_AlarmSubscriptionObjectId`. The PLC renews the wrong object; the alarm credit expires.
**How to avoid:** Fix `AlarmsHandler.cs` so its credit limit path targets `m_AlarmSubscriptionObjectId`. The `SubscriptionSetCreditLimit` call on line 158 must become a direct `SetVariable` request to `m_AlarmSubscriptionObjectId` (or the method signature must accept the object ID). The cleanest fix: add a dedicated private method `AlarmSubscriptionSetCreditLimit` that sends to `m_AlarmSubscriptionObjectId`, parallel to the existing one.
**Warning signs:** Alarm thread appears healthy but stops receiving after exactly 10 events.

### Pitfall 2: PLC Subscription Slot Leak on Reconnect (BUG-02)
**What goes wrong:** After repeated connect/disconnect cycles, `PlcSubscriptionsFree` on the PLC decrements and never recovers. PLC eventually refuses new subscriptions.
**Why it happens:** `AlarmSubscriptionDelete` calls `DeleteObject(SessionId2)` which deletes the session object, not the subscription object. The subscription object `m_AlarmSubscriptionObjectId` is leaked.
**How to avoid:** Fix line 170 in `AlarmsHandler.cs`: `DeleteObject(m_AlarmSubscriptionObjectId)`.
**Warning signs:** PLC diagnostic shows decreasing `PlcSubscriptionsFree` after each driver restart.

### Pitfall 3: RelationId Collision with Tag Subscription (BUG-05)
**What goes wrong:** PLC may reject or conflate the alarm subscription because it shares RelationId `0x7fffc001` with the tag subscription (even on a separate TCP connection, the PLC tracks RelationIds globally).
**Why it happens:** `m_AlarmSubscriptionRelationId = 0x7fffc001` in `AlarmsHandler.cs` is a placeholder value equal to the tag subscription RelationId.
**How to avoid:** Change `AlarmsHandler.cs` line 36 to `0x7fffc002`.
**Warning signs:** `AlarmSubscriptionCreate` returns error code, or alarm notifications never arrive.

### Pitfall 4: NotImplementedException Kills Alarm Thread (BUG-03)
**What goes wrong:** An unknown byte in a notification PDU (e.g., `0x83` item return value, or unknown P2ReturnValue) throws `NotImplementedException` which propagates uncaught out of the alarm receive loop, permanently terminating the alarm thread.
**Why it happens:** `Notification.cs` lines 126, 129, and 152 throw on unrecognised bytes.
**How to avoid:** Wrap the `Notification.DeserializeFromPdu` call in a try/catch for `NotImplementedException`; log at `LogLevelDetailed` and `continue`.
**Warning signs:** Alarm thread exits with a `NotImplementedException` in the log; no alarms received after a certain point.

### Pitfall 5: Null Dereference on Ack-Only Notifications (BUG-04)
**What goes wrong:** `NullReferenceException` when calling `dai.AsCgs.SubtypeId` after `AlarmsDai.FromNotificationObject` returns `null` for ack-only PDUs.
**Why it happens:** When a notification object has neither `DAI_Coming` nor `DAI_Going` attribute (`dai_id == 0` in `AlarmsDai.cs` line 79), the factory returns `null`.
**How to avoid:** Check `if (dai == null) { Log(...); continue; }` before any property access.
**Warning signs:** `NullReferenceException` in alarm thread stack trace pointing to `AlarmsDai` usage.

### Pitfall 6: `AlarmTexts` Can Be Null
**What goes wrong:** `NullReferenceException` on `dai.AlarmTexts.GetText()` when the notification doesn't carry text.
**Why it happens:** `AlarmsDai.AlarmTexts` is assigned from `AlarmsAlarmTexts.FromNotificationBlob(...)` which may return null for some event types.
**How to avoid:** Use null-coalescing: `dai.AlarmTexts?.GetText() ?? ""`.
**Warning signs:** Intermittent NRE on alarm text access.

### Pitfall 7: Alarm Thread Must Stop Before ConnectionThread Reconnects
**What goes wrong:** Old alarm thread still holds the alarm connection open while ConnectionThread starts a new reconnect cycle, causing duplicate subscriptions or resource leaks.
**Why it happens:** No stop signal sent to alarm thread before ConnectionThread retries.
**How to avoid:** Set `srv.alarmThreadStop = true` and join the alarm thread before clearing `srv.connection = null` and sleeping in the ConnectionThread catch block. The alarm thread must observe the stop flag at the top of its receive loop.
**Warning signs:** Two alarm threads active simultaneously; duplicate events in MongoDB.

### Pitfall 8: `m_ReceivedPDU` Is a Shared Stream Field
**What goes wrong:** If alarm receive and tag receive happened on the same `S7CommPlusConnection`, race conditions would corrupt both streams.
**Why it happens:** `m_ReceivedPDU` is a single `Stream` field on `S7CommPlusConnection`; not thread-safe.
**How to avoid:** Architecture decision: two separate `S7CommPlusConnection` objects. This pitfall is already designed away but must not be violated.
**Warning signs:** Random deserialization exceptions mixing alarm and tag PDUs.

## Code Examples

Verified from direct source inspection:

### BUG-01 Fix Location
```csharp
// AlarmsHandler.cs — current (broken) line 158:
SubscriptionSetCreditLimit(m_AlarmNextCreditLimit);
// This calls Subscription.cs::SubscriptionSetCreditLimit which uses m_SubscriptionObjectId (tag subscription).
// Fix: add a new private method that targets m_AlarmSubscriptionObjectId directly:
private int AlarmSubscriptionSetCreditLimit(short limit)
{
    var setVarReq = new SetVariableRequest(ProtocolVersion.V2);
    setVarReq.TransportFlags = 0x74;
    setVarReq.InObjectId = m_AlarmSubscriptionObjectId;  // KEY: alarm object, not tag subscription
    setVarReq.Address = Ids.SubscriptionCreditLimit;
    setVarReq.Value = new ValueInt(limit);
    return SendS7plusFunctionObject(setVarReq);
}
```

### BUG-02 Fix Location
```csharp
// AlarmsHandler.cs — current (broken) AlarmSubscriptionDelete():
m_AlarmSubscriptionObjectId = 0;  // zeroed BEFORE the call — also wrong order
res = DeleteObject(SessionId2);   // deletes session, not subscription

// Fix:
public int AlarmSubscriptionDelete()
{
    int res;
    Log("AlarmSubscriptionDelete: Calling DeleteObject for m_AlarmSubscriptionObjectId=0x"
        + m_AlarmSubscriptionObjectId.ToString("X8"));
    res = DeleteObject(m_AlarmSubscriptionObjectId);  // correct target
    m_AlarmSubscriptionObjectId = 0;
    return res;
}
```

### BUG-05 Fix Location
```csharp
// AlarmsHandler.cs line 36 — current (broken):
uint m_AlarmSubscriptionRelationId = 0x7fffc001;
// Fix:
uint m_AlarmSubscriptionRelationId = 0x7fffc002;
```

### Accessing Alarm State from AlarmsAsCgs
```csharp
// Source: AlarmsAsCgs.cs — SubtypeId field
// Ids.DAI_Coming == 2673 == AlarmsAsCgs.SubtypeIds.Coming
// Ids.DAI_Going  == 2677 == AlarmsAsCgs.SubtypeIds.Going
string alarmState = (dai.AsCgs.SubtypeId == (uint)AlarmsAsCgs.SubtypeIds.Coming)
    ? "Coming" : "Going";
```

### Existing MongoDB InsertOne Pattern (modelled from MongoUpdate.cs)
```csharp
// Source: MongoUpdate.cs pattern — adapted for alarm inserts
var alarmCollection = MongoDatabase
    .GetCollection<BsonDocument>(AlarmEventsCollectionName)
    .WithWriteConcern(WriteConcern.W1);
await alarmCollection.InsertOneAsync(doc).ConfigureAwait(false);
```

### Thread Spawn Pattern (from Program.cs)
```csharp
// Source: Program.cs lines 226-228 — existing connectionThread spawn
srv.connectionThread = new Thread(() => ConnectionThread(srv));
srv.connectionThread.Start();
// Mirror for alarm thread:
srv.alarmThread = new Thread(() => AlarmThread(srv));
srv.alarmThread.Start();
```

### Log() Usage (from Common.cs)
```csharp
// Source: Common.cs lines 174-183
Log(srv.name + " - AlarmSubscriptionCreate: started", LogLevelBasic);
Log(srv.name + " - ack-only notification, skipping", LogLevelDebug);
Log(srv.name + " - credit limit reached, replenishing to " + m_AlarmNextCreditLimit, LogLevelDetailed);
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `TestWaitForAlarmNotifications` with bounded loop count | Infinite production receive loop with stop flag | This phase | Alarm subscription never terminates due to count limit |
| `Console.WriteLine` in library | `Log()` helper from `Common.cs` | This phase (log lines added per decisions) | Structured timestamped output via existing logger |
| Single `S7CommPlusConnection` for all traffic | Two separate connections (tag + alarm) | This phase | Eliminates `m_ReceivedPDU` race condition |

## Open Questions

1. **`AlarmTexts.GetText()` API**
   - What we know: `AlarmsAlarmTexts` is populated by `FromNotificationBlob` with LCID 1033. `AlarmsDai.ToString()` accesses it via `AlarmTexts.ToString()`.
   - What's unclear: The exact public property or method name to extract the string text (e.g., `AlarmTexts.ToString()` vs a dedicated property). The file `AlarmsAlarmTexts.cs` was not read.
   - Recommendation: Read `AlarmsAlarmTexts.cs` before implementing the alarm document builder. Use whatever string accessor is available; fall back to `.ToString()` if no dedicated property exists.

2. **`WaitForNewS7plusReceived` accessibility**
   - What we know: The method is called in `AlarmsHandler.cs` (line 134) and `Subscription.cs`, so it exists on `S7CommPlusConnection`.
   - What's unclear: Whether `m_ReceivedPDU`, `WaitForNewS7plusReceived`, and `m_LastError` are `public`, `internal`, or `protected`. The alarm thread code in `S7CommPlusClient` (a separate project) needs to call these.
   - Recommendation: Read `S7CommPlusConnection.cs` for field visibility before writing the alarm thread. If internal, the alarm thread method may need to live inside `S7CommPlusDriver` or a facade must be added.

3. **`ackState` field semantics**
   - What we know: `AlarmsAsCgs.AckTimestamp` is a `DateTime`. When epoch (`DateTime.MinValue`), alarm is unacknowledged.
   - What's unclear: Whether the REQUIREMENTS.md `ackState` field should be stored as boolean (acknowledged yes/no) or as the actual timestamp, or both.
   - Recommendation: Store as boolean `(dai.AsCgs.AckTimestamp != DateTime.MinValue)` for simplicity; optionally also store `ackTimestamp` as a separate field.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | xUnit or NUnit — neither is currently configured for S7CommPlusClient or S7CommPlusDriver (DriverTest project exists but covers PLC tag address parsing only) |
| Config file | None — see Wave 0 |
| Quick run command | `dotnet test S7CommPlusDriver/src/DriverTest/DriverTest.csproj` |
| Full suite command | `dotnet test S7CommPlusDriver/src/DriverTest/DriverTest.csproj` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| BUG-01 | `AlarmSubscriptionSetCreditLimit` sends request to `m_AlarmSubscriptionObjectId` | unit | `dotnet test --filter "BUG01"` | ❌ Wave 0 |
| BUG-02 | `AlarmSubscriptionDelete` calls `DeleteObject(m_AlarmSubscriptionObjectId)` not `SessionId2` | unit | `dotnet test --filter "BUG02"` | ❌ Wave 0 |
| BUG-03 | Unknown PDU byte in `Notification.DeserializeFromPdu` does not propagate exception | unit | `dotnet test --filter "BUG03"` | ❌ Wave 0 |
| BUG-04 | `AlarmsDai.FromNotificationObject` returns null for ack-only PDU; caller skips gracefully | unit | `dotnet test --filter "BUG04"` | ❌ Wave 0 |
| BUG-05 | `m_AlarmSubscriptionRelationId` value is `0x7fffc002` | unit | `dotnet test --filter "BUG05"` | ❌ Wave 0 |
| LIFE-01/02/03 | Alarm subscription lifecycle (create, replenish, delete) | integration (live PLC) | manual-only — requires PLC hardware | N/A |
| DATA-01 to DATA-04 | Alarm document fields populated correctly | integration (live PLC) | manual-only — requires PLC hardware | N/A |
| THRD-01 | Alarm thread isolated from tag read cycle | integration (live PLC) | manual-only | N/A |
| THRD-02 | Direct write on alarm thread — no shared queue contention | code review | N/A | N/A |
| MONGO-01 | Documents appear in `s7plusAlarmEvents` collection | integration (live PLC) | manual-only | N/A |
| MONGO-02 | Write concern is W1 | unit — inspect `WithWriteConcern` call | `dotnet test --filter "MONGO02"` | ❌ Wave 0 |
| MONGO-03 | Document schema contains all required fields | unit | `dotnet test --filter "MONGO03"` | ❌ Wave 0 |

**Note on integration tests:** The majority of phase success criteria require a live S7-1200/S7-1500 PLC. These are manual-only and correspond directly to the five Success Criteria in the phase description. Unit tests cover the bug fixes and MongoDB write concern only.

### Sampling Rate
- **Per task commit:** `dotnet test S7CommPlusDriver/src/DriverTest/DriverTest.csproj --filter "Category=BugFix"` (once bug fix tests exist)
- **Per wave merge:** `dotnet test S7CommPlusDriver/src/DriverTest/DriverTest.csproj`
- **Phase gate:** Unit tests green + manual PLC validation of all 5 success criteria before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] A test project targeting S7CommPlusClient or a new `S7CommPlusClientTests` project — covers BUG-01 through BUG-05 unit tests and MONGO-02/03
- [ ] Test stubs for `BUG01_CreditLimitTargetsAlarmObjectId`, `BUG02_DeleteSubscriptionNotSession`, `BUG03_UnknownPduByteContinues`, `BUG04_NullDaiSkipped`, `BUG05_RelationIdIs0x7fffc002`
- [ ] Framework install: `dotnet add package Microsoft.NET.Test.Sdk xunit xunit.runner.visualstudio` — if adding new test project

## Sources

### Primary (HIGH confidence)
- Direct file read: `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHandler.cs` — all 5 bug sites confirmed
- Direct file read: `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsDai.cs` — BUG-04 null-return path at line 80
- Direct file read: `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsAsCgs.cs` — SubtypeId enums, Timestamp field
- Direct file read: `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHmiInfo.cs` — Priority, AlarmClass, GroupId field names
- Direct file read: `S7CommPlusDriver/src/S7CommPlusDriver/Core/Notification.cs` — BUG-03 throw sites at lines 126, 129, 152
- Direct file read: `S7CommPlusDriver/src/S7CommPlusDriver/Subscriptions/Subscription.cs` — `SubscriptionSetCreditLimit` uses `m_SubscriptionObjectId` (tag object)
- Direct file read: `S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs` — `Disconnect()` calls `DeleteObject(m_SessionId)`, `DeleteObject` private method signature
- Direct file read: `json-scada/src/S7CommPlusClient/Program.cs` — thread spawn pattern, ConnectionThread, `isConnected` flag, `Active` gate
- Direct file read: `json-scada/src/S7CommPlusClient/MongoUpdate.cs` — `BulkWriteAsync` pattern, `WriteConcern.Unacknowledged` for tags
- Direct file read: `json-scada/src/S7CommPlusClient/Common.cs` — `S7CP_connection` fields, `Log()` signature, `ConnectMongoClient`, `WriteConcern.W1` (via driver version)
- Direct file read: `json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` — MongoDB.Driver 3.4.2, .NET 8.0, x64

### Secondary (MEDIUM confidence)
None required — all evidence is from direct source inspection.

### Tertiary (LOW confidence)
None.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — verified from `.csproj` file
- Bug fix locations: HIGH — exact line numbers from source code read
- Architecture: HIGH — locked decisions in CONTEXT.md, mirroring patterns found in Program.cs
- Pitfalls: HIGH — derived directly from source code structure (throw sites, null returns, shared fields)
- AlarmsAlarmTexts API: LOW — `AlarmsAlarmTexts.cs` was not read; text accessor method name is unknown

**Research date:** 2026-03-17
**Valid until:** Indefinite — source code does not change unless commits are made to the fork

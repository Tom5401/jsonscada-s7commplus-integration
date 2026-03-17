# Stack Research

**Domain:** C#/.NET driver — S7CommPlus alarm subscriptions writing to MongoDB
**Researched:** 2026-03-17
**Confidence:** HIGH (all core choices grounded in existing codebase; version facts verified via NuGet)

---

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| .NET / C# | net8.0 (LTS) | Runtime and language | Already used by S7CommPlusClient and S7CommPlusDriver; both `.csproj` files target `net8.0`. Changing the runtime is out of scope. |
| S7CommPlusDriver | Project reference (local) | S7CommPlus protocol implementation, alarm subscription API | The only reverse-engineered C# library that implements the S7CommPlus alarm subscription wire protocol. `AlarmSubscriptionCreate`, `TestWaitForAlarmNotifications`, `AlarmSubscriptionDelete` are already present in `Alarming/AlarmsHandler.cs`. No NuGet alternative exists. |
| MongoDB.Driver | 3.4.2 (already in `S7CommPlusClient.csproj`) | Write alarm events to MongoDB | Already in use for all realtimeData and soeData writes. Upgrading to 3.7.x is unnecessary for a PoC and could introduce breaking changes. Stick with the version the project already locks. |
| MongoDB.Bson | 3.4.2 (already in `S7CommPlusClient.csproj`) | BSON document construction | Paired with MongoDB.Driver; the project already uses `BsonDocument`, `BsonDateTime`, `BsonString`, etc. for all document writes. Same version pinning rationale applies. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| System.Collections.Concurrent (BCL) | Built into .NET 8 | `ConcurrentQueue<AlarmEvent>` as the bridge between the alarm notification thread and the MongoDB write task | Use the same producer/consumer queue pattern that `DataQueue` (`ConcurrentQueue<S7CPValue>`) already uses for tag data. No extra NuGet package needed. |
| System.Threading (BCL) | Built into .NET 8 | `Thread`, `Mutex` for the alarm notification loop | S7CommPlusDriver's `WaitForNewS7plusReceived` spins with `Thread.Sleep(2)` and a `Mutex`; it is a blocking synchronous call and must run on a dedicated `Thread`, not a `Task` or `async` method body. |
| System.Globalization (BCL) | Built into .NET 8 | `CultureInfo` for the `alarmTextsLanguageId` LCID passed to `AlarmSubscriptionCreate` and `AlarmsDai.FromNotificationObject` | Use `CultureInfo.CurrentCulture.LCID` or hardcode the target language ID (e.g. `1033` for en-US, `1031` for de-DE) in the connection configuration. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| dotnet CLI / `dotnet build` | Build the solution | Both projects are already `net8.0`; no additional toolchain setup is required. |
| MongoDB Compass (optional) | Inspect the `alarmEvents` collection during development | Useful for verifying inserted documents during end-to-end testing. Not a build dependency. |

---

## Installation

No new NuGet packages are required. All dependencies are either already in the `.csproj` file or are BCL types.

```bash
# Verify existing packages restore cleanly
dotnet restore json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj
```

If a version bump of MongoDB.Driver is ever desired (e.g. to 3.7.x):

```bash
dotnet add json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj package MongoDB.Driver --version 3.7.0
dotnet add json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj package MongoDB.Bson    --version 3.7.0
```

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Dedicated `Thread` for alarm notification loop | `Task` / `async`/`await` loop | Never for this driver. `WaitForNewS7plusReceived` internally busy-waits with `Thread.Sleep(2)` and a `Mutex`. Wrapping it in `Task.Run` works but does not make it truly async — it just hides a blocking call on a ThreadPool thread. Using a named `Thread` matches the existing `ConnectionThread` pattern, is explicit about the blocking nature, and avoids ThreadPool starvation on slow responses. |
| `ConcurrentQueue<AlarmEvent>` + shared `ProcessMongo`-style writer | Direct `InsertOneAsync` inside the notification loop | A direct insert inside the loop would require awaiting inside a synchronous blocking loop and mixes concerns. The queue+writer pattern is already battle-tested in this codebase (`DataQueue`) and decouples PLC timing from MongoDB latency. |
| `InsertManyAsync` (batched) for alarm writes | Single `InsertOneAsync` per alarm | Alarm events are low-frequency (bursts during fault conditions, otherwise quiet). `InsertOneAsync` is simpler and sufficient. Use batching only if load testing reveals a bottleneck. |
| `WriteConcern.W1` (acknowledged) for alarm collection | `WriteConcern.Unacknowledged` (used for realtimeData bulk writes) | Alarm events are audit-grade data. Use at least `W1` acknowledged writes for the alarm collection. The existing `WriteConcern.Unacknowledged` on `realtimeData` is a performance trade-off acceptable for process values, but not for alarm events which must be reliably persisted. |
| Project reference to S7CommPlusDriver | NuGet package | No NuGet package for S7CommPlusDriver exists. A project reference is the only option. The `.csproj` already has `<ProjectReference Include="..\..\..\S7CommPlusDriver\src\S7CommPlusDriver\S7CommPlusDriver.csproj" />`. |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `async`/`await` directly inside the alarm wait loop | `WaitForNewS7plusReceived` is a synchronous busy-wait. Mixing `await` inside that loop produces non-obvious thread re-entrancy and will appear to work until a timeout fires on the wrong thread. | Dedicated `Thread` (matching `ConnectionThread`) |
| Calling `AlarmSubscriptionCreate` from within the existing `ConnectionThread` body that also runs the read cycle | The alarm notification loop blocks indefinitely waiting for PDUs. Running it on the same thread as the read cycle would permanently starve the read cycle. | Separate dedicated thread, spawned after a successful connection, similar to how `thrMongoCmd` is spawned in `Program.cs`. |
| Re-using the `soeData` collection for alarm events | `soeData` has its own schema used by other json-scada drivers and the AdminUI. Writing S7CommPlus alarm events there without the expected fields will corrupt the consumer's view of that collection. | A new `alarmEvents` collection (or a name that does not collide with existing json-scada collections). |
| `MongoDB.Driver` v2.x legacy package | The project already uses v3.x. The v2.x package (`mongocsharpdriver` on NuGet) has a different namespace layout and connection API. Mixing them in one project causes build failures. | `MongoDB.Driver` 3.4.2 as pinned in `S7CommPlusClient.csproj`. |
| Upgrading MongoDB.Driver beyond 3.4.2 for this PoC | The 3.x series has breaking changes between minor versions. The existing working read/write code was written and tested against 3.4.2. Version drift mid-project introduces risk with zero benefit for a PoC. | Pin at 3.4.2 until the working driver is stable; evaluate upgrades in a separate maintenance step. |

---

## Stack Patterns by Variant

**If alarm texts should be embedded in the notification document at write time (recommended for PoC):**
- Set `AlarmSubscriptionRef_SendAlarmTexts_Rid = true` in `AlarmSubscriptionCreate` (already done in `AlarmsHandler.cs` line 76).
- Call `AlarmsDai.FromNotificationObject(noti.P2Objects[0], languageId)` which reads texts from the embedded `DAI_AlarmTexts_Rid` blob in the `Notification`.
- No separate `ExploreASAlarms` call at subscription time is needed.
- Because: this path is simpler, lower latency, and the notification already carries the text blob when `SendAlarmTexts = true`.

**If pre-loading the alarm text dictionary is required (not needed for PoC):**
- Call `ExploreASAlarms(ref Alarms, languageId)` once after connect to populate `Dictionary<ulong, AlarmData>`.
- Store it on `S7CP_connection` as a field (`AlarmDictionary`).
- On each notification, look up `AlarmsDai.CpuAlarmId` in the dictionary for text.
- Because: useful if you need texts for alarms that fired before the subscription was created, but adds complexity and a long blocking call at connect time.

**If the credit limit management becomes a bottleneck (unlikely for PoC):**
- Increase `creditLimitStep` from `5` to a larger value, or set `SubscriptionCreditLimit = -1` (unlimited).
- Because: the credit mechanism is how the PLC throttles alarm delivery. For a PoC with a handful of alarms, the default values in `AlarmsHandler.cs` (initial limit 10, step 5) are sufficient.

---

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| MongoDB.Driver 3.4.2 | MongoDB Server 4.4, 5.0, 6.0, 7.0, 8.0 | Verified via the [Compatibility table](https://www.mongodb.com/docs/drivers/csharp/current/compatibility/). The project does not pin a specific MongoDB server version; all modern json-scada deployments use 6.x or 7.x. |
| MongoDB.Driver 3.4.2 | .NET 8 (net8.0) | HIGH confidence. Already building and running in the project. |
| S7CommPlusDriver (project ref) | net8.0, x64 | Both projects target `net8.0` and `x64`. The OpenSSL native DLLs are x64-only (`_WIN64` defined in S7CommPlusDriver.csproj). Do not attempt AnyCPU or x86 builds. |
| `WaitForNewS7plusReceived` (blocking pattern) | Must not be called from a `Task` directly scheduled on the ThreadPool | The method polls every 2 ms using `Thread.Sleep(2)`. On a ThreadPool thread this is safe but wastes a pool thread for the entire wait duration. A dedicated `Thread` is semantically clearer and matches the existing `connectionThread` pattern. |

---

## Threading Architecture (derived from codebase analysis)

The existing driver uses this threading model:

```
Main thread
  ├── thrRedundancy (Thread)       — MongoDB redundancy/heartbeat
  ├── ProcessMongo (Task.Run)      — async MongoDB bulk-write consumer of DataQueue
  ├── thrMongoCmd (Thread)         — command polling loop
  └── connectionThread[] (Thread)  — one per S7CP_connection, runs ConnectionThread()
```

The alarm subscription fits here:

```
connectionThread (existing, per connection)
  — connects, browses tags, runs read cycle
  └── alarmThread (new Thread, spawned after successful connect)
        — calls AlarmSubscriptionCreate()
        — loop: WaitForNewS7plusReceived() → parse Notification → enqueue AlarmEvent
        — on disconnect/error: calls AlarmSubscriptionDelete(), exits

ProcessMongo or AlarmMongoWriter (Task.Run, shared or new)
  — dequeues AlarmEvent from AlarmQueue (ConcurrentQueue<AlarmEvent>)
  — InsertOneAsync into alarmEvents collection with WriteConcern.W1
```

Key constraint: `WaitForNewS7plusReceived` blocks the calling thread. The alarm loop must be on its own `Thread`, not on `connectionThread` (which owns the read cycle) and not blocking `ProcessMongo`.

---

## Sources

- Codebase: `S7CommPlusClient.csproj` — MongoDB.Driver 3.4.2, net8.0, x64 confirmed
- Codebase: `S7CommPlusDriver.csproj` — net8.0, x64, OpenSSL DLL dependency confirmed
- Codebase: `AlarmsHandler.cs` — `AlarmSubscriptionCreate`, `TestWaitForAlarmNotifications`, `AlarmSubscriptionDelete` API surface confirmed
- Codebase: `AlarmsDai.cs` / `AlarmsAsCgs.cs` / `AlarmsAlarmTexts.cs` — alarm data model fields confirmed (CpuAlarmId, AllStatesInfo, Timestamp, AckTimestamp, AlarmText, AlarmDomain)
- Codebase: `Notification.cs` — PDU-level notification structure; `P2Objects` list carries alarm DAI objects
- Codebase: `S7CommPlusConnection.cs` — `WaitForNewS7plusReceived` confirmed as 2ms-sleep polling with `Mutex`, not async
- Codebase: `MongoUpdate.cs` — existing `ProcessMongo` async Task with `ConcurrentQueue`, `BulkWriteAsync`, `WriteConcern.Unacknowledged` pattern documented
- Codebase: `Program.cs` — existing thread/task spawn pattern documented
- [NuGet: MongoDB.Driver](https://www.nuget.org/packages/mongodb.driver) — 3.7.0 is latest stable (March 2026); 3.4.2 is current project pin (MEDIUM: NuGet page, consistent with codebase)
- [MongoDB C# Driver Compatibility](https://www.mongodb.com/docs/drivers/csharp/current/compatibility/) — .NET 8 / MongoDB server version matrix (HIGH: official docs)

---

*Stack research for: S7CommPlus alarm subscriptions in C#/.NET driver with MongoDB write*
*Researched: 2026-03-17*

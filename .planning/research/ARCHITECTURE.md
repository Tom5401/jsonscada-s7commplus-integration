# Architecture Research

**Domain:** S7CommPlus Alarm Origin & Cleanup (v1.2)
**Researched:** 2026-03-23
**Confidence:** HIGH — based on direct source reading of all affected files

---

## System Overview

```
+-------------------------------------------------------------------+
|                       Frontend (Vue 3)                            |
|  S7PlusAlarmsViewerPage.vue                                       |
|  - 11-col table + filters + per-row Ack btn  [existing]          |
|  + per-row Delete btn  [NEW]                                      |
|  + "Delete Filtered" bulk button  [NEW]                           |
|  + DB/FB Name column  [NEW]                                       |
+----------------------------+--------------------------------------+
                             | HTTP (fetch)
+----------------------------v--------------------------------------+
|          Backend -- server_realtime_auth (Node.js)                |
|  GET  /Invoke/auth/listS7PlusAlarms   [MODIFIED: expose _id]     |
|  POST /Invoke/auth/ackS7PlusAlarm     [existing, unmodified]     |
|  POST /Invoke/auth/deleteS7PlusAlarms [NEW -- single + bulk]     |
+----------------------------+--------------------------------------+
                             | MongoDB driver
+----------------------------v--------------------------------------+
|                         MongoDB                                   |
|  s7plusAlarmEvents collection                                     |
|  - existing 15+ fields                                            |
|  + relationId (int32/long)  [NEW field]                          |
|  + dbFbName   (string)      [NEW field]                          |
+----------------------------+--------------------------------------+
                             |
+----------------------------v--------------------------------------+
|              S7CommPlusClient (C# driver)                         |
|  Program.cs / ConnectionThread                                    |
|  + BrowseRelationIdNames() at startup  [NEW]                     |
|  AlarmThread.cs / AlarmThread()                                   |
|  + reads noti.P2Objects[0].RelationId  [MODIFIED]                |
|  AlarmThread.cs / BuildAlarmDocument()                            |
|  + looks up dbFbName from in-memory map  [MODIFIED]              |
|  Common.cs / S7CP_connection                                      |
|  + RelationIdNameMap field  [NEW]                                 |
+----------------------------+--------------------------------------+
                             | S7CommPlus protocol (TCP)
+----------------------------v--------------------------------------+
|       S7CommPlusDriver submodule (C# library)                     |
|  BrowseAlarms.cs -- ExploreASAlarms()   [REUSED AS-IS]           |
|  AlarmsHandler.cs -- WaitForAlarmNotification()  [UNMODIFIED]    |
|  PObject.RelationId -- already populated in notifications         |
|  AlarmsDai -- FromNotificationObject()  [UNMODIFIED]             |
+-------------------------------------------------------------------+
```

---

## Component Responsibilities

| Component | v1.2 Responsibility | Status |
|-----------|---------------------|--------|
| `BrowseAlarms.cs` ExploreASAlarms() | PLC browse for all alarm DataBlocks; returns `Dictionary<ulong, AlarmData>` where `AlarmData.RelationId` is the DB/FB identifier | Reuse as-is |
| `AlarmThread.cs` AlarmThread() | Read `noti.P2Objects[0].RelationId` before calling `AlarmsDai.FromNotificationObject`; pass to `BuildAlarmDocument` | Modify |
| `AlarmThread.cs` BuildAlarmDocument() | Accept `uint relationId` parameter; write `relationId` and `dbFbName` fields into each MongoDB document | Modify |
| `AlarmThread.cs` or new AlarmOrigin.cs | `BrowseRelationIdNames()` — calls `ExploreASAlarms`, builds `RelationIdNameMap` | New |
| `Program.cs` ConnectionThread() | Call `BrowseRelationIdNames(srv)` after connect, before AlarmThread starts | Modify |
| `Common.cs` S7CP_connection | Add `Dictionary<uint, string> RelationIdNameMap` runtime field (`[BsonIgnore]`) | Modify |
| `server_realtime_auth/index.js` listS7PlusAlarms | Remove `_id: 0` projection exclusion so frontend can identify rows for deletion | Modify |
| `server_realtime_auth/index.js` | Add `POST /Invoke/auth/deleteS7PlusAlarms` endpoint | New |
| `S7PlusAlarmsViewerPage.vue` | Add DB/FB Name column, Delete column, per-row Delete btn, "Delete Filtered" bulk btn, `deleteAlarm()` and `deleteFiltered()` functions | Modify |

---

## Question 1: DB/FB Lookup at Startup

### Where the browse happens

`ConnectionThread()` in `Program.cs` is the right insertion point. The sequence after `srv.connection.Connect()` succeeds is:

```
1. srv.isConnected = true
2. BrowseRelationIdNames(srv)        <-- INSERT HERE (new)
3. srv.alarmThread = new Thread(AlarmThread)
4. srv.alarmThread.Start()
5. BrowseAndCreateTags(srv)          <-- existing (autoCreateTags)
```

Step 2 must precede `AlarmThread.Start()` because `AlarmThread` reads from the map on the first notification. `BrowseRelationIdNames` calls `srv.connection.ExploreASAlarms()` on the existing tag connection. This is safe: the tag polling loop has not yet entered `PerformReadCycle` at this point -- the thread is still in the one-time setup block guarded by `srv.isConnected == false`.

### Map lifetime: driver memory only (not MongoDB)

Keep the map in memory as `Dictionary<uint, string>` on `S7CP_connection` (`[BsonIgnore]`, named `RelationIdNameMap`). Reasons:

- The map is per-connection and per-session; it reflects PLC program state at connect time.
- Persisting to MongoDB would require a separate collection, TTL management, and cross-process read-back -- complexity with no PoC benefit.
- On reconnect, `ConnectionThread` rebuilds it automatically.
- The map is small (typically fewer than 100 entries per PLC program).

### How it flows into alarm event writes

Read `noti.P2Objects[0].RelationId` in `AlarmThread()` before calling `AlarmsDai.FromNotificationObject()`, then pass it as a parameter to `BuildAlarmDocument`:

```csharp
// In AlarmThread(), before FromNotificationObject:
uint relationId = noti.P2Objects[0].RelationId;

// Signature change:
var doc = BuildAlarmDocument(dai, srv, relationId);
```

```csharp
// In BuildAlarmDocument signature:
static BsonDocument BuildAlarmDocument(AlarmsDai dai, S7CP_connection srv, uint relationId)
{
    srv.RelationIdNameMap.TryGetValue(relationId, out string dbFbName);
    // ... existing fields ...
    { "relationId",  (long)relationId },
    { "dbFbName",    dbFbName ?? "" },
}
```

### Building the map from ExploreASAlarms results

`ExploreASAlarms()` returns `Dictionary<ulong, AlarmData>` where each `AlarmData.RelationId` is the DB/FB identifier and `AlarmData.AlText.AlarmText` is the configured alarm text (not the DB name).

The DB name itself is in the PLC program object tree. During the "Explore Alarm AP" phase of `ExploreASAlarms`, the code iterates `obj.GetObjects()` where `obj` is the `PLCProgram_Class_Rid` object. Each sub-object (`ob`) has `ob.RelationId` as the identifier. Adding `Ids.ObjectVariableTypeName` to the attribute list for the AP explore request would yield the DB/FB name.

**Fallback:** If name extraction proves complex in the initial implementation, store `relationId` as a uint in MongoDB and display it numerically in the viewer. The DB name column can then be populated in a follow-on commit once the name extraction is validated.

---

## Question 2: RelationID Field in Alarm Notifications

### Where RelationId is available

`PObject` (the type of `noti.P2Objects[0]`) has a public field `RelationId` (type `UInt32`) confirmed in `PObject.cs`. It is deserialized from the PDU automatically. `AlarmsDai.FromNotificationObject()` does NOT currently read or expose this field -- and does not need to be modified.

### Relationship to cpuAlarmId

From `BrowseAlarms.cs`:
```csharp
cpualarmid = (ulong)t1_relid << 32 | (ulong)t2_alid << 16;
```
The high 32 bits of `cpuAlarmId` are the `RelationId`. This means:
```csharp
uint relationId = (uint)(dai.CpuAlarmId >> 32);
```
is a valid alternative derivation without touching `PObject` at all. Both approaches are equivalent; reading `noti.P2Objects[0].RelationId` directly is more explicit and does not depend on the bit layout assumption remaining stable.

### How to add it to the MongoDB document

No changes to `AlarmsDai`, `AlarmsHandler.cs`, or any submodule file. The value is read from `noti.P2Objects[0].RelationId` in `AlarmThread()` before `AlarmsDai.FromNotificationObject()` is called, then forwarded as a parameter to `BuildAlarmDocument()`.

---

## Question 3: Delete API Design

### HTTP method

Use `POST` for both single-row delete and bulk delete. Reasons:

- The existing ack endpoint is `POST`. Consistency avoids fetch method divergence in the frontend.
- HTTP DELETE has no standard request body; passing filter state requires non-standard use that some proxies reject.
- The json-scada backend already uses POST for all mutation operations.

**Endpoint:** `POST /Invoke/auth/deleteS7PlusAlarms`

### Single vs bulk in one endpoint

One endpoint handles both cases, distinguished by body shape:

```json
// Single row delete (or explicit list of IDs)
{ "ids": ["<ObjectId string>", ...] }

// Bulk delete matching current viewer filter
{
  "filter": {
    "alarmState": "All",
    "alarmClassName": "Acknowledgment required"
  }
}
```

Backend logic:
- `ids` present: `collection.deleteMany({ _id: { $in: ids.map(id => new ObjectId(id)) } })`
- `filter` present: build MongoDB filter from fields (same logic as the viewer's client-side filter), then `collection.deleteMany(mongoFilter)`

**Why not two separate endpoints?** Single endpoint reduces backend surface. The payload shape makes the intent unambiguous. The pattern mirrors ack, which also handles different alarm IDs via a single endpoint.

---

## Question 4: Frontend Filter State to Backend

### How filter state is currently held

`statusFilter` and `alarmClassFilter` are `ref` values. `filteredAlarms` is a `computed` that derives the visible set client-side. The filter state is immediately available for sending in any fetch call.

### Passing filter state to deleteS7PlusAlarms

```javascript
const deleteFiltered = async () => {
  await fetch('/Invoke/auth/deleteS7PlusAlarms', {
    method: 'post',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      filter: {
        alarmState: statusFilter.value,        // "All" | "Incoming" | "Outgoing"
        alarmClassName: alarmClassFilter.value  // "All" | specific class name string
      }
    })
  })
  await fetchAlarms()
}
```

Backend maps frontend filter values to MongoDB:
- `"Incoming"` -> `{ alarmState: "Coming" }`
- `"Outgoing"` -> `{ alarmState: "Going" }`
- `"All"` -> omit alarmState condition
- `alarmClassName !== "All"` -> `{ alarmClassName: value }`

### Exposing _id for single-row delete

The `listS7PlusAlarms` endpoint currently projects `{ _id: 0 }`. This must be changed so the frontend receives `_id` as a string. The frontend passes it back in `ids` for single-row delete. This is a one-line change in `index.js` (remove `_id: 0` from the projection, or change to `{ _id: 1 }`).

---

## Question 5: Suggested Build Order

### Dependency graph

```
Phase 1 (RelationID in MongoDB)
  -- no external dependencies --
  enables: Phase 2 (dbFbName populated once map exists)
  enables: Phase 4 (field exists in documents for column display)

Phase 2 (DB/FB Name at startup)
  depends on: Phase 1 complete

Phase 3 (Delete backend + _id exposure)
  -- no dependency on Phase 1 or 2 --
  enables: Phase 4 (frontend delete buttons need endpoint)

Phase 4 (Frontend Delete + DB/FB column)
  depends on: Phase 1 (dbFbName field in documents)
  depends on: Phase 2 (dbFbName populated with actual name)
  depends on: Phase 3 (backend delete endpoint + _id in list response)
```

### Recommended build order

**Phase 1 -- RelationID in MongoDB**
- Add `RelationIdNameMap` to `S7CP_connection` (empty dictionary initially).
- Read `noti.P2Objects[0].RelationId` in `AlarmThread()`.
- Change `BuildAlarmDocument()` to accept and write `relationId` and `dbFbName` (`""` placeholder until Phase 2).
- No submodule changes. No backend changes. No frontend changes.
- Deliverable: new alarm events in MongoDB have `relationId` populated; `dbFbName` is `""`.

**Phase 2 -- DB/FB Name at startup**
- Implement `BrowseRelationIdNames(srv)`.
- Call it in `ConnectionThread()` after connect, before AlarmThread start.
- Populate `srv.RelationIdNameMap` with `relationId -> dbFbName` entries.
- Deliverable: new alarm events show populated `dbFbName`; existing events remain `""`.

**Phase 3 -- Delete backend + _id exposure**
- Change `listS7PlusAlarms` projection to include `_id`.
- Add `POST /Invoke/auth/deleteS7PlusAlarms` in `index.js`.
- Deliverable: operator can verify the endpoint works via curl/Postman before frontend wires it.

**Phase 4 -- Frontend Delete + DB/FB column**
- Add `dbFbName` to `headers` array (automatic once field exists in documents).
- Add Delete column with per-row Delete button (mirrors Ack button template slot pattern).
- Add "Delete Filtered" button above the table.
- Wire `deleteAlarm(id)` (single-row) and `deleteFiltered()` (bulk) functions.
- After each delete, call `fetchAlarms()` to refresh from source of truth.
- Deliverable: full v1.2 feature set visible in the viewer.

---

## Data Flow Changes

### Alarm event write (with origin enrichment) -- MODIFIED

```
PLC notification arrives
  |
  v
AlarmThread: noti.P2Objects[0].RelationId         <-- NEW read
  |
  v
AlarmThread: srv.RelationIdNameMap.TryGetValue()   <-- NEW lookup
  |
  v
BuildAlarmDocument: writes relationId + dbFbName   <-- MODIFIED
  |
  v
s7plusAlarmEvents MongoDB: 17+ fields              <-- MODIFIED schema
```

### Alarm deletion flow -- NEW

```
User clicks Delete (single row)           User clicks Delete Filtered
  |                                           |
  v                                           v
POST /deleteS7PlusAlarms                  POST /deleteS7PlusAlarms
body: { ids: ["<_id>"] }                  body: { filter: { alarmState, alarmClassName } }
  |                                           |
  v                                           v
backend: deleteMany by _id               backend: deleteMany with filter
  |                                           |
  v                                           v
s7plusAlarmEvents: rows removed          s7plusAlarmEvents: rows removed
  |                                           |
  v                                           v
frontend: fetchAlarms()                   frontend: fetchAlarms()
```

### Startup browse flow -- NEW

```
ConnectionThread: srv.connection.Connect() succeeds
  |
  v
BrowseRelationIdNames(srv):
  srv.connection.ExploreASAlarms(ref alarmDict, 1033)
  for each AlarmData in alarmDict:
    srv.RelationIdNameMap[alarmData.RelationId] = dbFbName
  |
  v
srv.RelationIdNameMap populated in driver memory
  |
  v
AlarmThread starts -- map is ready before first notification arrives
```

---

## New vs Modified Components (explicit)

| Component | Change Type | What Changes |
|-----------|------------|--------------|
| `Common.cs` S7CP_connection | Modified | Add `Dictionary<uint, string> RelationIdNameMap` field (`[BsonIgnore]`) |
| `Program.cs` ConnectionThread() | Modified | Call `BrowseRelationIdNames(srv)` before `AlarmThread.Start()` |
| `AlarmThread.cs` AlarmThread() | Modified | Read `noti.P2Objects[0].RelationId` before `FromNotificationObject`; pass to `BuildAlarmDocument` |
| `AlarmThread.cs` BuildAlarmDocument() | Modified | Add `uint relationId` parameter; write `relationId` and `dbFbName` BsonDocument fields |
| `AlarmThread.cs` or new `AlarmOrigin.cs` | New | `BrowseRelationIdNames(S7CP_connection srv)` method |
| `server_realtime_auth/index.js` listS7PlusAlarms | Modified | Remove `_id: 0` from projection so `_id` is returned to frontend |
| `server_realtime_auth/index.js` | New endpoint | `POST /Invoke/auth/deleteS7PlusAlarms` |
| `S7PlusAlarmsViewerPage.vue` | Modified | dbFbName column, Delete column, per-row Delete btn, "Delete Filtered" btn, `deleteAlarm()`, `deleteFiltered()` |
| `BrowseAlarms.cs` (submodule) | Unmodified | `ExploreASAlarms()` reused as-is |
| `AlarmsHandler.cs` (submodule) | Unmodified | `WaitForAlarmNotification()` unmodified |
| `AlarmsDai.cs` (submodule) | Unmodified | `FromNotificationObject()` unmodified -- `RelationId` is read from `PObject` by the caller |
| `PObject.cs` (submodule) | Unmodified | `RelationId` field already public and populated |

---

## Integration Points

| Boundary | Communication | Notes |
|----------|---------------|-------|
| ConnectionThread -> AlarmThread | `srv.RelationIdNameMap` written before thread starts | Read-only after population; no locking needed |
| AlarmThread -> BuildAlarmDocument | New `uint relationId` parameter | Internal to `partial class MainClass`; clean break |
| ConnectionThread -> S7CommPlusConnection | `ExploreASAlarms()` called on tag connection during startup | Safe: called before polling loop begins, single-threaded at that point |
| Frontend -> Backend | New `POST deleteS7PlusAlarms` endpoint | POST body carries either `ids` array or `filter` object; never both |
| Backend -> MongoDB | `deleteMany` on `s7plusAlarmEvents` | No cascade effects; collection is append-only from the driver perspective |
| listS7PlusAlarms response | Add `_id` field | Vue component receives string representation of ObjectId |

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Persisting RelationIdNameMap to MongoDB

**What people do:** Store the browse results in a separate MongoDB collection so the map survives driver restart without re-browsing.

**Why it's wrong:** Adds a collection, TTL or manual cleanup, and cross-process synchronisation -- complexity with no PoC benefit. The map is cheap to rebuild at connect time.

**Do this instead:** Keep map in `S7CP_connection` memory. Rebuild on every reconnect.

### Anti-Pattern 2: Modifying AlarmsDai to expose RelationId

**What people do:** Add `public uint RelationId` to `AlarmsDai` and populate it inside `FromNotificationObject`.

**Why it's wrong:** The submodule is a vendored library. Adding fields to `AlarmsDai` for driver-specific concerns pollutes the library API and complicates future submodule updates.

**Do this instead:** Read `noti.P2Objects[0].RelationId` directly in `AlarmThread()` before calling `FromNotificationObject`. Pass it as a plain parameter to `BuildAlarmDocument`.

### Anti-Pattern 3: Client-side-only delete

**What people do:** Remove items from `alarms.value` in Vue state without calling the backend.

**Why it's wrong:** The next auto-refresh (5 seconds) restores deleted rows from MongoDB.

**Do this instead:** Always call the backend delete endpoint, then call `fetchAlarms()` to refresh state from the source of truth.

### Anti-Pattern 4: HTTP DELETE with filter body

**What people do:** Route bulk filtered delete to `DELETE /deleteS7PlusAlarms` with a JSON body.

**Why it's wrong:** HTTP DELETE with a request body is non-standard. Some reverse proxies strip the body. Inconsistent with the existing POST-for-mutation convention in this codebase.

**Do this instead:** `POST /deleteS7PlusAlarms` with a body containing either `ids` or `filter`.

### Anti-Pattern 5: Separate endpoints for single and bulk delete

**What people do:** Create `POST /deleteS7PlusAlarm` (singular) and `POST /deleteS7PlusAlarmsFiltered` (plural bulk).

**Why it's wrong:** Doubles backend surface area for no benefit.

**Do this instead:** Single `POST /deleteS7PlusAlarms` with body shape determining single vs bulk.

### Anti-Pattern 6: Building RelationIdNameMap inside AlarmThread

**What people do:** Call `ExploreASAlarms` from inside `AlarmThread` on first notification.

**Why it's wrong:** The alarm connection (`alarmConn`) is dedicated to subscriptions. Sending an `ExploreRequest` on it mid-subscription is untested and may produce unexpected PDU interleaving. The tag connection (`srv.connection`) is available and confirmed safe for explore requests during startup.

**Do this instead:** Build the map in `ConnectionThread` on `srv.connection`, before `AlarmThread.Start()`.

---

## Sources

- Direct source reading: `AlarmThread.cs`, `MongoCommands.cs`, `Common.cs`, `Program.cs` (S7CommPlusClient)
- Direct source reading: `AlarmsHandler.cs`, `BrowseAlarms.cs`, `AlarmsDai.cs`, `PObject.cs` (S7CommPlusDriver submodule)
- Direct source reading: `server_realtime_auth/index.js` (listS7PlusAlarms + ackS7PlusAlarm endpoints)
- Direct source reading: `S7PlusAlarmsViewerPage.vue` (frontend component)
- `.planning/PROJECT.md` -- v1.2 requirements and constraints

---

*Architecture research for: S7CommPlus Alarm Origin & Cleanup (v1.2)*
*Researched: 2026-03-23*

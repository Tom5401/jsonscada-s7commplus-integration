# Architecture Patterns

**Domain:** S7CommPlus Tag Tree Browser — DatablockBrowser + TagTreeBrowser (v1.4)
**Researched:** 2026-03-26
**Confidence:** HIGH — based on direct code inspection of all affected files

---

## System Overview

```
+-------------------------------------------------------------------+
|                         PLC Layer                                 |
|  S7-1200 / S7-1500 (S7CommPlus protocol)                         |
+------------------+---------------------+--------------------------+
                   | GetListOfDatablocks  | getTypeInfoByRelId(tiRelId)
                   | (startup)            | (on-demand browse request)
+------------------v---------------------v--------------------------+
|              Driver Layer (C# / .NET)                             |
|  S7CommPlusClient / Program.cs (ConnectionThread)                 |
|                                                                   |
|  STARTUP:                                                         |
|    GetListOfDatablocks() -> s7plusDatablocks collection (NEW)     |
|    stores: db_name, db_number, db_block_relid, db_block_ti_relid  |
|    connectionNumber (for multi-connection future-proofing)        |
|                                                                   |
|  ON-DEMAND BROWSE:                                                |
|    ProcessMongoCmd() watches commandsQueue (change stream)        |
|    ASDU = "s7plus-browse-request"                                 |
|    address = tiRelId (uint as string)                             |
|    -> getTypeInfoByRelId(tiRelId) -> PObject                      |
|    -> serialize PObject to s7plusTypeInfoCache (NEW collection)   |
|    stores: tiRelId, varnames[], vartypes[], softdatatypes[]        |
|    status: "ok" | "error"                                         |
+------------------+---------------------+--------------------------+
                   | MongoDB (two new collections)
+------------------v---------------------v--------------------------+
|                 Data Layer (MongoDB)                              |
|  s7plusDatablocks  (startup snapshot; replaced on reconnect)      |
|    { connectionNumber, db_name, db_number,                        |
|      db_block_relid, db_block_ti_relid, updatedAt }               |
|                                                                   |
|  s7plusTypeInfoCache  (on-demand type info; persistent cache)     |
|    { tiRelId, varnames, vartypes, status, fetchedAt }             |
|                                                                   |
|  realtimeData  (existing; live values for configured tags)        |
|  commandsQueue (existing; browse requests inserted by API)        |
+------------------+---------------------+--------------------------+
                   | REST / JSON  (fetch)
+------------------v---------------------v--------------------------+
|               API Layer (Node.js)                                 |
|  server_realtime_auth / index.js                                  |
|  GET  /Invoke/auth/listS7PlusDatablocks  [NEW]                    |
|    -> find all docs in s7plusDatablocks                           |
|  GET  /Invoke/auth/getS7PlusTypeInfo?tiRelId=X  [NEW]             |
|    -> find cached doc in s7plusTypeInfoCache                      |
|    -> if not found: returns { status: "pending" }                 |
|    -> frontend polls until status === "ok"                        |
|  POST /Invoke/auth/requestS7PlusTypeInfo  [NEW]                   |
|    -> inserts browse command into commandsQueue                   |
|    -> ASDU="s7plus-browse-request", address=tiRelId               |
|  GET  /Invoke/auth/getRealtimeValues?addresses=...  [NEW or reuse]|
|    -> query realtimeData by protocolSourceObjectAddress list      |
+------------------+---------------------+--------------------------+
                   | fetch() (on-demand + polling)
+------------------v---------------------v--------------------------+
|              UI Layer (Vue 3 / Vuetify)                           |
|  DatablockBrowserPage.vue  [NEW]                                  |
|    - on mount: GET listS7PlusDatablocks                           |
|    - renders v-treeview with one root node per datablock          |
|    - click row -> router.push('/s7plus-tag-tree?db=<name>')       |
|                                                                   |
|  TagTreeBrowserPage.vue  [NEW]                                    |
|    - receives dbName (query param) or tiRelId (query param)       |
|    - lazy expand: GET getS7PlusTypeInfo, if pending POST request  |
|      then poll until resolved (max ~10s)                          |
|    - live values: GET getRealtimeValues for visible tag addresses  |
|    - window.open() from S7PlusAlarmsViewerPage.vue  [MODIFIED]    |
|                                                                   |
|  S7PlusAlarmsViewerPage.vue  [MODIFIED]                           |
|    - originDbName cell becomes a clickable link                   |
|    - click: window.open('/#/s7plus-tag-tree?db=<originDbName>')   |
|                                                                   |
|  DashboardPage.vue  [MODIFIED]                                    |
|    - add DatablockBrowser shortcut card                           |
|                                                                   |
|  router/index.js  [MODIFIED]                                      |
|    - /s7plus-datablocks -> DatablockBrowserPage                   |
|    - /s7plus-tag-tree   -> TagTreeBrowserPage                     |
+-------------------------------------------------------------------+
```

---

## Key Design Decision: On-Demand Browse vs Startup Pre-Cache

**Recommendation: On-demand browse via command queue.**

### Option A — Startup Pre-Cache (pre-fetch all type info at startup)

The driver calls `getTypeInfoByRelId(db_block_ti_relid)` for every datablock during the startup sequence and stores all results in `s7plusTypeInfoCache`.

**Pros:**
- Browse is instant once startup completes — no waiting in the UI
- Simpler frontend (just GET, no polling loop)

**Cons:**
- GetListOfDatablocks() already involves two round-trips to the PLC (ExploreRequest + ReadValues for tiRelId). Adding getTypeInfoByRelId for every DB extends startup time proportionally to the number of datablocks. A PLC with 50 datablocks could add 50 serial ExploreRequests before the tag polling loop starts.
- Type info is fetched even for datablocks the operator never browses.
- The driver's `typeInfoList` in-memory cache is already sized for the connection thread — adding a write path to MongoDB for every DB at startup increases coupling between the browse path and the alarm path.
- `getTypeInfoByRelId` is synchronous/blocking in the driver (uses `WaitForNewS7plusReceived`). Doing this in the ConnectionThread before `AlarmThread` starts delays alarm subscription.

### Option B — On-Demand Browse via Command Queue (recommended)

The driver does not pre-fetch type info. The frontend dispatches a browse request by inserting a command into `commandsQueue` (ASDU = "s7plus-browse-request"). The driver's existing `ProcessMongoCmd()` change stream detects the insert and calls `getTypeInfoByRelId`. The result is stored in `s7plusTypeInfoCache`. The frontend polls `getS7PlusTypeInfo` until `status === "ok"`.

**Pros:**
- Startup time unchanged — no additional PLC round-trips at boot
- AlarmThread starts immediately after existing startup browse (RelationIdNameMap)
- Only fetches type info for datablocks the operator actually expands
- The command queue pattern is already used for alarm ack — same mental model, same change stream handler
- Results are cached in MongoDB — second open of the same datablock is instant
- Fits PoC principle: minimal, lazy, no over-engineering

**Cons:**
- First expand of an uncached node requires a 1–3 second wait (PLC round-trip latency + MongoDB poll)
- Slightly more frontend logic: request → poll until resolved → render
- If the PLC is disconnected, the browse request will time out (driver not connected → cancel with "not connected")

**Verdict:** On-demand wins for this PoC. The startup-time penalty of pre-caching outweighs the one-time latency of lazy browse, and the command queue pattern is already proven in the codebase.

---

## Component Boundaries

| Component | Responsibility | File | Status |
|-----------|---------------|------|--------|
| `ConnectionThread` (Program.cs) | At connect: call `GetListOfDatablocks`, write snapshot to `s7plusDatablocks`, replace prior docs for this connectionNumber | `S7CommPlusClient/Program.cs` | MODIFIED |
| `ProcessMongoCmd` (MongoCommands.cs) | Watch commandsQueue; on ASDU="s7plus-browse-request", call `getTypeInfoByRelId(tiRelId)`, serialize PObject to `s7plusTypeInfoCache` | `S7CommPlusClient/MongoCommands.cs` | MODIFIED |
| `listS7PlusDatablocks` endpoint | Return all docs from `s7plusDatablocks` sorted by `db_name` | `server_realtime_auth/index.js` | NEW |
| `getS7PlusTypeInfo` endpoint | Return cached doc for `tiRelId`; return `{ status: "pending" }` if not found | `server_realtime_auth/index.js` | NEW |
| `requestS7PlusTypeInfo` endpoint | Insert browse command into commandsQueue; idempotent (skip if already cached with status="ok") | `server_realtime_auth/index.js` | NEW |
| `DatablockBrowserPage.vue` | Fetch and display datablock list; navigate to TagTreeBrowser on row click | `AdminUI/src/components/DatablockBrowserPage.vue` | NEW |
| `TagTreeBrowserPage.vue` | Lazy tag tree with live values; opened by router or window.open | `AdminUI/src/components/TagTreeBrowserPage.vue` | NEW |
| `S7PlusAlarmsViewerPage.vue` | Add clickable originDbName cell that opens TagTreeBrowser in new window | `AdminUI/src/components/S7PlusAlarmsViewerPage.vue` | MODIFIED |
| `DashboardPage.vue` | Add DatablockBrowser shortcut card | `AdminUI/src/components/DashboardPage.vue` | MODIFIED |
| `router/index.js` | Add two new routes | `AdminUI/src/router/index.js` | MODIFIED |

---

## New MongoDB Collections

### s7plusDatablocks

Written by driver at startup (on each successful connect). Replaces prior snapshot for the same connectionNumber.

```
{
  connectionNumber: int,        // protocolConnectionNumber
  connectionName:   string,     // for display
  db_name:          string,     // e.g. "AlarmData_DB"
  db_number:        uint,       // e.g. 1
  db_block_relid:   uint,       // raw relation ID of the DB block
  db_block_ti_relid: uint,      // type-info relation ID (key for browse)
  updatedAt:        ISODate
}
```

Index: `{ connectionNumber: 1, db_name: 1 }` (unique). Replaced per-connection on reconnect using `ReplaceOneAsync` with upsert, or `deleteMany + insertMany`.

### s7plusTypeInfoCache

Written by driver on demand when a browse command is processed.

```
{
  tiRelId:     uint,            // type-info relation ID (the browse key)
  status:      string,          // "ok" | "error"
  errorMsg:    string,          // populated only when status="error"
  varnames:    [ string ],      // PObject.VarnameList.Names
  vartypes:    [                // parallel array to varnames
    {
      softdatatype:  int,       // Softdatatype enum value
      hasRelation:   bool,      // OffsetInfoType.HasRelation()
      relationId:    uint,      // ioit.GetRelationId() — child tiRelId for lazy expansion
      is1Dim:        bool,      // OffsetInfoType.Is1Dim()
      isMDim:        bool,      // OffsetInfoType.IsMDim()
      arrayElementCount: uint,  // if is1Dim
      arrayLowerBounds:  int,   // if is1Dim
      mdimElementCounts: [uint],// if isMDim
      mdimLowerBounds:   [int]  // if isMDim
    }
  ],
  fetchedAt:   ISODate
}
```

Index: `{ tiRelId: 1 }` (unique). No TTL needed for PoC — type schema is stable for the lifetime of the TIA Portal project.

---

## Data Flow: DatablockBrowser

```
1. Driver connects to PLC
2. ConnectionThread calls GetListOfDatablocks() -> dbInfoList
3. For each dbInfo: upsert doc into s7plusDatablocks
   (deleteMany { connectionNumber } + insertMany for simplicity)
4. Vue: DatablockBrowserPage mounts -> GET /Invoke/auth/listS7PlusDatablocks
5. Server: db.collection('s7plusDatablocks').find({}).sort({ db_name: 1 }).toArray()
6. Vue: renders v-data-table or v-list with one row per datablock
7. Operator clicks row: router.push('/s7plus-tag-tree?tiRelId=<db_block_ti_relid>&db=<db_name>')
```

---

## Data Flow: TagTreeBrowser — Lazy Expand

```
1. Vue: TagTreeBrowserPage mounts with query param tiRelId=X (root node)
2. Vue: GET /Invoke/auth/getS7PlusTypeInfo?tiRelId=X
3a. Cache hit (status="ok"): render node children immediately
3b. Cache miss:
    - Vue: POST /Invoke/auth/requestS7PlusTypeInfo { tiRelId: X, connectionNumber: N }
    - Server: inserts commandsQueue doc:
        { protocolSourceASDU: "s7plus-browse-request",
          protocolSourceObjectAddress: tiRelId.toString(),
          protocolSourceConnectionNumber: connectionNumber,
          timeTag: now }
    - Driver: ProcessMongoCmd change stream fires
    - Driver: calls getTypeInfoByRelId(tiRelId) on srv.connection
    - Driver: serializes PObject into s7plusTypeInfoCache with status="ok"
    - Vue: polls GET getS7PlusTypeInfo?tiRelId=X every 500ms, max 10s
    - Vue: on status="ok", renders children
    - Vue: on timeout or status="error", shows error message on node
4. Each child node with hasRelation=true gets a "Loading..." placeholder
5. User expands child node -> repeat from step 2 with child's relationId as new tiRelId
```

---

## Data Flow: TagTreeBrowser — Live Values

```
1. Vue: visible leaf tags (hasRelation=false) have a symbolic TIA Portal address
   composed by walking the tree path (same escapeTiaString logic as GUIBrowser)
   e.g. "AlarmData_DB".AlarmSlot[0].Active
2. Vue: query GET /Invoke/auth/getRealtimeTag?address=<symbolic> OR
        query realtimeData by protocolSourceObjectAddress
3. Server: db.collection('realtimeData')
           .find({ protocolSourceObjectAddress: { $in: addressList } })
           .project({ protocolSourceObjectAddress:1, value:1, valueString:1, timeTagAtSource:1 })
4. Vue: shows value next to each leaf node, refreshes every 5s
```

Note: Live values are only available for tags that are already configured in realtimeData (i.e., the driver has a tag entry for that symbolic address). Tags that exist in the type tree but are not in realtimeData show "—" (not configured). This is expected behavior.

---

## Data Flow: Opening TagTreeBrowser from Alarms Viewer

```
1. S7PlusAlarmsViewerPage.vue: originDbName cell rendered as clickable chip/link
2. Operator clicks originDbName (e.g. "AlarmData_DB")
3. Vue: looks up tiRelId from the datablock list (already fetched or re-fetched)
   OR: navigates directly with db_name, and TagTreeBrowserPage resolves tiRelId via
       listS7PlusDatablocks on mount
4. window.open('/#/s7plus-tag-tree?db=AlarmData_DB', '_blank')
5. New browser tab opens with TagTreeBrowserPage displaying that datablock's root
```

The hash-based router (`createWebHashHistory`) means `/#/s7plus-tag-tree?db=X` is a valid standalone URL in a new tab. The page mounts, reads query params, fetches the datablock list to resolve tiRelId, and begins the lazy browse flow.

---

## Vue Router: New Window Behavior

The router uses `createWebHashHistory`. Opening `window.open('/#/s7plus-tag-tree?db=<name>')` in a new tab works correctly:

- The new tab loads the same SPA entry point
- The hash routes to TagTreeBrowserPage
- The query param `?db=<name>` is preserved in `route.query.db`
- The page operates independently (its own JWT cookie, its own fetch calls)

**No router changes beyond adding the two new routes are required for the window.open pattern.** The SPA auth cookie is shared across same-origin tabs.

---

## Patterns to Follow

### Pattern 1: Command Queue Dispatch (from existing alarm ack)

The alarm ack path (MongoCommands.cs lines 97–145) is the template for the browse request dispatch. Key differences for browse:

- ASDU: `"s7plus-browse-request"` (new discriminator)
- address: `tiRelId.toString()`
- No TaskCompletionSource needed — the result is written to MongoDB asynchronously; the frontend polls independently

```csharp
// In ProcessMongoCmd, add a new ASDU branch:
if (asdu == "s7plus-browse-request")
{
    uint tiRelId = uint.Parse(address);
    PObject pObj = null;
    string errorMsg = "";
    try { pObj = srv.connection.getTypeInfoByRelId(tiRelId); }
    catch (Exception ex) { errorMsg = ex.Message; }

    // Serialize pObj to s7plusTypeInfoCache (upsert by tiRelId)
    // ... (see Serialization section)

    // Mark command as delivered
    filter = new BsonDocument("_id", change.FullDocument.id);
    update = new BsonDocument("$set", new BsonDocument {
        { "delivered", true }, { "ack", true },
        { "ackTimeTag", new BsonDateTime(DateTime.Now) }
    });
    await collection.UpdateOneAsync(filter, update);
    break;
}
```

### Pattern 2: Idempotent Cache Check (server-side)

Before inserting a browse command, the `requestS7PlusTypeInfo` endpoint checks whether `s7plusTypeInfoCache` already has a document with `tiRelId` and `status === "ok"`. If so, it returns 200 without inserting — avoids duplicate PLC requests.

```javascript
app.post(OPCAPI_AP + 'auth/requestS7PlusTypeInfo', [authJwt.isAdmin], async (req, res) => {
  const { tiRelId, connectionNumber } = req.body
  // Idempotency check
  const cached = await db.collection('s7plusTypeInfoCache').findOne({ tiRelId, status: 'ok' })
  if (cached) return res.status(200).send({ status: 'already_cached' })
  // Insert browse command
  await db.collection(COLL_COMMANDS).insertOne({
    protocolSourceConnectionNumber: new Double(connectionNumber),
    protocolSourceObjectAddress: tiRelId.toString(),
    protocolSourceASDU: 's7plus-browse-request',
    timeTag: new Date()
  })
  res.status(202).send({ status: 'pending' })
})
```

### Pattern 3: Polling with Exponential Backoff (frontend)

The TagTreeBrowserPage polls `getS7PlusTypeInfo` when a browse request is pending. Use a simple interval poll (500ms, max 20 retries = 10s timeout) rather than exponential backoff — the driver response time is bounded by `WaitForNewS7plusReceived` timeout (5s by default).

```javascript
const waitForTypeInfo = async (tiRelId, maxRetries = 20, intervalMs = 500) => {
  for (let i = 0; i < maxRetries; i++) {
    const res = await fetch(`/Invoke/auth/getS7PlusTypeInfo?tiRelId=${tiRelId}`, { credentials: 'include' })
    const data = await res.json()
    if (data.status === 'ok') return data
    if (data.status === 'error') throw new Error(data.errorMsg || 'Browse failed')
    await new Promise(r => setTimeout(r, intervalMs))
  }
  throw new Error('Timeout waiting for type info')
}
```

### Pattern 4: PObject Serialization

`PObject` is a driver-internal C# class. The driver must serialize the relevant fields into a flat structure for MongoDB storage. Only the fields needed by the frontend need to be stored:

```csharp
var varnames = pObj.VarnameList.Names;  // List<string>
var vartypes = new BsonArray();
for (int i = 0; i < varnames.Count; i++) {
    var vte = pObj.VartypeList.Elements[i];
    var vtDoc = new BsonDocument {
        { "softdatatype", (int)vte.Softdatatype },
        { "hasRelation", vte.OffsetInfoType.HasRelation() },
        { "is1Dim", vte.OffsetInfoType.Is1Dim() },
        { "isMDim", vte.OffsetInfoType.IsMDim() }
    };
    if (vte.OffsetInfoType.HasRelation()) {
        var ioit = (IOffsetInfoType_Relation)vte.OffsetInfoType;
        vtDoc.Add("relationId", (long)(uint)ioit.GetRelationId());
    }
    if (vte.OffsetInfoType.Is1Dim()) {
        var ioitarr = (IOffsetInfoType_1Dim)vte.OffsetInfoType;
        vtDoc.Add("arrayElementCount", (long)ioitarr.GetArrayElementCount());
        vtDoc.Add("arrayLowerBounds", ioitarr.GetArrayLowerBounds());
    }
    // MDim serialization analogous — omit for brevity in PoC
    vartypes.Add(vtDoc);
}
var cacheDoc = new BsonDocument {
    { "tiRelId", (long)(uint)tiRelId },
    { "status", pObj != null ? "ok" : "error" },
    { "varnames", new BsonArray(varnames) },
    { "vartypes", vartypes },
    { "fetchedAt", new BsonDateTime(DateTime.UtcNow) }
};
// Upsert into s7plusTypeInfoCache
var filter = Builders<BsonDocument>.Filter.Eq("tiRelId", (long)(uint)tiRelId);
var options = new ReplaceOptions { IsUpsert = true };
await cacheCollection.ReplaceOneAsync(filter, cacheDoc, options);
```

**Note on tiRelId type:** `tiRelId` is `uint` in the driver but MongoDB's native integer type is Int32 or Int64. Store as `BsonInt64` (long) to avoid overflow for large relIds (values above 2^31). The API endpoint and Vue must treat `tiRelId` consistently (send as number, not string).

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Pre-caching all type info at startup

Fetching type info for every datablock at startup delays `AlarmThread` start and adds PLC round-trips proportional to datablock count. The lazy on-demand pattern achieves the same result with zero startup cost.

**Instead:** Browse on demand via command queue. Cache results in MongoDB so repeat expansions are instant.

### Anti-Pattern 2: Storing full PObject graph as nested BsonDocument

`PObject` can contain nested structs with their own `PObject` children (via `getTypeInfoByRelId` recursive calls in GUIBrowser). Storing the full nested graph at cache time defeats lazy loading and could produce large documents.

**Instead:** Store only the direct children of each tiRelId. Each child with `hasRelation=true` stores the child's `relationId` as the key for a subsequent cache lookup when the user expands that node.

### Anti-Pattern 3: Resolving symbolic addresses in the driver at browse time

The GUIBrowser's `getPlcTagBySymbol` path resolves a full address string to a `PlcTag` with an `ItemAddress`. This involves multiple driver calls and is appropriate for one-off reads. The TagTreeBrowser displays the address string only; live values come from `realtimeData` by string match, not driver resolution.

**Instead:** Compose the symbolic address string in the Vue component by walking the tree node path (same `escapeTiaString` logic as GUIBrowser Form1.cs lines 334–348). Query `realtimeData` by `protocolSourceObjectAddress` string match.

### Anti-Pattern 4: Opening TagTreeBrowser via Vue Router push in the same tab

If the operator opens TagTreeBrowser from the Alarms Viewer in the same tab, the back-navigation experience breaks the alarm monitoring workflow.

**Instead:** Always use `window.open('/#/s7plus-tag-tree?db=<name>', '_blank')` for the Alarms Viewer link. The direct menu navigation (DatablockBrowserPage -> TagTreeBrowserPage) may use `router.push` since that is a deliberate navigation action, not an auxiliary lookup.

### Anti-Pattern 5: Polling realtimeData for all tree nodes

Querying realtimeData for every visible node in a deep tree (potentially hundreds of addresses) is wasteful.

**Instead:** Only show live values for leaf nodes (tags with `hasRelation=false`) that are currently expanded/visible. Refresh every 5s for the visible set only. Non-leaf (structure) nodes show no value.

---

## Scalability Considerations (PoC scope)

| Concern | At PoC scale (single PLC, ~50 DBs) | Notes |
|---------|-----------------------------------|-------|
| s7plusDatablocks size | ~50 documents, trivial | Single replace on reconnect |
| s7plusTypeInfoCache growth | 1 doc per expanded node; bounded by PLC type count | No TTL needed for PoC |
| Browse request latency | 1–3s per uncached node (PLC round-trip) | Cached on second expand |
| Live value refresh | 1 API call per 5s for visible leaf addresses | Acceptable for PoC |
| Concurrent browse requests | Not designed for concurrent users | PoC scope: single operator |

---

## Build Order

Dependencies determine ordering. Build in this sequence:

**Step 1 — Driver: write datablock list to MongoDB on connect**
- Prerequisite for all other features — the datablock list must exist in MongoDB before the API or Vue can use it.
- Modify `ConnectionThread` in Program.cs: after `RelationIdNameMap` is built, write the same `dbInfoList` to `s7plusDatablocks` (replace prior docs for this connectionNumber).
- Verify: MongoDB shows `s7plusDatablocks` documents after driver restart.
- Files: `Program.cs` (MODIFIED)

**Step 2 — API: listS7PlusDatablocks endpoint**
- Simple read from the new collection.
- Verify with curl/Postman that the endpoint returns the datablock list.
- Files: `server_realtime_auth/index.js` (MODIFIED)

**Step 3 — Vue: DatablockBrowserPage + router routes**
- New page that fetches listS7PlusDatablocks and renders a simple table.
- Add `/s7plus-datablocks` and `/s7plus-tag-tree` routes.
- Add dashboard shortcut for DatablockBrowser.
- At this step, TagTreeBrowserPage is a stub that just shows the db name from the query param.
- Verify: operator can navigate to DatablockBrowser from dashboard, see the datablock list.
- Files: `DatablockBrowserPage.vue` (NEW), `TagTreeBrowserPage.vue` (NEW stub), `router/index.js` (MODIFIED), `DashboardPage.vue` (MODIFIED)

**Step 4 — Driver: on-demand browse via command queue**
- Add ASDU="s7plus-browse-request" branch in `ProcessMongoCmd`.
- Serialize PObject to `s7plusTypeInfoCache`.
- Verify: insert a test command into commandsQueue manually; confirm s7plusTypeInfoCache is populated.
- Files: `MongoCommands.cs` (MODIFIED)

**Step 5 — API: getS7PlusTypeInfo + requestS7PlusTypeInfo endpoints**
- Two simple endpoints (cache read + command insert).
- Verify: POST requestS7PlusTypeInfo -> driver processes -> GET getS7PlusTypeInfo returns status="ok".
- Files: `server_realtime_auth/index.js` (MODIFIED)

**Step 6 — Vue: TagTreeBrowserPage lazy expand**
- Implement full lazy tree logic: expand node -> requestS7PlusTypeInfo -> poll getS7PlusTypeInfo -> render children.
- Verify: operator can expand a datablock and see its tags.
- Files: `TagTreeBrowserPage.vue` (MODIFIED)

**Step 7 — Vue: live values in TagTreeBrowserPage**
- Add getRealtimeValues API endpoint (or reuse existing realtimeData query pattern).
- Show live values for configured leaf tags.
- Verify: tags present in realtimeData show current value next to tree node.
- Files: `TagTreeBrowserPage.vue` (MODIFIED), `server_realtime_auth/index.js` (MODIFIED if new endpoint needed)

**Step 8 — Vue: Alarms Viewer integration**
- Make originDbName a clickable link that opens TagTreeBrowserPage in a new tab.
- Verify: click originDbName in alarm row -> new tab opens showing that datablock's tag tree.
- Files: `S7PlusAlarmsViewerPage.vue` (MODIFIED)

---

## Integration Points Summary

| Boundary | Communication | New vs Existing |
|----------|---------------|-----------------|
| Driver -> s7plusDatablocks | `deleteMany + insertMany` on each connect | NEW collection, NEW write path |
| Driver -> s7plusTypeInfoCache | `ReplaceOneAsync` (upsert) in ProcessMongoCmd | NEW collection, NEW write path in existing handler |
| Driver -> commandsQueue | existing change stream read; NEW ASDU branch | EXISTING collection, MODIFIED handler |
| API -> s7plusDatablocks | `find().sort()` for listS7PlusDatablocks | NEW endpoint |
| API -> s7plusTypeInfoCache | `findOne` for getS7PlusTypeInfo | NEW endpoint |
| API -> commandsQueue | `insertOne` for requestS7PlusTypeInfo | NEW endpoint (existing collection) |
| API -> realtimeData | `find({ $in: addressList })` for live values | NEW endpoint (existing collection) |
| Vue -> API (datablocks) | `fetch()` on mount | NEW page |
| Vue -> API (browse) | `fetch()` request + poll | NEW page |
| Vue -> API (live values) | `fetch()` every 5s | NEW page |
| S7PlusAlarmsViewerPage -> TagTreeBrowserPage | `window.open()` with hash URL | MODIFIED existing page |

---

## Sources

- Direct code inspection: `S7CommPlusGUIBrowser/Form1.cs` — lazy browse implementation (GetListOfDatablocks at connect, getTypeInfoByRelId on expand, PObject/VarnameList/VartypeList traversal)
- Direct code inspection: `S7CommPlusDriver/S7CommPlusConnection.cs` lines 970–983, 1184–1297 — getTypeInfoByRelId implementation, DatablockInfo struct, GetListOfDatablocks
- Direct code inspection: `S7CommPlusClient/Program.cs` — ConnectionThread structure; GetListOfDatablocks call at line 286; RelationIdNameMap build pattern
- Direct code inspection: `S7CommPlusClient/MongoCommands.cs` lines 97–145 — ASDU dispatch pattern; alarm ack as template for browse request dispatch
- Direct code inspection: `S7CommPlusClient/Common.cs` — S7CP_connection fields; existing collection names; commandsQueue pattern
- Direct code inspection: `server_realtime_auth/index.js` lines 362–450 — existing S7Plus alarm endpoints; HTTP pattern; authJwt.isAdmin guard
- Direct code inspection: `AdminUI/src/router/index.js` — createWebHashHistory; existing route pattern
- Direct code inspection: `AdminUI/src/components/DashboardPage.vue` — shortcut card pattern (s7plusAlarms entry at lines 88–92)
- Direct code inspection: `AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — originDbName column rendering; existing window/tab patterns
- `.planning/PROJECT.md` — v1.4 target features; v1.3 current state; key decisions

---

*Architecture research for: S7CommPlus Tag Tree Browser — DatablockBrowser + TagTreeBrowser (v1.4)*
*Researched: 2026-03-26*

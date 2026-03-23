# Stack Research

**Domain:** S7CommPlus alarm origin lookup + alarm delete — additions to existing v1.1 driver (v1.2 milestone)
**Researched:** 2026-03-23
**Confidence:** HIGH — all findings verified directly from submodule source code and existing project files

---

## Summary of New Stack Needs

v1.2 adds two capabilities to an already-validated stack. **No new dependencies are required.** Every API needed exists in the current codebase. The sections below cover only what is new or changed for v1.2.

---

## 1. PLC Browse at Startup — DB/FB Name Lookup

### API available: `GetListOfDatablocks` + `ExploreASAlarms`

**Source:** `S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs`, lines 1191–1313

`GetListOfDatablocks(out List<DatablockInfo> dbInfoList)` is a public method on `S7CommPlusConnection`. It issues an `ExploreRequest` to `Ids.NativeObjects_thePLCProgram_Rid` filtered on `Ids.DB_Class_Rid`, retrieves `ObjectVariableTypeName` for each DB, and populates:

```csharp
public class DatablockInfo
{
    public string db_name;           // symbolic name, e.g. "AlarmDB"
    public UInt32 db_number;         // numeric DB number
    public UInt32 db_block_relid;    // 0x8a0e0000 | db_number
    public UInt32 db_block_ti_relid; // TypeInfo RID (for variable browsing)
}
```

**Also available:** `ExploreASAlarms(ref Dictionary<ulong, AlarmData> Alarms, int languageId)` in the same file (line 42 of BrowseAlarms.cs). The second explore pass in this method queries `NativeObjects_thePLCProgram_Rid` and requests `Ids.ObjectVariableTypeName` on each sub-object. Each sub-object's `ob.RelationId` is the same RelationId that ends up encoded in the upper 32 bits of `CpuAlarmId`:

```
CpuAlarmId (ulong, 64-bit):
  bits 63..32 = RelationId of the source block (FB instance / DB)
  bits 31..16 = Alid (alarm number within that block)
  bits 15..0  = 0
```

Confirmed in `BrowseAlarms.cs` line 414:
```csharp
return ((ulong)(RelationId) << 32) | ((ulong)(MultipleStai.Alid) << 16);
```

### What to build

At driver startup, after the tag-browse phase, walk the PLCProgram sub-objects to build a `Dictionary<uint, string>` of `RelationId → block name`. This can be done with either:

- `GetListOfDatablocks` for DB-area blocks (`0x8a0e` prefix); or
- a targeted `ExploreRequest` to `NativeObjects_thePLCProgram_Rid` requesting `ObjectVariableTypeName` for all sub-objects (this is already what the second loop in `ExploreASAlarms` does)

**Recommended approach for PoC simplicity:** Call `GetListOfDatablocks` (already used in the tag-browse path) and build the map from `db_block_relid → db_name`. When writing an alarm event, extract the block RelationId with `(uint)(dai.CpuAlarmId >> 32)` and look it up. If the DB area (`0x8a0e`) matches, the name is available. Store the result as `blockName` in the MongoDB document at write time — no separate lookup later.

### Integration point

Add one field to `S7CP_connection` in `Common.cs`:

```csharp
public Dictionary<uint, string> RelationIdToBlockName = new Dictionary<uint, string>();
```

Populate it in `ConnectionThread` right after the existing `GetListOfDatablocks` / tag-address-cache phase, before `AlarmThread` starts. Use the same `srv.connection` that is already open at that point. Pass `srv` into `AlarmThread` (already happens) so `BuildAlarmDocument` can consult the dict.

### Protocol note

`GetListOfDatablocks` uses `ExploreRequest`, which is already exercised every startup for tag browsing. No new protocol functions are needed. Timeout behaviour matches the existing pattern: `WaitForNewS7plusReceived(m_ReadTimeout)` with 5000 ms default.

---

## 2. Storing RelationID + Block Name in MongoDB

### What to add to `BuildAlarmDocument` in `AlarmThread.cs`

Two new fields in the `BsonDocument` returned by `BuildAlarmDocument`:

```csharp
{ "relationId",  (long)(dai.CpuAlarmId >> 32) },
{ "blockName",   srv.RelationIdToBlockName.TryGetValue((uint)(dai.CpuAlarmId >> 32), out var bn) ? bn : "" },
```

`relationId` is stored as `long` (BsonInt64) because `uint` fits safely in 64-bit. `blockName` is an empty string when the block is not found in the map (e.g. FB types not present in the DB browse result).

### Storage decision: in-memory dictionary (NOT MongoDB collection, NOT JSON file)

| Option | Verdict | Reason |
|--------|---------|--------|
| In-memory `Dictionary<uint, string>` on `S7CP_connection` | **USE THIS** | Zero latency at alarm-write time; rebuilt on every reconnect (reflects PLC program changes automatically); consistent with existing `AddressCache` pattern in `Common.cs` line 93 |
| Separate MongoDB `s7plusBlockNames` collection | Avoid | Adds a write phase at startup, a read at alarm-write time, and stale data risk if PLC program changes without driver restart |
| JSON file on disk | Avoid | Manual maintenance; out of step with automated browse; not how json-scada stores protocol metadata |

---

## 3. DELETE Endpoint — Node.js server_realtime_auth

**Existing pattern to mirror:** `POST /Invoke/auth/ackS7PlusAlarm` (index.js lines 372–407) uses `db.collection(...)` (native MongoDB driver, not Mongoose) with `insertOne`.

**New endpoints needed:**

### DELETE single row

```js
app.delete(
  OPCAPI_AP + 'auth/deleteS7PlusAlarm',
  [authJwt.isAdmin],
  async (req, res) => {
    try {
      if (!db) return res.status(200).send({ error: 'DB not connected' })
      const { cpuAlarmId } = req.body
      if (!cpuAlarmId) return res.status(400).send({ error: 'Missing cpuAlarmId' })
      const result = await db.collection('s7plusAlarmEvents')
        .deleteOne({ cpuAlarmId: cpuAlarmId.toString() })
      res.status(200).send({ ok: true, deleted: result.deletedCount })
    } catch (err) {
      Log.log(err)
      res.status(200).send({ error: err.message })
    }
  }
)
```

### DELETE bulk (filtered rows)

```js
app.delete(
  OPCAPI_AP + 'auth/deleteS7PlusAlarms',   // plural
  [authJwt.isAdmin],
  async (req, res) => {
    try {
      if (!db) return res.status(200).send({ error: 'DB not connected' })
      const { cpuAlarmIds } = req.body      // array of strings
      if (!Array.isArray(cpuAlarmIds) || cpuAlarmIds.length === 0)
        return res.status(400).send({ error: 'Missing or empty cpuAlarmIds array' })
      const result = await db.collection('s7plusAlarmEvents')
        .deleteMany({ cpuAlarmId: { $in: cpuAlarmIds.map(String) } })
      res.status(200).send({ ok: true, deleted: result.deletedCount })
    } catch (err) {
      Log.log(err)
      res.status(200).send({ error: err.message })
    }
  }
)
```

**Why `deleteMany` with explicit ID list (not a filter object):**

The Vue component already computed `filteredAlarms` client-side (status filter + alarm class filter). Sending the IDs of those filtered rows avoids translating Vue filter state into a MongoDB query — the server stays simple, the client sends exactly what it wants deleted. Consistent with the PoC simplicity constraint.

**MongoDB driver version:** `mongodb ^7.0.0` (package.json). `deleteOne` / `deleteMany` API is stable since v4+. `result.deletedCount` is available in all v4+ versions. No version concern.

**HTTP verb:** `DELETE` is semantically correct. The existing `ackS7PlusAlarm` uses `POST` because it creates a command record (a write). Delete is direct data removal — `DELETE` is the right verb. Vue's `fetch` supports `method: 'DELETE'` natively.

---

## 4. Vue 3 Delete Button + Bulk Delete

### Per-row Delete button

Mirror the existing Ack button pattern (S7PlusAlarmsViewerPage.vue lines 47–54). Add a new column slot:

```vue
<template #[`item.deleteAction`]="{ item }">
  <v-btn
    size="x-small"
    variant="tonal"
    color="error"
    :loading="pendingDeletes.has(item.cpuAlarmId)"
    @click="deleteAlarm(item.cpuAlarmId)"
  >
    Delete
  </v-btn>
</template>
```

```js
const pendingDeletes = ref(new Set())

const deleteAlarm = async (cpuAlarmId) => {
  pendingDeletes.value = new Set([...pendingDeletes.value, cpuAlarmId])
  try {
    await fetch('/Invoke/auth/deleteS7PlusAlarm', {
      method: 'DELETE',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ cpuAlarmId }),
    })
    alarms.value = alarms.value.filter(a => a.cpuAlarmId !== cpuAlarmId)
  } finally {
    pendingDeletes.value = new Set([...pendingDeletes.value].filter(id => id !== cpuAlarmId))
  }
}
```

**Key difference from Ack:** After delete succeeds, remove the row from `alarms.value` immediately. The Ack button leaves the row in place and waits for the next poll to confirm `ackState: true`. Delete should vanish the row at once — matches operator expectation.

### Bulk Delete Filtered

Add a button above the table (not inside a row):

```vue
<v-btn
  color="error"
  variant="tonal"
  :disabled="filteredAlarms.length === 0 || bulkDeletePending"
  :loading="bulkDeletePending"
  @click="deleteFiltered"
>
  Delete Filtered ({{ filteredAlarms.length }})
</v-btn>
```

```js
const bulkDeletePending = ref(false)

const deleteFiltered = async () => {
  const ids = filteredAlarms.value.map(a => a.cpuAlarmId)
  if (ids.length === 0) return
  bulkDeletePending.value = true
  try {
    await fetch('/Invoke/auth/deleteS7PlusAlarms', {
      method: 'DELETE',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ cpuAlarmIds: ids }),
    })
    const idSet = new Set(ids)
    alarms.value = alarms.value.filter(a => !idSet.has(a.cpuAlarmId))
  } finally {
    bulkDeletePending.value = false
  }
}
```

**Why `filteredAlarms.value` (not `alarms.value`):** The user applies status/class filters, then clicks "Delete Filtered" — they expect only visible rows to be removed. `filteredAlarms` is already the computed subset. This avoids any need to replicate filter logic server-side.

**Vuetify version:** v3 (already in use). `v-btn` with `:loading` prop is Vuetify 3 standard. No library additions needed.

---

## Core Technologies (unchanged from v1.1)

| Technology | Version | Purpose |
|------------|---------|---------|
| S7CommPlusDriver | submodule | PLC protocol — `GetListOfDatablocks`, `ExploreASAlarms`, `S7CommPlusConnection` |
| MongoDB.Driver (C#) | 3.4.2 | Alarm event document writes — new `relationId` + `blockName` fields via existing `InsertOneAsync` |
| mongodb (Node.js) | ^7.0.0 | `deleteOne` / `deleteMany` in new delete endpoints |
| Vue 3 + Vuetify 3 | existing | Per-row delete button + bulk delete toolbar button |

## Supporting Libraries — No New Additions

All required APIs (`GetListOfDatablocks`, `ExploreASAlarms`, `deleteOne`, `deleteMany`, `v-btn :loading`) are present in already-installed versions. No `npm install` or NuGet package changes are needed for v1.2.

---

## What NOT to Add

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| Separate `s7plusBlockNames` MongoDB collection | Extra write at startup, stale data risk, unnecessary complexity for a PoC | In-memory `Dictionary<uint, string>` on `S7CP_connection` — matches `AddressCache` pattern |
| New S7CommPlusDriver NuGet package | Driver is a submodule built locally; protocol APIs already exist in source | Use `GetListOfDatablocks` + `ExploreASAlarms` directly from existing submodule |
| `GetVarSubSL` / `GetVarSubstreamed` for block discovery | These are variable-value read APIs, not block browse APIs | `ExploreRequest` via `GetListOfDatablocks` is the correct browse path |
| Full PLCProgram variable tree browse for origin lookup | Fetches all variable type info — expensive and irrelevant for just block names | Single `GetListOfDatablocks` call is sufficient |
| Axios or other HTTP libraries in Vue | Unnecessary dependency; `fetch` is already used throughout the viewer | Continue with `fetch` |
| Confirmation dialog for per-row delete | Adds friction; not in scope for PoC | Simple button click, row vanishes immediately |
| MongoDB index on `cpuAlarmId` field | PoC with limited data volume; existing pattern has no indexes on this collection | No index needed at PoC scale |

---

## Version Compatibility

| Component | Version | Compatibility Note |
|-----------|---------|-------------------|
| MongoDB.Driver (C#) | 3.4.2 | `InsertOneAsync`, `UpdateManyAsync` already in use; `DeleteOneAsync`/`DeleteManyAsync` available with identical async pattern — no version bump needed |
| mongodb (Node.js) | ^7.0.0 | `deleteOne`/`deleteMany` return `{ deletedCount }` — stable API since v4 |
| Vue 3 | existing | `ref`, `computed`, `fetch` — no new Vue APIs needed |
| Vuetify 3 | existing | `v-btn :loading` prop — standard Vuetify 3; no new components needed |

---

## Sources

- `S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs` lines 1183–1313 — `GetListOfDatablocks` implementation and `DatablockInfo` struct (HIGH confidence, source code)
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/BrowseAlarms.cs` lines 408–416 — `AlarmData.GetCpuAlarmId()` confirming `RelationId << 32` encoding (HIGH confidence, source code)
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsDai.cs` — `FromNotificationObject` showing `CpuAlarmId` comes from `DAI_CPUAlarmID` attribute (HIGH confidence, source code)
- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — `BuildAlarmDocument` pattern for adding new fields to alarm documents (HIGH confidence, source code)
- `json-scada/src/S7CommPlusClient/Common.cs` line 93 — `AddressCache` field confirming per-connection in-memory dict pattern (HIGH confidence, source code)
- `json-scada/src/server_realtime_auth/index.js` lines 350–407 — existing list/ack endpoints showing `db.collection(...).find()/insertOne()` pattern with `authJwt.isAdmin` middleware (HIGH confidence, source code)
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — existing Ack button + `pendingAcks` Set pattern to mirror (HIGH confidence, source code)
- `json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` — MongoDB.Driver 3.4.2 confirmed (HIGH confidence, project file)
- `json-scada/src/server_realtime_auth/package.json` — mongodb ^7.0.0 confirmed (HIGH confidence, project file)

---

*Stack research for: S7CommPlus Alarm Origin & Cleanup (v1.2)*
*Researched: 2026-03-23*

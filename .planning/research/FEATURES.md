# Feature Research

**Domain:** S7CommPlus alarm origin enrichment + alarm history management (v1.2)
**Researched:** 2026-03-23
**Confidence:** HIGH — all findings derived directly from the live codebase and protocol source

---

## Context: What Exists (v1.1 Baseline)

Already shipped. Not in scope for this milestone but required as dependencies:

- `AlarmThread.cs` receives `Notification` PDUs, calls `AlarmsDai.FromNotificationObject()`, writes `BsonDocument` to `s7plusAlarmEvents` MongoDB collection.
- `BuildAlarmDocument()` stores 15 fields per document. `cpuAlarmId` is stored as a string (`dai.CpuAlarmId.ToString()`).
- `S7PlusAlarmsViewerPage.vue` has 11 columns. Ack button per row. 5s auto-refresh. Status and alarm class filters.
- Backend: `GET /Invoke/auth/listS7PlusAlarms` (returns last 200, sorted `createdAt` desc, projects `{ _id: 0 }`) and `POST /Invoke/auth/ackS7PlusAlarm`.

---

## Domain Answers (Research Questions)

### 1. How do SCADA systems show alarm origin?

Standard practice in industrial HMI/SCADA (TIA Portal WinCC, Intouch, Ignition):

- **DB name** is the primary identifier operators recognize. It matches the name visible in TIA Portal (e.g. `"MotorControl_DB"`, `"SafetyFB_DB3"`). Displayed in a dedicated "Origin" or "Source Block" column.
- **DB number** (e.g. `DB5`) is secondary context, useful for cross-referencing hardware documentation. Low value on its own.
- **FB type name** (the backing FB type if the DB is an instance DB) is engineering-level detail. Operators do not use it. Not needed.
- **Symbolic variable path** within the DB (e.g. `MotorDB.AlarmBit_1`) is not available from alarm notification metadata alone. Matching `Alid` to variables would require a full DB browse. Out of scope.

**Recommendation:** Single "Origin DB" column showing `db_name` (symbolic name from TIA Portal). `db_number` as optional secondary. No FB type needed.

### 2. What does RelationID represent in the alarm notification PDU?

Confirmed from `BrowseAlarms.cs` and `AlarmThread.cs`:

- `cpuAlarmId = (ulong)RelationId << 32 | (ulong)Alid << 16`
- The upper 32 bits of `cpuAlarmId` IS the `RelationId`.
- `RelationId` structure: upper 16 bits = area code (`0x8a0e` for user data blocks), lower 16 bits = DB number.
- One `RelationId` per DB instance. Multiple alarm definitions within the same DB share the same `RelationId` — distinguished by `Alid` in bits 16–31.
- **Stability:** Stable for the life of the PLC program. Changes only if the DB is deleted and recreated (e.g. after a full program download), which reassigns the RID. For a PoC targeting a fixed PLC program, the map built at startup is valid for the entire session.
- `RelationId` is NOT per-alarm-class. It is per-DB-instance.

**Implication for storage:** `relationId` can be extracted from `cpuAlarmId` at any time (`(uint)(cpuAlarmId >> 32)`), so storing it explicitly is optional. However, explicit storage makes the MongoDB field trivially queryable without bit arithmetic in JavaScript.

### 3. How does GetListOfDatablocks work?

Confirmed from `S7CommPlusConnection.cs` (lines 1191–1314). The method already exists and is proven in `S7CommPlusGUIBrowser`.

**What it does:**
1. Sends `ExploreRequest` for `Ids.NativeObjects_thePLCProgram_Rid`, filtered by `Ids.DB_Class_Rid` (instance-of filter), requesting `Ids.ObjectVariableTypeName` per object.
2. Parses response: each DB object yields `RelationId` = `db_block_relid` and symbolic name = `db_name`.
3. Second pass: reads `LID=1` from each `db_block_relid` to get `db_block_ti_relid` (TypeInfo RID — only needed for variable browse, NOT needed for origin lookup).
4. Returns `List<DatablockInfo>`: `db_name` (string), `db_number` (uint = lower 16 bits of relid), `db_block_relid` (uint = RelationId).
5. DBs only in load memory get `db_block_ti_relid = 0` and are excluded. Alarm DBs must be in work memory to fire alarms — this exclusion does not affect the origin map.

**Mapping:** `RelationId → db_name` is a direct lookup. The key in the map is `db_block_relid`, which equals `(uint)(cpuAlarmId >> 32)`.

**Cost:** Two network round-trips (one ExploreRequest + one ReadValues batch). Adds ~100–500ms to startup. Acceptable.

**Limitation:** The `db_block_ti_relid` lookup (second pass, `ReadValues`) is unnecessary for origin mapping. The method can be used as-is, or a lighter single-pass variant can be added. Using it as-is avoids new code in the driver library.

### 4. Delete alarm history UX patterns

Standard SCADA alarm log management patterns (TIA Portal WinCC, Ignition, Wonderware):

- **Per-row delete** is uncommon in traditional SCADA (audit trail concerns), but for a PoC engineering log it is practical and expected. Mirrors the per-row Ack button already present.
- **Bulk delete** with "clear visible" (delete all currently filtered rows) is a standard pattern in log viewers. The current filter state defines the scope.
- **No soft-delete:** A PoC has no audit trail requirement. Hard `deleteOne` / `deleteMany` from MongoDB is correct.
- **No confirmation modal on per-row delete:** Adds friction in a dense table. A single click is sufficient. Button label "Delete" is unambiguous.
- **Bulk delete** should have a distinct visual indicator showing row count (e.g. "Delete Filtered (12)") to avoid accidental mass deletion. No blocking modal needed for a PoC.

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features the v1.2 milestone explicitly requires. Missing any of these makes the milestone incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Store `relationId` in `s7plusAlarmEvents` documents | Foundation for origin lookup; without it the Vue viewer cannot show DB name without per-alarm client-side computation | LOW | Computed in `BuildAlarmDocument`: `(uint)(dai.CpuAlarmId >> 32)`. Zero protocol changes. |
| Build `relationId → db_name` map at AlarmThread startup | Origin display is impossible without the name map; map is stable for the session | MEDIUM | Calls existing `conn.GetListOfDatablocks()` on the alarm connection before `AlarmSubscriptionCreate()`. Stores result as `Dictionary<uint, string>` on `S7CP_connection` (or passed into AlarmThread). |
| Store `originDbName` in `s7plusAlarmEvents` documents | Operators need the DB name in every alarm record; avoids a client-side lookup on every row render | LOW | Map lookup by `relationId` at `BuildAlarmDocument` time. Falls back to `"DB" + db_number` (derivable from `relationId & 0xFFFF`) if not in map. |
| "Origin DB" column in `S7PlusAlarmsViewerPage.vue` | Operators need to see which DB produced each alarm | LOW | Add one entry to `headers` array. Render `item.originDbName`. No new filter required for MVP. |
| Include `_id` in `listS7PlusAlarms` response | Per-row delete requires a stable document identifier; current projection excludes `_id` | LOW | Change `{ _id: 0 }` to `{}` in the `find` projection, or explicitly include `{ _id: 1 }`. Convert ObjectId to string for JSON transport. |
| `DELETE /Invoke/auth/deleteS7PlusAlarm` endpoint | Backend must remove a single document by `_id` | LOW | `db.collection('s7plusAlarmEvents').deleteOne({ _id: new ObjectId(id) })`. Auth guard matches existing pattern. |
| Per-row Delete button in `S7PlusAlarmsViewerPage.vue` | Operators need to remove individual stale or test entries | LOW | Mirrors Ack button template slot. Calls delete endpoint with `item._id`. Removes row optimistically from local `alarms` array on success. |
| `POST /Invoke/auth/deleteS7PlusAlarms` bulk endpoint | Backend must remove all documents matching current filter | MEDIUM | Translates `alarmState` and `alarmClassName` filter params from request body to a MongoDB query document. `deleteMany` on the result. |
| "Delete Filtered (N)" button in `S7PlusAlarmsViewerPage.vue` | Operators need to clear all currently visible rows in one action | LOW | Button renders `filteredAlarms.length` as count. Sends current filter values (`statusFilter`, `alarmClassFilter`) to bulk delete endpoint. Calls `fetchAlarms()` after success. |

### Differentiators (Competitive Advantage)

Features beyond the strict requirements that would strengthen the PoC demonstration.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| DB number secondary display (e.g. "MotorDB [DB5]") | Gives operators both symbolic and numeric reference without extra storage | LOW | Derivable: `db_number = relationId & 0xFFFF`. Pure display formatting in Vue. Only add if DB names alone are ambiguous. |
| "Origin DB" filter dropdown in viewer | Lets operators isolate alarms by source subsystem when multiple DBs produce alarms | LOW | Same client-side filter pattern as `alarmClassName` filter. Only valuable if PoC PLC has multiple alarm-producing DBs. |

### Anti-Features (Commonly Requested, Often Problematic)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Show FB type name (backing FB of instance DB) | Seems like useful engineering context | Requires a separate `GetTypeInformation()` browse per DB after `GetListOfDatablocks()` — doubles startup cost. FB name is not in `DatablockInfo` and is not meaningful to operators. | Show `db_name` only. Engineers look up FB type in TIA Portal. |
| Symbolic variable path within DB (e.g. `MotorDB.AlarmBit_1`) | Maximum diagnostic detail for fault-finding | Not in alarm notification metadata. Would require matching `Alid` field against variable type info — full DB browse at startup, large overhead, complex Alid-to-variable matching. Out of scope for PoC. | Show DB name. The alarm text already describes the condition. |
| Soft-delete / recycle bin | "What if I delete the wrong one?" | Doubles schema complexity (deleted flag, query filter on every read). PoC has no audit trail requirement. | Hard delete. Viewer shows current live log only. |
| Confirmation modal on per-row delete | Prevent accidental single-row deletion | Adds two clicks per deletion in a dense engineering table. PoC users understand consequences. | No modal. Button label is "Delete". Optimistic removal gives immediate feedback. |
| Persist `relationId → db_name` map to MongoDB | Survive driver restart without PLC re-browse | PoC with a fixed PLC program has no restart scenario requiring persistence. Map rebuild at startup is always authoritative. Persisting adds write/read complexity and a staleness problem if PLC program changes. | Rebuild map at every driver startup. In-memory only. |
| Auto-resubscribe alarm subscription after failure | Robustness for production | Known technical debt item, out of scope for this milestone. | Manual driver restart for PoC. |

---

## Feature Dependencies

```
[GetListOfDatablocks() at AlarmThread startup]
    └──produces──> [Dictionary<uint, string>: relationId → db_name]
                       └──consumed by──> [BuildAlarmDocument: originDbName lookup]
                                             └──stores──> [originDbName in MongoDB]
                                             └──stores──> [relationId in MongoDB]

[originDbName in MongoDB documents]
    └──enables──> [Origin DB column in Vue viewer]

[_id included in listS7PlusAlarms response]
    └──enables──> [Per-row Delete button in Vue]
                      └──requires──> [DELETE /deleteS7PlusAlarm endpoint]

[filteredAlarms computed property (already exists in v1.1)]
    └──enables──> [Delete Filtered (N) button in Vue]
                      └──requires──> [POST /deleteS7PlusAlarms endpoint]
```

### Dependency Notes

- **`_id` projection change:** Current `listS7PlusAlarms` endpoint projects `{ _id: 0 }`. Changing to `{}` (or explicitly including `_id: 1`) is a one-line change. MongoDB ObjectId must be serialized as a string for JSON (`_id.toString()`). Vue receives it and passes it back to the delete endpoint.

- **Startup map placement:** `AlarmThread` currently: connect → `AlarmSubscriptionCreate()` → receive loop. The DB name map must be built before the receive loop starts, but after connection is established. Correct placement: after `Connect()` and before `AlarmSubscriptionCreate()`. Uses the same `alarmConn` connection. No third connection needed.

- **`GetListOfDatablocks()` second pass is unnecessary:** The method's second pass (reading `LID=1` for `db_block_ti_relid`) is needed only for variable browse, not for origin mapping. The method can be called as-is and the `db_block_ti_relid` field ignored. Alternatively, a lighter helper can be extracted — but calling the existing method avoids new driver library changes.

- **No new C# alarm protocol work:** Delete is purely a Node.js/MongoDB operation. The C# driver writes alarm events; it never reads or deletes them. No AlarmThread changes needed for the delete feature.

- **Bulk delete filter translation:** The Vue-side filters are `statusFilter` (values: "All", "Incoming", "Outgoing") and `alarmClassFilter` (values: "All" or an `alarmClassName` string). The backend translates: "Incoming" → `{ alarmState: "Coming" }`, "Outgoing" → `{ alarmState: "Going" }`, "All" → no alarmState filter. `alarmClassFilter !== "All"` → `{ alarmClassName: value }`. These directly match the MongoDB field names already in documents.

---

## MVP Definition (v1.2)

### Launch With

- [ ] `relationId` stored in `s7plusAlarmEvents` documents — computed from `cpuAlarmId >> 32`
- [ ] Startup DB name map built via `conn.GetListOfDatablocks()` on alarm connection before subscription
- [ ] `originDbName` stored in every new alarm document (map lookup at insert time, graceful fallback)
- [ ] "Origin DB" column added to `S7PlusAlarmsViewerPage.vue`
- [ ] `_id` included in `listS7PlusAlarms` response (remove `{ _id: 0 }` projection)
- [ ] `DELETE /Invoke/auth/deleteS7PlusAlarm` endpoint — single-row delete by `_id`
- [ ] Per-row Delete button in Vue viewer (mirrors Ack button pattern)
- [ ] `POST /Invoke/auth/deleteS7PlusAlarms` endpoint — bulk delete by filter
- [ ] "Delete Filtered (N)" button in Vue viewer

### Add After Validation

- [ ] "Origin DB" filter dropdown in viewer — only if multi-DB PLC scenario is needed
- [ ] DB number secondary display `[DB5]` — cosmetic, add if db_name alone is ambiguous

### Future Consideration (v2+)

- [ ] FB type name column — requires full type info browse per DB, engineering-level detail
- [ ] Variable path within DB — requires Alid-to-variable matching, high complexity
- [ ] Alarm log retention policy (auto-expire after N days) — operational need, not PoC

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| `relationId` stored in MongoDB | MEDIUM | LOW | P1 |
| Startup DB name map (GetListOfDatablocks) | HIGH | MEDIUM | P1 |
| `originDbName` stored in MongoDB | HIGH | LOW | P1 |
| Origin DB column in Vue | HIGH | LOW | P1 |
| `_id` in list response | HIGH (enables delete) | LOW | P1 |
| DELETE single endpoint + per-row button | HIGH | LOW | P1 |
| Bulk delete endpoint + Delete Filtered button | HIGH | MEDIUM | P1 |
| DB number secondary display | LOW | LOW | P2 |
| Origin DB filter dropdown | LOW | LOW | P2 |

---

## Sources

- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/BrowseAlarms.cs` — `AlarmData.RelationId`, `GetCpuAlarmId()` — confirms RelationId encoding in cpuAlarmId
- `S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs` lines 1183–1314 — `GetListOfDatablocks()` implementation and `DatablockInfo` struct (db_name, db_number, db_block_relid)
- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — `BuildAlarmDocument()`, current MongoDB schema, alarm receive loop structure
- `json-scada/src/server_realtime_auth/index.js` lines 350–407 — existing alarm endpoints, `{ _id: 0 }` projection, ack pattern to mirror for delete
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — existing headers, filters, Ack button template slot to mirror for Delete
- `.planning/PROJECT.md` — confirmed active requirements for v1.2, out-of-scope constraints
- `S7CommPlusGUIBrowser/Form1.cs` lines 60–75 — `GetListOfDatablocks()` usage reference (proven in GUI browser tool)

---

*Feature research for: S7CommPlus alarm origin and history cleanup (v1.2)*
*Researched: 2026-03-23*

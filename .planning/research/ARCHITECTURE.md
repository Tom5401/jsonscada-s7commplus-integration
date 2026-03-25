# Architecture Research

**Domain:** S7CommPlus Alarm Viewer Enhancements (v1.3)
**Researched:** 2026-03-25
**Confidence:** HIGH — based on direct code inspection of all affected files

---

## System Overview

```
+---------------------------------------------------------------------+
|                           PLC Layer                                 |
|  S7-1200 / S7-1500 -- alarm events (native S7CommPlus protocol)    |
+----------------------------------+----------------------------------+
                                   | WaitForAlarmNotification / GetActiveAlarms
+----------------------------------v----------------------------------+
|                   Driver Layer (C# / .NET)                          |
|  S7CommPlusClient / AlarmThread.cs                                  |
|  BuildAlarmDocument(AlarmsDai, S7CP_connection)                     |
|    priority       = dai.HmiInfo.Priority  [ALREADY STORED]         |
|    alarmClass     = dai.HmiInfo.AlarmClass [ALREADY STORED]        |
|  + isAcknowledgeable = (alarmClass == 33)  [NEW]                   |
|  + alarmText  wrapped with ResolveAlarmText()  [NEW]               |
|  + infoText   wrapped with ResolveAlarmText()  [NEW]               |
|  -> InsertOneAsync -> s7plusAlarmEvents                             |
|                                                                     |
|  MongoCommands.cs: watches rtCommands, enqueues PendingAlarmAck    |
+----------------------------------+----------------------------------+
                                   | MongoDB
+----------------------------------v----------------------------------+
|                         Data Layer (MongoDB)                        |
|  s7plusAlarmEvents  (alarm event log, append-only from driver)     |
|  rtCommands         (ack dispatch queue, insert by API)            |
+----------------------------------+----------------------------------+
                                   | REST / JSON  (fetch)
+----------------------------------v----------------------------------+
|                   API Layer (Node.js)                               |
|  server_realtime_auth / index.js                                    |
|  GET  listS7PlusAlarms    -- .limit(200) REMOVED [MODIFIED]        |
|  POST ackS7PlusAlarm      -- single alarm, unchanged               |
|  POST deleteS7PlusAlarms  -- unchanged                             |
+----------------------------------+----------------------------------+
                                   | fetch() every 5s
+----------------------------------v----------------------------------+
|                   UI Layer (Vue 3 / Vuetify)                        |
|  S7PlusAlarmsViewerPage.vue                                         |
|  alarms ref <- fetchAlarms() every 5s                              |
|  filteredAlarms computed (status, alarmClass, + connectionName)    |
|  v-data-table: built-in pagination, :items-per-page=50             |
|  + priority column, combined timestamp column  [NEW columns]       |
|  + source PLC filter (connectionName)  [NEW filter]               |
|  + Ack All button (sequential calls to ackS7PlusAlarm)  [NEW]     |
|  + page ref preserved across refresh  [NEW]                        |
+---------------------------------------------------------------------+
```

---

## Component Responsibilities

| Component | Responsibility | File |
|-----------|---------------|------|
| `AlarmThread.cs` AlarmThread() | Receive PLC notifications, drain PendingAcks, call BuildAlarmDocument, InsertOneAsync | `S7CommPlusClient/AlarmThread.cs` |
| `AlarmThread.cs` BuildAlarmDocument() | Construct BsonDocument from AlarmsDai + S7CP_connection; single point of truth for alarm schema | `AlarmThread.cs` lines 202-263 |
| `AlarmThread.cs` ResolveAlarmText() | Substitute @N%f@ placeholders in a single text string using AssociatedValues | `AlarmThread.cs` lines 270-293 |
| `MongoCommands.cs` | Watch rtCommands; on `s7plus-alarm-ack`, enqueue PendingAlarmAck; await TaskCompletionSource | `S7CommPlusClient/MongoCommands.cs` |
| `listS7PlusAlarms` | Return all alarm documents sorted by createdAt desc; limit(200) to be removed | `server_realtime_auth/index.js` line 361 |
| `ackS7PlusAlarm` | Insert one rtCommand per call; single cpuAlarmId + connectionNumber | `server_realtime_auth/index.js` lines 415-450 |
| `deleteS7PlusAlarms` | Delete by ids array or filter object; 204 on success | `server_realtime_auth/index.js` lines 371-411 |
| `S7PlusAlarmsViewerPage.vue` | Poll alarms, filter, display in paged table, ack/delete per row and bulk | `AdminUI/src/components/S7PlusAlarmsViewerPage.vue` |

---

## Key Finding: priority and alarmClass Already Stored

**Critical pre-check result:** `priority` (`(int)dai.HmiInfo.Priority`) and `alarmClass` (`(int)dai.HmiInfo.AlarmClass`) are **already present** in every alarm document. They were added in a prior commit (AlarmThread.cs lines 255-256). The milestone description lists "add alarmPriority" but the field is already `"priority"` in MongoDB.

**What IS missing:**
- `isAcknowledgeable` boolean (derivable from `alarmClass == 33`, never stored)
- `alarmText` and `infoText` placeholder substitution (currently stored raw)

**What only needs a Vue column added (no driver change):**
- Priority sortable column — field `"priority"` already in every document

---

## Integration Point Analysis

### 1. isAcknowledgeable — BuildAlarmDocument() addition

**Insertion point:** `BuildAlarmDocument()` BsonDocument literal, immediately after `"alarmClassName"` (line 257 of AlarmThread.cs).

The `AlarmClassNames` dictionary already maps `33` to `"Acknowledgment required"`. The `AlarmClass` field is `ushort` (confirmed in `AlarmsHmiInfo.cs` line 31). The comparison is straightforward:

```csharp
{ "isAcknowledgeable", dai.HmiInfo.AlarmClass == 33 }
```

No other files change for this field. It flows MongoDB → API → Vue automatically since `listS7PlusAlarms` returns all fields with no projection exclusion.

**Confidence:** HIGH — `AlarmsHmiInfo.AlarmClass` is `ushort`, value 33 confirmed in `AlarmClassNames` dictionary already in the file.

### 2. Remove 200-document limit — listS7PlusAlarms

**Exact change:** `server_realtime_auth/index.js` line 361: delete `.limit(200)`.

The query becomes:
```javascript
.find({}).sort({ createdAt: -1 }).toArray()
```

**Rationale:** With no API limit and the `v-data-table` already providing client-side pagination (`:items-per-page="50"` on line 39), the display volume is managed in the UI. For this single-PLC PoC, total alarm event count is bounded by operator usage patterns.

**Risk:** None for PoC scale. No other file changes required.

### 3. Bulk Ack — sequential calls vs new endpoint

**Recommendation: sequential calls from the frontend to the existing `ackS7PlusAlarm` endpoint. No new endpoint.**

**Rationale:**
- The ack dispatch pipeline is inherently sequential: MongoCommands.cs reads rtCommands and enqueues PendingAlarmAck objects one at a time; AlarmThread.cs dequeues and sends one ack per loop iteration. A server-side bulk endpoint would still process acks serially.
- The existing endpoint inserts one rtCommand per call with full audit fields (`originatorUserName`, `originatorIpAddress`). Bulk insertion in a server-side loop preserves this audit trail with no extra design work.
- The Vue component already has `filteredAlarms` as the visible unacked set. A `for...of` loop with `await` on each `ackAlarm()` call is three lines of code.

**Frontend pattern:**
```javascript
const ackAllFiltered = async () => {
  const unacked = filteredAlarms.value.filter(
    a => !a.ackState && !pendingAcks.value.has(a.cpuAlarmId) && a.isAcknowledgeable
  )
  for (const alarm of unacked) {
    await ackAlarm(alarm.cpuAlarmId, alarm.connectionId)
  }
}
```

The `isAcknowledgeable` guard is important: without it, the button would attempt to ack alarms in classes that do not require acknowledgement (e.g., Logging class 43), which would generate unnecessary rtCommands.

### 4. Pagination vs 5s Auto-Refresh interaction

**Finding:** Pagination is already present. `v-data-table` has `:items-per-page="50"` and `:items-per-page-options="[25, 50, 100, 200]"` (Vue file lines 39-40). The table paginates client-side on the full `filteredAlarms` array.

**The problem:** `fetchAlarms()` replaces `alarms.value` with a fresh array every 5 seconds. Vuetify's `v-data-table` resets to page 1 when the items array reference changes, because it cannot know whether the user intended to stay on the current page.

**Resolution:** Add a `currentPage` ref and bind it to `v-model:page` on the table. Vuetify `v-data-table` supports this two-way binding. The ref persists across the `alarms.value` replacement because it is a separate reactive variable.

```javascript
const currentPage = ref(1)
// In template: <v-data-table ... v-model:page="currentPage">
```

**No API or MongoDB changes needed.**

### 5. Source PLC Filter (connectionName)

**Finding:** `connectionName` is already stored in every alarm document (AlarmThread.cs line 253). The Vue viewer's `connectionId` column slot already renders `item.connectionName || item.connectionId` (Vue file lines 100-101).

**Integration:** Mirror the `alarmClassOptions` computed pattern:

```javascript
const connectionOptions = computed(() => {
  const names = [...new Set(alarms.value.map(a => a.connectionName).filter(Boolean))]
  return ['All', ...names.sort()]
})
```

Add a `connectionFilter` ref, a `v-select` in the toolbar row, and extend `filteredAlarms` with a third condition. No backend or driver changes required.

### 6. Placeholder Substitution — alarmText and infoText

**Finding:** `ResolveAlarmText()` already handles `@N%f@` substitution for all 9 additional texts. The main `alarmText` and `infoText` fields are stored raw without substitution (BuildAlarmDocument lines 245-246).

**Current data flow:**
```
additionalTexts[0-8] -> ResolveAlarmText(template, av) -> substituted, stored
alarmText            -> texts?.AlarmText ?? ""          -> raw, stored
infoText             -> texts?.Infotext  ?? ""          -> raw, stored
```

**Required change in BuildAlarmDocument():**
```csharp
{ "alarmText",  ResolveAlarmText(texts?.AlarmText ?? "", av) },
{ "infoText",   ResolveAlarmText(texts?.Infotext  ?? "", av) },
```

`ResolveAlarmText()` guards `av == null` at line 272, returning the template unchanged — no additional null-check needed. `av` is `dai.AsCgs.AssociatedValues`.

**Caveat:** This is a write-time change. Existing documents in MongoDB retain unresolved placeholders. Only new events after the fix show resolved text. For this PoC this is acceptable.

### 7. Combined Timestamp Column

**No driver or API changes.** The `timestamp` field (`BsonDateTime` from `dai.AsCgs.Timestamp`) is already stored. This is a pure Vue change: replace the separate `date` and `time` virtual columns in the `headers` array with a single `timestamp` column, formatted as `2026-03-24_12:57:10.758`.

---

## Data Flow Changes (v1.2 → v1.3)

### Current flow (v1.2)

```
dai.HmiInfo.Priority   -> BuildAlarmDocument -> "priority"       stored, NOT in viewer
dai.HmiInfo.AlarmClass -> BuildAlarmDocument -> "alarmClass"     stored, NOT in viewer
                                              -> "alarmClassName" stored, in viewer
texts.AlarmText        -> BuildAlarmDocument -> "alarmText"      raw (no substitution)
texts.Infotext         -> BuildAlarmDocument -> "infoText"       raw (no substitution)
additionalTexts        -> ResolveAlarmText   -> substituted      in viewer
listS7PlusAlarms       -> .limit(200)        -> max 200 docs
```

### Target flow (v1.3)

```
dai.HmiInfo.Priority   -> BuildAlarmDocument -> "priority"          stored + NEW viewer column
dai.HmiInfo.AlarmClass -> BuildAlarmDocument -> "alarmClass"        stored
                                              -> "alarmClassName"    existing viewer column
                                              + "isAcknowledgeable" NEW stored + NEW ack-all guard
texts.AlarmText        -> ResolveAlarmText   -> "alarmText"         NOW substituted [CHANGED]
texts.Infotext         -> ResolveAlarmText   -> "infoText"          NOW substituted [CHANGED]
additionalTexts        -> ResolveAlarmText   -> substituted         unchanged
listS7PlusAlarms       -> no limit           -> all docs
v-data-table           -> currentPage ref    -> page preserved across 5s refresh
ackAll button          -> sequential fetch() -> existing ackS7PlusAlarm endpoint N times
connectionName filter  -> computed options   -> new v-select in toolbar
date + time columns    -> single timestamp   -> formatted 2026-03-24_12:57:10.758
```

---

## Component Change Classification

### Modified components (no new files needed)

| Component | Change | Scope |
|-----------|--------|-------|
| `AlarmThread.cs` BuildAlarmDocument() | Add `isAcknowledgeable` field; wrap `alarmText` + `infoText` with `ResolveAlarmText()` | 3 lines |
| `server_realtime_auth/index.js` | Remove `.limit(200)` from listS7PlusAlarms | 1 line deleted |
| `S7PlusAlarmsViewerPage.vue` | Add priority column, combined timestamp column, source PLC filter + computed options, ack-all button, `currentPage` ref + `v-model:page` binding | UI changes only |

### New components

None. All v1.3 features integrate into existing components.

---

## Suggested Build Order

Dependencies drive the ordering:

**Step 1 — Driver: isAcknowledgeable + alarmText/infoText substitution**
- Add `isAcknowledgeable` to `BuildAlarmDocument()`.
- Wrap `alarmText` and `infoText` with `ResolveAlarmText()`.
- No downstream dependencies; verifiable immediately via new alarm events in MongoDB.
- Deliverable: new alarm events carry `isAcknowledgeable: true/false`, and alarm/info text has substituted values.

**Step 2 — API: remove 200-doc limit**
- Delete `.limit(200)` from `listS7PlusAlarms`.
- Prerequisite for pagination to show more than 200 rows. Single-line change, low risk.
- Deliverable: all alarm history is accessible from the UI.

**Step 3 — Vue: priority column + combined timestamp**
- Replace `date` and `time` headers with a single `timestamp` column.
- Add `priority` to headers.
- Pure display changes; validates that new driver fields flow through correctly.
- Deliverable: viewer shows priority and compact timestamp.

**Step 4 — Vue: source PLC filter**
- Add `connectionFilter` ref, `connectionOptions` computed, new v-select.
- Extend `filteredAlarms` condition.
- Independent of ack and pagination changes.

**Step 5 — Vue: pagination page-preservation**
- Add `currentPage` ref, bind `v-model:page`.
- Depends on pagination already existing (it does).
- Low-risk isolated change.

**Step 6 — Vue: Ack All button**
- Add button above table, filtered to unacked + `isAcknowledgeable == true`.
- Depends on `isAcknowledgeable` field being available in data (Step 1).
- Calls existing `ackAlarm()` in a sequential loop.

---

## Anti-Patterns

### Anti-Pattern 1: New bulk ack endpoint

**What people do:** Add `POST /ackS7PlusAlarms` accepting an array of cpuAlarmIds, inserting N rtCommands in a server-side loop.

**Why it's wrong:** The PLC ack dispatch pipeline (PendingAcks queue → AlarmThread dequeue → SendAlarmAck) is sequential by design. A server-side loop adds backend complexity without changing throughput. It also complicates the per-command audit trail.

**Do this instead:** Sequential `await ackAlarm()` calls from the Vue component. Three lines of code; no new endpoint.

### Anti-Pattern 2: Server-side pagination (skip/limit API params)

**What people do:** Add `?page=N&pageSize=50` query params to `listS7PlusAlarms`, use `.skip(N*50).limit(50)`.

**Why it's wrong:** The 5s auto-refresh is a full collection reload for live operator monitoring. Server-side pagination breaks this: inserting a new alarm shifts all subsequent page offsets, causing alarms to jump between pages mid-session.

**Do this instead:** Remove the 200-doc limit, return all docs, let `v-data-table` paginate client-side with `v-model:page` preserving the user's position.

### Anti-Pattern 3: Resolving placeholders in the API layer (Node.js)

**What people do:** Store raw alarm text in MongoDB, then substitute @N%f@ placeholders in the Node.js API when serving data.

**Why it's wrong:** `associatedValues` are stored as typed BsonArray. Reconstructing substitution logic in JavaScript duplicates the C# `ResolveAlarmText()` logic and introduces format discrepancies (particularly REAL/LREAL decimal formatting using InvariantCulture, which differs from JavaScript's default).

**Do this instead:** Resolve at write time in `BuildAlarmDocument()`. The AssociatedValues are already in scope.

### Anti-Pattern 4: Renaming "priority" to "alarmPriority" in the schema

**What people do:** Add a new `alarmPriority` field alongside the existing `priority` field, because the milestone description used that name.

**Why it's wrong:** `priority` is already stored in every existing alarm document. Adding a duplicate field with a different name creates schema inconsistency between historical and new documents.

**Do this instead:** Use the existing `"priority"` field. Just add the Vue column that references it.

---

## Integration Points Summary

| Boundary | Communication | Notes |
|----------|---------------|-------|
| AlarmThread -> MongoDB | InsertOneAsync (sync-over-async) | New fields added here; existing docs unchanged |
| MongoCommands -> AlarmThread | ConcurrentQueue<PendingAlarmAck> | Unchanged for v1.3 |
| Node.js API -> MongoDB | Native driver, no Mongoose | Remove .limit(200) only |
| Vue -> Node.js API | fetch(), 5s poll + per-alarm ack calls | Sequential ack loop for bulk ack |
| Vue -> v-data-table | items array + v-model:page | Add page ref to preserve position across refresh |

---

## Sources

- Direct code inspection: `AlarmThread.cs` lines 202-293 — BuildAlarmDocument, ResolveAlarmText, AlarmClassNames dictionary
- Direct code inspection: `AlarmsHmiInfo.cs` — confirmed `Priority: byte`, `AlarmClass: ushort`
- Direct code inspection: `server_realtime_auth/index.js` lines 350-450 — all three alarm endpoints; confirmed `.limit(200)` at line 361
- Direct code inspection: `S7PlusAlarmsViewerPage.vue` — existing filters, ack/delete logic, pagination props
- Direct code inspection: `Common.cs` — PendingAcks ConcurrentQueue, RelationIdNameMap on S7CP_connection
- Direct code inspection: `MongoCommands.cs` — s7plus-alarm-ack dispatch path confirmed sequential
- `.planning/PROJECT.md` — v1.2 current state, v1.3 target features, key decisions

---

*Architecture research for: S7CommPlus alarm viewer enhancements (v1.3)*
*Researched: 2026-03-25*

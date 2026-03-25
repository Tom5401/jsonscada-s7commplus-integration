# Stack Research

**Domain:** S7CommPlus alarm viewer enhancements — additions to existing v1.2 driver (v1.3 milestone)
**Researched:** 2026-03-25
**Confidence:** HIGH — all findings verified directly from submodule source code and existing project files

---

## Summary of New Stack Needs

v1.3 adds enrichment, UX improvements, and bulk ack to an already-validated stack. **No new npm packages or NuGet packages are required.** Every API, field, and component needed already exists. The sections below cover only what is new or changed for v1.3.

---

## 1. S7CommPlus Protocol — Priority and isAcknowledgeable

### Both fields are already captured and stored in MongoDB

**Source:** `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHmiInfo.cs` and `AlarmThread.cs` (BuildAlarmDocument)

The alarm notification PDU carries a `HmiInfo` blob (attribute `DAI_HmiInfo`, ID 7813). When `SyntaxId >= 258`, this blob contains:

| Byte offset | Field | C# type | Meaning |
|-------------|-------|---------|---------|
| 0–1 | SyntaxId | ushort | Protocol version indicator |
| 2–3 | Version | ushort | Alarm version |
| 4–7 | ClientAlarmId | uint | Client-side alarm ID |
| 8 | Priority | byte | Alarm priority (0–n, PLC-configured) |
| 9 | Reserved1 | byte | — |
| 10 | Reserved2 | byte | — |
| 11 | Reserved3 | byte | — |
| 12–13 | AlarmClass | ushort | TIA Portal alarm class ID |
| 14 | Producer | byte | — |
| 15 | GroupId | byte | Alarm group |
| 16 | Flags | byte | Additional flags |

The existing `BuildAlarmDocument` in `AlarmThread.cs` already stores these:

```csharp
{ "priority",       (int)dai.HmiInfo.Priority },
{ "alarmClass",     (int)dai.HmiInfo.AlarmClass },
{ "alarmClassName", AlarmClassNames.TryGetValue(dai.HmiInfo.AlarmClass, ...) },
```

**Both `priority` and `alarmClass` are already in every MongoDB document.** No changes to the C# driver are needed to support v1.3 priority or acknowledgeable features.

### isAcknowledgeable derivation

`isAcknowledgeable` is not a separate protocol field. It is implied by `AlarmClass`:

- `AlarmClass == 33` → acknowledgement required (TIA Portal "Acknowledgment required" class)
- All other classes → no acknowledgement required

This derivation lives in the Vue component. Deriving it in Vue avoids a MongoDB schema change, and AlarmClass 33 is project-validated (see `AlarmClassNames` dict in `AlarmThread.cs`).

**Implementation pattern:**

```js
// In Vue computed or template
const isAcknowledgeable = (alarm) => alarm.alarmClass === 33
```

**Confidence:** HIGH — `AlarmsHmiInfo.cs` source code, `AlarmThread.cs` source code, `PROJECT.md` explicit statement "class 33 = true".

---

## 2. Vuetify v-data-table — Sortable Columns and Client-Side Pagination

### Already working in the existing component

**Installed version:** Vuetify 3.10.12 (confirmed from `node_modules/vuetify/package.json`)

The existing `S7PlusAlarmsViewerPage.vue` already uses `v-data-table` with:

```vue
<v-data-table
  :headers="headers"
  :items="filteredAlarms"
  density="compact"
  :items-per-page="50"
  :items-per-page-options="[25, 50, 100, 200]"
>
```

And header definitions already include `sortable: true`:

```js
const headers = [
  { title: 'Source',         key: 'connectionId',   sortable: true },
  { title: 'Status',         key: 'alarmState',     sortable: true },
  { title: 'Acknowledge',    key: 'ackState',        sortable: true },
  { title: 'Alarm class name', key: 'alarmClassName', sortable: true },
  // ... etc
]
```

**Vuetify 3 `v-data-table` built-in behaviour (v3.10.12):**

| Feature | How it works | Props/Config |
|---------|-------------|--------------|
| Column sorting | Click header → sorts ascending/descending/none | `sortable: true` on header definition |
| Client-side pagination | Paginator rendered automatically below table | `:items-per-page`, `:items-per-page-options` |
| Pagination with all data in memory | Works with any array size; Vuetify handles slicing | No server-side call needed |
| Numeric sort | Sorts numbers as numbers, not strings | Default for Number fields |
| ISO date string sort | Lexicographic — correct for ISO 8601 | Default for string fields |

**For the new combined timestamp column** (replacing separate date + time):

```js
{ title: 'Timestamp', key: 'timestamp', sortable: true }
```

ISO 8601 strings (`"2026-03-24T12:57:10.758Z"`) sort correctly lexicographically. No custom sort comparator needed.

**For the priority column:**

```js
{ title: 'Priority', key: 'priority', sortable: true }
```

`priority` is a number in MongoDB → number in JSON → Vuetify sorts numerically by default. No custom comparator needed.

**No new Vuetify components or plugins required.** `v-data-table` is already registered via `vite-plugin-vuetify` auto-import.

**Confidence:** HIGH — installed version confirmed from `node_modules`, existing component source read.

---

## 3. MongoDB — Removing the 200-Alarm Cap

### Current implementation

`server_realtime_auth/index.js` (listS7PlusAlarms endpoint):

```js
const docs = await db
  .collection('s7plusAlarmEvents')
  .find({})
  .sort({ createdAt: -1 })
  .limit(200)        // <-- remove this line
  .toArray()
```

### Change required

Remove `.limit(200)`. No other changes. The `.find({}).sort({ createdAt: -1 })` pattern is already correct.

**Performance consideration:** At PoC scale (hundreds to low thousands of alarms), fetching all documents is acceptable. The entire collection is loaded into Vue's `alarms` ref on each 5-second poll; Vuetify pages the display client-side. No server-side pagination API is needed.

**MongoDB driver version:** `mongodb ^7.0.0` (Node.js, `server_realtime_auth/package.json`). `.find().sort().toArray()` without `.limit()` is stable across all versions.

**Confidence:** HIGH — source code read, `package.json` confirmed.

---

## 4. Bulk Ack — Sending Multiple Ack Commands

### How the single-ack path works

The existing `POST /Invoke/auth/ackS7PlusAlarm` endpoint inserts one document into `commandsQueue` with `protocolSourceASDU: 's7plus-alarm-ack'`. The `ProcessMongoCmd` change stream listener in `MongoCommands.cs` picks it up, enqueues a `PendingAlarmAck` on `srv.PendingAcks`, and `AlarmThread` drains the queue sending each ack via `alarmConn.SendAlarmAck()`. Acks are sent sequentially, one at a time — this is the correct approach because S7CommPlus ack is a request-response PDU pair per alarm.

### Recommended approach for "Ack All" button

**Client-side sequential fetch loop — no new backend endpoint needed:**

```js
const ackAllPending = ref(false)

const ackAll = async () => {
  const unacked = filteredAlarms.value.filter(
    a => !a.ackState && isAcknowledgeable(a)
  )
  if (unacked.length === 0) return
  ackAllPending.value = true
  try {
    for (const alarm of unacked) {
      await ackAlarm(alarm.cpuAlarmId, alarm.connectionId)
    }
  } finally {
    ackAllPending.value = false
  }
}
```

**Why this is correct:**
- Each ack goes through the same `commandsQueue → MongoCommands.cs → AlarmThread → SendAlarmAck` path that single ack uses — no new code paths in C#.
- There is no batch-ack PDU in S7CommPlus; the PLC processes them one at a time regardless.
- Sequential `await` means each ack waits for the previous one to complete — avoids flooding `PendingAcks` queue.
- Consistent with PoC simplicity constraint — no new backend endpoint, no new C# changes.

**Why not a new bulk-ack backend endpoint:** An `insertMany` into commandsQueue would fire N change stream events simultaneously. `ProcessMongoCmd` picks them up concurrently via `ForEachAsync` but `SendAlarmAck` is synchronous on `alarmConn`. Concurrent enqueues would pile up on `PendingAcks`; the loop in `AlarmThread` already handles that, but the end result is the same serial execution. Sequential client-side `for` loop gives identical throughput with less code.

**Confidence:** HIGH — `MongoCommands.cs` and `AlarmThread.cs` source code read, architecture confirmed.

---

## 5. Source PLC Filter (connectionName)

The `connectionName` field is already stored in every alarm document:

```csharp
{ "connectionName", srv.name ?? "" },
```

And the viewer already displays it (lines 99–101 of S7PlusAlarmsViewerPage.vue):

```vue
<template #[`item.connectionId`]="{ item }">
  {{ item.connectionName || item.connectionId }}
</template>
```

A source filter works exactly like the existing `alarmClassFilter`: a `v-select` over the distinct `connectionName` values computed from `alarms.value`. No backend changes needed.

**Confidence:** HIGH — source code confirmed field exists in all documents.

---

## 6. Placeholder Substitution in alarmText and infoText

The `ResolveAlarmText` static method already exists in `AlarmThread.cs` and handles `@N%x@` placeholders using `AlarmsAssociatedValues`. It is already applied to `additionalTexts` in `BuildAlarmDocument`. To extend to `alarmText` and `infoText`, apply the same method to those two fields:

```csharp
{ "alarmText", ResolveAlarmText(texts?.AlarmText ?? "", av) },
{ "infoText",  ResolveAlarmText(texts?.Infotext ?? "", av) },
```

These are the only lines that need to change — no new protocol parsing or library.

**Confidence:** HIGH — `ResolveAlarmText` signature and implementation confirmed in source.

---

## Core Technologies (unchanged from v1.2)

| Technology | Version | Purpose | Role in v1.3 |
|------------|---------|---------|--------------|
| S7CommPlusDriver (submodule) | pinned | PLC protocol | No changes — HmiInfo already decoded |
| .NET 8 / C# | 8.0 | Driver host | Minor: apply ResolveAlarmText to alarmText + infoText |
| MongoDB.Driver (C#) | 3.4.2 | Alarm writes | No changes — priority/alarmClass already written |
| mongodb (Node.js) | ^7.0.0 | Query endpoint | Remove `.limit(200)` only |
| Vue 3 | ^3.4.31 | UI framework | Ack All loop, timestamp column, source filter, priority column |
| Vuetify 3 | 3.10.12 (installed) | Component library | v-data-table sortable already works; no new components |

## Supporting Libraries — No New Additions

All required capabilities are present in already-installed versions.

| Need | Library | Capability | Status |
|------|---------|-----------|--------|
| Sortable columns | Vuetify 3 v-data-table | `sortable: true` on header | Already present in headers |
| Client-side pagination | Vuetify 3 v-data-table | `:items-per-page` prop | Already configured |
| Priority numeric sort | Vuetify 3 v-data-table | Default numeric sort | Works automatically for number fields |
| Timestamp ISO sort | Vuetify 3 v-data-table | Lexicographic string sort | ISO 8601 sorts correctly |
| Bulk ack loop | Vue 3 | `async/await for...of` | Native JS, no library |
| Source PLC filter | Vue 3 computed | Same pattern as alarmClassFilter | Already in component |
| isAcknowledgeable flag | Vue 3 computed | `alarm.alarmClass === 33` | Derived from existing field |

---

## What NOT to Add

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| PrimeVue | Not in the project — the UI uses Vuetify 3 | Continue with Vuetify `v-data-table` |
| Server-side pagination API | PoC scale does not require it; adds endpoint complexity | Client-side via Vuetify `:items-per-page` |
| New bulk-ack endpoint (`ackS7PlusAlarms`) | Sequential individual acks through existing path is architecturally correct; no batch PDU in S7CommPlus | Client-side `for` loop calling existing endpoint |
| `isAcknowledgeable` as a stored MongoDB field | Derivable from `alarmClass == 33` with zero overhead; adding it requires driver change and all existing documents would be missing the field | Compute in Vue component |
| Custom sort function for `priority` | Vuetify sorts numbers natively | Keep header as `{ key: 'priority', sortable: true }` |
| Separate non-sortable date + time columns | Cannot be sorted; UX downgrade | Single `timestamp` column with ISO string, sortable |
| Removing `connectionName` in favour of `connectionId` | `connectionName` is already stored and is human-readable | Display `connectionName` as source filter value |

---

## Version Compatibility

| Component | Version | Notes |
|-----------|---------|-------|
| Vuetify | 3.10.12 installed | `v-data-table` with `sortable`, `:items-per-page`, `:items-per-page-options` — stable throughout 3.x |
| mongodb (Node.js) | ^7.0.0 | `.find().sort().toArray()` without limit — unchanged, stable API |
| MongoDB.Driver (C#) | 3.4.2 | No new calls needed for v1.3 |
| Vue 3 | ^3.4.31 | `async/await`, `computed`, `ref`, `for...of` — all standard; no new APIs |

---

## Sources

- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHmiInfo.cs` — `Priority` (byte, byte-offset 8) and `AlarmClass` (ushort, byte-offset 12–13) confirmed in HmiInfo blob deserialization (HIGH confidence, source code)
- `json-scada/src/S7CommPlusClient/AlarmThread.cs` lines 255–257 — `priority` and `alarmClass` already stored in every alarm document via `dai.HmiInfo.Priority` and `dai.HmiInfo.AlarmClass` (HIGH confidence, source code)
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` lines 34–41, 179–194 — `v-data-table` already configured with `sortable: true`, `:items-per-page`, and pagination options (HIGH confidence, source code)
- `json-scada/src/AdminUI/node_modules/vuetify/package.json` — Vuetify 3.10.12 confirmed as installed version (HIGH confidence, package file)
- `json-scada/src/AdminUI/package.json` — Vue ^3.4.31, Vuetify 3.10 declared dependencies (HIGH confidence, project file)
- `json-scada/src/server_realtime_auth/index.js` lines 352–368 — `.limit(200)` in listS7PlusAlarms confirmed; removal is the only change needed (HIGH confidence, source code)
- `json-scada/src/server_realtime_auth/index.js` lines 414–444 — existing single-ack endpoint inserting to `commandsQueue` (HIGH confidence, source code)
- `json-scada/src/S7CommPlusClient/MongoCommands.cs` lines 97–132 — `PendingAcks` enqueue and sequential `SendAlarmAck` confirmed; sequential client-side loop is architecturally correct (HIGH confidence, source code)
- `json-scada/src/S7CommPlusClient/AlarmThread.cs` line 253 — `connectionName` stored in every alarm document (HIGH confidence, source code)
- `.planning/PROJECT.md` — "AlarmClass 33 = isAcknowledgeable = true" stated explicitly (HIGH confidence, project document)

---

*Stack research for: S7CommPlus Alarm Viewer Enhancements & Priority (v1.3)*
*Researched: 2026-03-25*

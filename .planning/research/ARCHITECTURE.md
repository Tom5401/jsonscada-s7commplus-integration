# Architecture Research

**Domain:** S7CommPlus alarm ack write-back, alarm class name resolution, Vue 3 SCADA alarm viewer
**Researched:** 2026-03-18
**Confidence:** HIGH for C# integration points and Vue routing patterns; MEDIUM for ackState bit correction; LOW for alarm ack PDU structure (requires Wireshark spike)

---

## System Overview

```
PLC (S7-1515)
  │  S7CommPlus protocol
  ▼
S7CommPlusClient (C#/.NET)
  ├── ConnectionThread        — tag read/write cycle (unchanged)
  ├── AlarmThread             — alarm subscription receive loop (v1.0)
  │     ├── ackState bug fix  — MODIFIED in v1.1
  │     ├── alarmClassName    — NEW in v1.1 (lookup on write)
  │     └── ack write-back    — NEW in v1.1 (separate ack connection)
  └── MongoCommands.cs        — commandsQueue consumer
        └── AlarmAckToPLC()   — NEW in v1.1

MongoDB
  ├── s7plusAlarmEvents        — alarm events (v1.0 + alarmClassName in v1.1)
  └── commandsQueue            — extended with "ALARM_ACK" ASDU in v1.1

server_realtime_auth (Node.js/Express)
  ├── GET /api/s7plus-alarms   — NEW in v1.1
  └── POST /api/s7plus-alarms/ack — NEW in v1.1

AdminUI (Vue 3 + Vuetify 3)
  ├── AlarmsViewerPage.vue     — UNCHANGED (tag-based viewer)
  └── S7PlusAlarmsViewerPage.vue — NEW in v1.1
```

---

## Integration Points

### 1. ackState Bug Fix — `AlarmThread.cs`

**Location:** `S7CommPlusClient/AlarmThread.cs` line ~191

**Current (buggy):**
```csharp
ackState = dai.AsCgs.AckTimestamp != DateTime.MinValue
```

**Problem:** `AckTimestamp` is not `DateTime.MinValue` for unacknowledged alarms — the field has another sentinel or the wrong field is being read.

**Fix approach:** Check `AllStatesInfo` bits from `dai.AsCgs.AllStatesInfo` for the ack flag, OR verify the correct AckTimestamp sentinel via live PLC trace. The `AllStatesInfo` byte is already parsed and available.

**Risk:** MEDIUM — exact bit mask must be validated against a live PLC with a known-unacknowledged alarm before merging.

---

### 2. Alarm Class Name Resolution — `AlarmThread.cs` + `Common.cs`

**New field:** `alarmClassName` (string) added to MongoDB document alongside existing `alarmClass` (int)

**Implementation:** Static `Dictionary<ushort, string>` in `Common.cs`:
```csharp
private static readonly Dictionary<ushort, string> AlarmClassNames = new()
{
    { 1,   "Systemdiagnose" },
    { 2,   "Security" },
    { 256, "UserClass_0" },
    { 257, "UserClass_1" },
    // ... 256–272 = UserClass_0..UserClass_16
};
```

**Usage in AlarmThread.cs:**
```csharp
alarmClassName = AlarmClassNames.TryGetValue((ushort)alarmClass, out var name) ? name : $"Class_{alarmClass}"
```

**Note:** User-defined TIA Portal alarm class display names (e.g. "8_Logging", "4_UrgentOnderhoud") are NOT transmitted in S7CommPlus notifications. The names visible in TIA Portal's alarm viewer are configured in TIA Portal and not available via the protocol. The static map covers protocol-fixed IDs only; project-specific display names require manual configuration.

---

### 3. Ack Write-Back — New Data Flow

```
Vue S7PlusAlarmsViewerPage
  → POST /api/s7plus-alarms/ack { cpuAlarmId }
    → server_realtime_auth/index.js (new route)
      → commandsQueue.insertOne({
          protocolSourceASDU: "ALARM_ACK",
          protocolSourceObjectAddress: cpuAlarmId,
          ...
        })
        → MongoCommands.cs ProcessMongoCmd() change stream
          → new branch: case "ALARM_ACK":
              → AlarmAckToPLC(cpuAlarmId)
                → SetVariableRequest on alarmConn (separate from tag conn)
                  → PLC acknowledges alarm
                    → PLC sends ack-only notification
                      → AlarmThread receives ack-only notification
                        → UpdateOne on s7plusAlarmEvents: set ackState=true
```

**Key decisions:**
- Ack PDU sent on `alarmConn` (not `srv.connection`) to avoid interleaving with tag PDUs
- `commandsQueue` extended — no new collection needed
- `protocolSourceASDU = "ALARM_ACK"` — new sentinel value, distinct from tag write ASDU types
- `AlarmAckToPLC` is a new method — exact `SetVariableRequest` payload requires Wireshark trace

**BLOCKED:** Alarm ack PDU structure is not in `Ids.cs` or any existing driver code. A Wireshark capture of TIA Portal performing an ack is required before this can be implemented.

---

### 4. New REST Endpoints — `server_realtime_auth/index.js`

Two new Express routes, added alongside existing routes with same auth middleware:

**GET /api/s7plus-alarms**
- Query params: `limit`, `offset`, `connectionName`, `ackState`
- Returns: array of documents from `s7plusAlarmEvents`, sorted by `plcTimestamp` desc
- Auth: same `authJwt` middleware as existing protected routes

**POST /api/s7plus-alarms/ack**
- Body: `{ cpuAlarmId: string }`
- Inserts into `commandsQueue` with `protocolSourceASDU: "ALARM_ACK"`
- Returns: `{ queued: true }`
- Auth: same `authJwt` middleware

---

### 5. New Vue Component — `S7PlusAlarmsViewerPage.vue`

**Location:** `AdminUI/src/components/S7PlusAlarmsViewerPage.vue`

**Pattern:** Native Vue component (NOT an iframe — unlike existing `AlarmsViewerPage.vue` which is a 5-line iframe wrapper). Must be native Vue for the ack button to work.

**New route:** `/s7plus-alarms` in `router/index.js` — distinct from existing `/alarms-viewer`

**Data fetching:** `fetch('/api/s7plus-alarms')` + `setInterval(5000)` — pause polling during user interaction

**Key UI elements (from TIA Portal screenshot):**
- `v-data-table` with columns: Source, Date, Time, Status, Acknowledge, Alarm class name, Event text, ID, Additional text 1–3
- "Acknowledge" button per row — enabled only when `ackState: false` for that alarm
- "Receive alarms" dropdown (filter by `connectionName`)
- Auto-refresh toggle

**Component isolation:** Zero shared code with `AlarmsViewerPage.vue`. Independent state, independent API calls, independent route.

---

## New vs. Modified Components

| Component | Status | Change |
|-----------|--------|--------|
| `AlarmThread.cs` | MODIFIED | Fix ackState field; add alarmClassName lookup on document build |
| `Common.cs` | MODIFIED | Add `AlarmClassNames` static dictionary |
| `MongoCommands.cs` | MODIFIED | Add `"ALARM_ACK"` branch + `AlarmAckToPLC()` method |
| `server_realtime_auth/index.js` | MODIFIED | Add 2 new Express routes |
| `S7PlusAlarmsViewerPage.vue` | NEW | S7Plus alarm viewer page |
| `AdminUI/src/router/index.js` | MODIFIED | Add `/s7plus-alarms` route |
| `AlarmsViewerPage.vue` | UNCHANGED | Tag-based viewer — must not be touched |
| `s7plusAlarmEvents` collection | MODIFIED | New `alarmClassName` field; `ackState` bug fix |
| `commandsQueue` collection | MODIFIED | New `"ALARM_ACK"` ASDU type used |

---

## Suggested Build Order

1. **Fix ackState + add alarmClassName** (`AlarmThread.cs`, `Common.cs`) — isolated C# changes, no frontend dependency, fix data quality before building UI
2. **REST endpoints** (`server_realtime_auth/index.js`) — can be done in parallel with step 1; no C# compile coupling
3. **Vue alarm viewer page** (`S7PlusAlarmsViewerPage.vue`, router) — depends on REST endpoints being available
4. **Ack write-back** (`MongoCommands.cs`, `AlarmThread.cs` ack notification handler) — **BLOCKED** on Wireshark spike; implement after spike confirms PDU format
5. **Ack button in viewer** — depends on ack write-back API being functional

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why | Correct Approach |
|--------------|-----|------------------|
| Sending ack PDU on `srv.connection` (tag connection) | PDU interleaving with tag read/write cycle | Use dedicated `alarmConn` for ack PDU |
| Modifying `AlarmsViewerPage.vue` | Breaks existing tag-based alarm viewer | New `S7PlusAlarmsViewerPage.vue` only |
| Adding `/alarms-viewer` as the new route | Route collision — silently replaces existing viewer | Use `/s7plus-alarms` |
| Using `InsertOne` for ack confirmation update | Creates duplicate alarm document | `UpdateOne` on `cpuAlarmId` filter |
| Implementing ack write-back without Wireshark trace | Unknown PDU format — risk of crashing alarm thread | Spike first, then implement |
| Calling `ExploreASAlarms` per alarm event | 3 PDU round-trips per event | Static dictionary lookup only |

---

## Open Question — Requires Spike

**Alarm ack PDU structure** (confidence: LOW):
- What `SetVariableRequest` payload (object ID, attribute ID, value type) sends an ack command over S7CommPlus?
- This is not documented in `Ids.cs` or any existing driver code
- **Resolution:** Wireshark capture of TIA Portal HMI performing an ack on a live PLC, then decode the PDU fields
- **Gate:** Ack write-back phase cannot start until this spike is complete

---

## Sources

- Direct code inspection: `AlarmThread.cs` — ackState computation, alarm document build
- Direct code inspection: `AlarmsMultipleStai.cs` — AlarmDomain/AlarmClass numeric IDs
- Direct code inspection: `MongoCommands.cs` — commandsQueue consumer, ASDU routing pattern
- Direct code inspection: `server_realtime_auth/index.js` — Express route and auth middleware pattern
- Direct code inspection: `AdminUI/src/router/index.js` — existing route structure
- Direct code inspection: `AdminUI/src/components/AlarmsViewerPage.vue` — iframe pattern (5-line component)
- TIA Portal alarm viewer screenshot (user-provided) — column and UX reference

---

*Architecture research for: S7CommPlus alarm ack write-back, alarm class resolution, Vue 3 alarm viewer*
*Researched: 2026-03-18*

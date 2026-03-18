# Stack Research

**Domain:** S7CommPlus alarm ack write-back, alarm class name resolution, Vue 3 SCADA alarm viewer
**Researched:** 2026-03-18
**Confidence:** HIGH (all findings grounded in direct codebase inspection)

---

## Recommended Stack

### C# Driver — Ack Write-Back & Alarm Class Resolution

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| .NET / C# | net8.0 (LTS) | Runtime and language | Already used; no change |
| S7CommPlusDriver | Project reference (local) | S7CommPlus protocol — alarm ack PDU send | `SetVariableRequest` infrastructure already present; ack PDU reuses same mechanism. Wireshark trace required for exact attribute IDs (not in `Ids.cs`). |
| MongoDB.Driver | 3.4.2 (already pinned) | `UpdateOne` for ack confirmation writes | Already in use; `UpdateOne` on `s7plusAlarmEvents` with filter on `cpuAlarmId`. No version bump needed. |
| System.Collections.Generic (BCL) | Built into .NET 8 | `Dictionary<ushort, string>` for alarm class name map | Alarm class names are NOT transmitted over the wire; mapped from protocol-fixed numeric IDs (1=Systemdiagnose, 2=Security, 256–272=UserClass_0..UserClass_16). Static dictionary in `Common.cs`. No NuGet package needed. |

### Backend — Ack API

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| server_realtime_auth (Node.js/Express) | Existing | New REST endpoints for alarm data and ack commands | Already used by AdminUI; two new routes: `GET /s7plusAlarms` (query MongoDB) and `POST /s7plusAlarmAck` (insert into `commandsQueue`). No new npm packages. |
| commandsQueue (MongoDB collection) | Existing | Route ack command from frontend to C# driver | Extend with new sentinel `protocolSourceASDU = "alarm_ack"`. New `AlarmAckToPLC` method in `MongoCommands.cs` handles this ASDU type. |

### Frontend — Alarms Viewer

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Vue 3 | ^3.4.31 (already installed) | Component framework | Already in AdminUI |
| Vuetify 3 | 3.10 (already installed) | `v-data-table`, layout components | Already in AdminUI; matches existing AlarmsViewerPage.vue style |
| vue-router | ^4.4.0 (already installed) | New `/s7plus-alarms` route | Already in AdminUI |
| fetch() (browser built-in) | N/A | Poll `GET /s7plusAlarms` every 5s | No new npm package; setInterval(5000) pattern is sufficient for alarm viewer |

**No new NuGet packages. No new npm packages.**

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| Pinia / Vuex | State management overkill for a single alarm viewer page | Local component state + fetch polling |
| GraphQL subscriptions / WebSockets | Infrastructure not needed for 5s polling alarm viewer | fetch() + setInterval |
| New MongoDB alarm ack queue | Unnecessary — `commandsQueue` already routes commands to C# driver | Extend `commandsQueue` with `"alarm_ack"` ASDU type |
| Extending existing `AlarmsViewerPage.vue` | Must not modify existing tag-based viewer | New `S7PlusAlarmsViewerPage.vue` — fully independent component |
| MongoDB.Driver version bump | No benefit for PoC; v3.x has breaking changes between minors | Pin at 3.4.2 |

---

## Key Implementation Notes

### ackState Bug Fix
- **Location:** `AlarmThread.cs` line ~191 — `ackState = dai.AsCgs.AckTimestamp != DateTime.MinValue`
- **Risk:** Encoding for "unacknowledged" state on live PLC may differ from `DateTime.MinValue`
- **Required:** Live Wireshark trace or PLC test with known-unacknowledged alarm to verify correct sentinel value before fixing

### Alarm Class Name Resolution
- **Source:** `AlarmsMultipleStai.cs` — `AlarmDomain` encodes class as protocol-fixed numeric ID
- **Mapping:** Static `Dictionary<ushort, string>` — IDs 1, 2, 256–272 have known names; user-defined TIA Portal display names are NOT available from the protocol
- **Storage:** Add `alarmClassName` string field to MongoDB alarm document alongside existing `alarmClass` (int)

### Ack Write-Back — Open Question
- Exact `SetVariableRequest` payload (object ID, attribute ID, value) for alarm ack is NOT documented in `Ids.cs`
- **Required before implementation:** Wireshark capture of TIA Portal HMI performing an ack operation

### Alarm Viewer Data Flow
```
Vue S7PlusAlarmsViewerPage
  → GET /s7plusAlarms (server_realtime_auth)
    → MongoDB s7plusAlarmEvents.find()
  → POST /s7plusAlarmAck (server_realtime_auth)
    → commandsQueue.insertOne({ protocolSourceASDU: "alarm_ack", cpuAlarmId: ... })
      → MongoCommands.cs AlarmAckToPLC()
        → SetVariableRequest to PLC
```

---

## Open Questions for Phase-Specific Research

1. **Alarm ack wire format** — What `SetVariableRequest` payload acknowledges an alarm over S7CommPlus? Wireshark decode of TIA Portal HMI performing ack required before implementing `AlarmAckToPLC`.

2. **AckTimestamp encoding for unacknowledged alarms** — Is the sentinel `DateTime.MinValue`, zero timestamp, or something else? Requires live capture with known-unacknowledged alarm.

3. **server_realtime_auth route registration** — Inline or `app/routes/` file structure? Does `authJwt` middleware apply? Check `app/routes/auth.routes.js`.

---

## Sources

- Codebase: `S7CommPlusClient.csproj` — MongoDB.Driver 3.4.2, net8.0, x64 confirmed
- Codebase: `AlarmThread.cs` — ackState computation at line ~191 (`AckTimestamp != DateTime.MinValue`)
- Codebase: `AlarmsMultipleStai.cs` — AlarmDomain numeric encoding (1, 2, 256–272)
- Codebase: `MongoCommands.cs` — commandsQueue consumer, protocolSourceASDU routing
- Codebase: `AdminUI/package.json` — Vue 3.4.31, Vuetify 3.10, vue-router 4.4.0 confirmed
- Codebase: `AdminUI/src/components/AlarmsViewerPage.vue` — existing viewer reference (not to be modified)
- Codebase: `json-scada/src/server_realtime_auth/` — Express route pattern for new endpoints

---

*Stack research for: S7CommPlus alarm ack write-back, alarm class resolution, Vue 3 alarm viewer*
*Researched: 2026-03-18*

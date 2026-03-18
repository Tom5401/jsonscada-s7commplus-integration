# Feature Research

**Domain:** S7CommPlus alarm ack write-back, alarm class name resolution, Vue 3 SCADA alarm viewer
**Researched:** 2026-03-18
**Confidence:** HIGH (derived directly from protocol source code, v1.0 codebase, and TIA Portal alarm viewer reference)

---

## Existing v1.0 Data (Already in MongoDB `s7plusAlarmEvents`)

These fields are already stored in v1.0. They are the raw material for the viewer:

| Field | Type | Notes |
|-------|------|-------|
| `cpuAlarmId` | string | Unique alarm identity |
| `alarmState` | string | "coming" / "going" |
| `plcTimestamp` | DateTime | PLC-side event time |
| `serverTimestamp` | DateTime | Driver-receive time |
| `alarmText` | string | Primary alarm message |
| `additionalTexts` | string[] | SD_1–SD_9 resolved texts |
| `ackState` | bool | **BUG: always true** — fix is v1.1 scope |
| `ackTimestamp` | DateTime | PLC-side ack time |
| `alarmClass` | int | Numeric class ID — name resolution is v1.1 scope |
| `alarmDomain` | int | 1=System, 2=Security, 256–272=UserClass_0..16 |
| `priority` | byte | TIA Portal alarm priority |
| `associatedValues` | object[] | SD_1–SD_10 typed process values |
| `connectionName` | string | PLC connection identifier |

---

## Feature Landscape — v1.1

### Table Stakes (Must Have)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Fix `ackState` always-true bug | Core data correctness — ackState is meaningless if always true; blocks all ack-related features | LOW-MEDIUM | Requires live trace to verify correct sentinel for "unacknowledged" state |
| Alarm class name field (`alarmClassName`) | Operators cannot interpret a raw number; TIA Portal shows names like "8_Logging", "4_UrgentOnderhoud" | LOW | Static `Dictionary<ushort, string>` for protocol-fixed IDs; add `alarmClassName` string field to MongoDB document |
| New S7Plus Alarms Viewer page in AdminUI | Give operators a TIA Portal-style alarm view — the entire point of the data pipeline | MEDIUM | `S7PlusAlarmsViewerPage.vue` — Vue 3 + Vuetify 3, separate route `/s7plus-alarms`, zero impact on existing `AlarmsViewerPage.vue` |
| TIA Portal-style columns in viewer | Familiar to operators who use TIA Portal; matches the reference screenshot | LOW | Source, Date, Time, Status, Acknowledge, Alarm class name, Event text, ID, Additional text 1–3 |
| Acknowledge button in viewer | Operators need to ack alarms from the SCADA UI | MEDIUM | Button enabled only for unacknowledged alarms; sends command to PLC via ack write-back pipeline |

### Differentiators (Nice-to-Have)

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Ack write-back to PLC | Close the loop — operator acks in SCADA, PLC sees it | HIGH | No existing driver API; S7CommPlus ack PDU is undocumented; Wireshark trace required before implementation |
| Auto-refresh alarm table | Live alarm view without manual reload | LOW | setInterval(5000) polling `GET /s7plusAlarms`; pause during user interaction |
| Filter by status (Incoming/Outgoing/All) | Reduce noise; match TIA Portal filter behavior | LOW | Client-side filter on `alarmState` field |
| Filter by alarm class | Show only specific alarm classes | LOW | Client-side filter on `alarmClassName` |
| Pagination | Handle large alarm histories | LOW | Vuetify `v-data-table` built-in pagination |
| "Receive alarms" PLC selector | Match TIA Portal UX for multi-PLC environments | LOW | Dropdown filtered by `connectionName` |

### Anti-Features (Explicitly Out of Scope for v1.1)

| Feature | Why Requested | Why Out of Scope | Alternative |
|---------|---------------|-----------------|-------------|
| Modify existing `AlarmsViewerPage.vue` | Consolidate viewers | Existing viewer must not change per requirement | Fully separate component and route |
| Multi-language alarm class names | Internationalization | TIA Portal user-defined class display names are NOT transmitted over S7CommPlus wire protocol | English static mapping only |
| Alarm class browse for custom class names | Show user-defined TIA Portal class names | Protocol does not deliver custom class name strings — only numeric IDs | Static map for protocol-fixed IDs; custom IDs show as "UserClass_N" |
| Real-time WebSocket push | Instant alarm appearance | Infrastructure complexity not justified for viewer PoC | 5s polling is sufficient |
| Alarm history export (CSV/PDF) | Reporting | Out of scope — viewer is the deliverable | MongoDB Compass or future phase |
| Alarm suppression / shelving | Advanced alarm management | ISA 18.2 advanced states; out of scope | Not implemented |
| Multi-LCID alarm texts | Multi-language deployment | LCID hardcoded to 1033 (English) from v1.0 | Single language only |

---

## TIA Portal Column Reference

Based on the screenshot provided, the viewer must display:

| TIA Portal Column | MongoDB Field | Notes |
|-------------------|---------------|-------|
| Source | `connectionName` | PLC connection name |
| Date | `plcTimestamp` (date part) | Format: M/D/YYYY |
| Time | `plcTimestamp` (time part) | Format: HH:MM:SS.mmm AM/PM |
| Status | `alarmState` | "Incoming" / "Outgoing" |
| Acknowledge | `ackState` | "Required" if unacked + ack button, "—" if not required, checkmark if acked |
| Alarm class name | `alarmClassName` | **New in v1.1** — derived from `alarmClass` numeric ID |
| Event text | `alarmText` | Primary alarm message |
| Help | `infotext` | Optional info text |
| Info text | (additional metadata) | TBD — map to available fields |
| ID | `cpuAlarmId` | Alarm identity |
| Additional text 1 | `additionalTexts[0]` | SD_1 resolved text |
| Additional text 2 | `additionalTexts[1]` | SD_2 resolved text |
| Additional text 3 | `additionalTexts[2]` | SD_3 resolved text |

---

## ISA 18.2 Alarm State Model (Viewer Display Logic)

For displaying alarm state correctly in the viewer:

| ISA 18.2 State | `alarmState` | `ackState` | Display |
|----------------|--------------|------------|---------|
| Unacknowledged Active | "coming" | false | Incoming / **Required** (highlighted) |
| Acknowledged Active | "coming" | true | Incoming / Acknowledged |
| Unacknowledged Cleared | "going" | false | Outgoing / **Required** |
| Acknowledged Cleared | "going" | true | Outgoing / — |

---

## Feature Dependencies

```
[ackState bug fix]
    └──required by──> [Ack write-back]
    └──required by──> [Ack button in viewer] (meaningless if ackState always true)
    └──required by──> [ISA 18.2 state display] (Acknowledge column)

[Alarm class name resolution]
    └──required by──> ["Alarm class name" viewer column]

[S7Plus Alarms Viewer page]
    └──required by──> [Ack button UI]
    └──depends on──> [GET /s7plusAlarms backend endpoint]
    └──depends on──> [POST /s7plusAlarmAck backend endpoint] (for ack button)

[Ack write-back to PLC]
    └──required by──> [POST /s7plusAlarmAck endpoint]
    └──depends on──> [ackState bug fix] (fix read before adding write)
    └──HIGH RISK: Wireshark trace required before implementation
```

---

## MVP Recommendation — Phase Order

**Phase 2 (Driver fixes):** ackState bug fix + alarm class name resolution + `alarmClassName` field in MongoDB. These are pure C# changes with no frontend dependency. Fix data correctness before building UI.

**Phase 3 (Read-only viewer):** S7Plus Alarms Viewer page — displays all existing + new fields. No ack button yet (ack write-back not implemented). Validate the viewer looks correct and data flows.

**Phase 4 (Ack write-back + ack button):** Implement ack command pipeline (C# → commandsQueue → Express → Vue). Enable ack button in viewer. Validate end-to-end.

---

## MongoDB Document — v1.1 Additions

New fields added to `s7plusAlarmEvents` in v1.1:

```json
{
  "alarmClassName": "4_UrgentOnderhoud",  // NEW: derived from alarmClass numeric ID
  "ackState": false,                        // FIXED: was always true in v1.0
}
```

---

## Feature Prioritization Matrix

| Feature | v1.1 Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Fix `ackState` always-true | HIGH | LOW-MEDIUM | P1 |
| Alarm class name (`alarmClassName`) | HIGH | LOW | P1 |
| S7Plus Alarms Viewer page | HIGH | MEDIUM | P1 |
| TIA Portal-style columns | HIGH | LOW | P1 |
| Ack write-back to PLC | HIGH | HIGH | P2 |
| Acknowledge button in viewer | HIGH | LOW (depends on P2) | P2 |
| Auto-refresh | MEDIUM | LOW | P2 |
| Status / class filter | MEDIUM | LOW | P3 |
| Pagination | MEDIUM | LOW | P3 |

---

## Sources

- `.planning/PROJECT.md` — v1.0 validated requirements, v1.1 goals
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsMultipleStai.cs` — AlarmDomain numeric encoding (1, 2, 256–272)
- `S7CommPlusClient/AlarmThread.cs` — ackState computation (`AckTimestamp != DateTime.MinValue`)
- `AdminUI/src/components/AlarmsViewerPage.vue` — existing tag-based viewer reference
- `AdminUI/package.json` — Vue 3.4.31, Vuetify 3.10 confirmed
- TIA Portal alarm viewer screenshot (user-provided) — column reference
- ISA 18.2 alarm management standard — four-state alarm model

---

*Feature research for: S7CommPlus alarm ack write-back, alarm class resolution, Vue 3 alarm viewer*
*Researched: 2026-03-18*

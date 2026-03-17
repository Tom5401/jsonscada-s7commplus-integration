# S7CommPlus Alarm Subscriptions for json-scada

## What This Is

A proof-of-concept C#/.NET driver extension that integrates native S7CommPlus alarm subscriptions into json-scada. The driver (S7CommPlusClient) connects to Siemens S7-1200/S7-1500 PLCs using the reverse-engineered S7CommPlus protocol, subscribes to alarm events, and writes alarm state, text, timestamps, and acknowledgement status into a dedicated MongoDB collection in json-scada. The existing read/write with symbolic access (optimized block access) is already working; this project adds the alarm subscription capability on top.

## Core Value

Alarms from S7-1200/S7-1500 PLCs appear in json-scada via native protocol subscription — not polling — with full metadata (text, timestamp, ack state), so operators see real events the moment they occur.

## Requirements

### Validated

- ✓ S7CommPlus connection to PLC with symbolic (optimized block) access — existing
- ✓ Read/write tag data via S7CommPlusClient integrated with json-scada MongoDB — existing

### Active

- [ ] Subscribe to PLC alarm events via S7CommPlus alarm subscription protocol
- [ ] Receive alarm state (active/cleared) from PLC subscription notifications
- [ ] Receive alarm texts (as configured in TIA Portal) for each alarm event
- [ ] Receive PLC-side timestamps for each alarm event
- [ ] Receive acknowledgement state for each alarm (read-only, passive receive)
- [ ] Write alarm events to a dedicated MongoDB collection in json-scada
- [ ] Handle alarm subscription lifecycle (create, maintain, tear down on disconnect)
- [ ] End-to-end demo: trigger alarm on PLC → event appears in MongoDB with all fields

### Out of Scope

- Sending ack commands from json-scada back to the PLC — PoC is read-only
- AdminUI changes or alarm display in json-scada frontend — data in MongoDB is the deliverable
- Polling-based alarm detection — native subscription only
- Upstream contribution to json-scada — internal use only
- Multi-PLC load distribution or HA redundancy — single connection PoC

## Context

- **S7CommPlusDriver** (`../S7CommPlusDriver/`) — open source reverse-engineered S7CommPlus library by thomas-v2. Contains partial alarm subscription implementation in `Alarming/` directory (AlarmsHandler.cs with `AlarmSubscriptionCreate`, `TestWaitForAlarmNotifications`, `AlarmSubscriptionDelete`). This code is marked as experimental/test.
- **S7CommPlusClient** (`json-scada/src/S7CommPlusClient/`) — the custom C#/.NET driver built on top of S7CommPlusDriver that integrates with json-scada via MongoDB. Currently handles read/write tag data.
- json-scada's native alarm model uses `alarmState`, `alarmed`, `timeTagAlarm` fields on realtimeData point documents (polling-based). There is no built-in mechanism for subscribing to PLC alarm events — this project introduces that capability via a separate MongoDB collection.
- The AlarmsHandler in S7CommPlusDriver uses hardcoded/unknown values (marked with `// TODO Unknown`) for some subscription parameters — these will need investigation.

## Constraints

- **Simplicity**: PoC — minimal code, no over-engineering. Extend S7CommPlusClient cleanly, don't refactor existing working parts.
- **Protocol**: S7CommPlus only (no fallback to classic S7Comm). Target: S7-1200 and S7-1500 PLCs.
- **Language**: C#/.NET (consistent with existing S7CommPlusClient).
- **Storage**: MongoDB (consistent with json-scada architecture). New collection for alarm events.
- **Ack direction**: Read-only. No write-back to PLC for this PoC.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Separate MongoDB collection for alarms | Alarm events are time-series log entries, not live point state — separate collection is cleaner than polluting realtimeData | — Pending |
| Extend S7CommPlusClient (not new project) | Keep the PoC cohesive — alarm subscription belongs alongside the existing tag data driver | — Pending |
| Read-only ack | Reduces scope; ack write-back can be added later if needed | — Pending |

---
*Last updated: 2026-03-17 after initialization*

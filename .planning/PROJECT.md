# S7CommPlus Alarm Subscriptions for json-scada

## What This Is

A proof-of-concept C#/.NET driver extension that integrates native S7CommPlus alarm subscriptions into json-scada. The driver (S7CommPlusClient) connects to Siemens S7-1200/S7-1500 PLCs using the reverse-engineered S7CommPlus protocol, subscribes to alarm events, and writes alarm state, text, timestamps, acknowledgement status, and associated data values into a dedicated MongoDB collection. Tag read/write with symbolic (optimized block) access continues in parallel on a separate connection and thread.

**Shipped:** v1.0 — PoC validated live with PLCSIM Advanced V8 + TIA Portal v21 on 2026-03-18.

## Core Value

Alarms from S7-1200/S7-1500 PLCs appear in json-scada via native protocol subscription — not polling — with full metadata (text, timestamp, ack state, associated values), so operators see real events the moment they occur.

## Requirements

### Validated

- ✓ S7CommPlus connection to PLC with symbolic (optimized block) access — existing
- ✓ Read/write tag data via S7CommPlusClient integrated with json-scada MongoDB — existing
- ✓ Subscribe to PLC alarm events via S7CommPlus alarm subscription protocol — v1.0
- ✓ Receive alarm state (active/cleared) from PLC subscription notifications — v1.0
- ✓ Receive alarm texts (as configured in TIA Portal) for each alarm event — v1.0
- ✓ Receive PLC-side timestamps for each alarm event — v1.0
- ✓ Receive acknowledgement state for each alarm (read-only, passive receive) — v1.0
- ✓ Write alarm events to a dedicated MongoDB collection in json-scada — v1.0
- ✓ Handle alarm subscription lifecycle (create, maintain, tear down on disconnect) — v1.0
- ✓ End-to-end demo: trigger alarm on PLC → event appears in MongoDB with all fields — v1.0

### Active

*(None — v1.0 PoC complete. Next requirements defined in new milestone.)*

### Out of Scope

- Sending ack commands from json-scada back to the PLC — PoC is read-only
- AdminUI changes or alarm display in json-scada frontend — data in MongoDB is the deliverable
- Polling-based alarm detection — native subscription only
- Upstream contribution to json-scada — internal use only
- Multi-PLC load distribution or HA redundancy — single connection PoC
- Multi-language alarm text — LCID hardcoded to 1033 (English)

## Context

**Current state (v1.0):**
- S7CommPlusClient extended with AlarmThread.cs (~250 LOC) and alarm fields in Common.cs
- S7CommPlusDriver submodule patched: AlarmsHandler.cs with 5 bug fixes + WaitForAlarmNotification public API
- MongoDB collection: `s7plusAlarmEvents` — alarm event documents with 14 fields
- Validated live: PLCSIM Advanced V8, TIA Portal v21, S7-1515 PLC, Windows VM

**Known technical debt:**
- Alarm ack write-back not implemented (deferred)
- LCID hardcoded to 1033 (English only)
- No auto-resubscribe after connection loss (reconnect restarts alarm thread correctly, but no retry on alarm subscription failure mid-session)
- Join(3000) shorter than 5s alarm receive timeout — alarm thread may outlive bounded join on stop (non-defect)

## Constraints

- **Simplicity**: PoC — minimal code, no over-engineering
- **Protocol**: S7CommPlus only. Target: S7-1200 and S7-1500 PLCs
- **Language**: C#/.NET (consistent with existing S7CommPlusClient)
- **Storage**: MongoDB (consistent with json-scada architecture)
- **Ack direction**: Read-only. No write-back to PLC

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Separate MongoDB collection `s7plusAlarmEvents` | Alarm events are time-series log entries, not live point state | ✓ Good — clean separation confirmed |
| Extend S7CommPlusClient (not new project) | Cohesive PoC; alarm subscription belongs alongside tag driver | ✓ Good — single build, shared config |
| Read-only ack | Reduces PoC scope; write-back can be added later | — Pending for future milestone |
| Credit-limit replenishment inside WaitForAlarmNotification | Keeps alarm thread minimal; protocol knowledge stays in driver layer | ✓ Good — alarm thread is simple loop |
| Separate S7CommPlusConnection for alarm thread | Avoids shared state and PDU interleaving with tag connection | ✓ Good — no threading issues observed |
| InsertOneAsync + GetAwaiter().GetResult() for alarm writes | Sync-over-async acceptable for low-frequency events in sync thread | ✓ Good — no performance issues |
| LCID 1033 (English) hardcoded | Matches project language; multi-language out of scope for PoC | ⚠ Revisit — if multi-language needed |
| Treat WaitForNewS7plusReceived timeout as non-fatal | 5s idle window is normal when no alarms are active | ✓ Good — loop continues correctly |

---
*Last updated: 2026-03-18 after v1.0 milestone*

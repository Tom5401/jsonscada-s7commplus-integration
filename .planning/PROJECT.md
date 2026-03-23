# S7CommPlus Alarm Subscriptions for json-scada

## What This Is

A proof-of-concept C#/.NET driver extension that integrates native S7CommPlus alarm subscriptions into json-scada with full alarm management capability. The driver (S7CommPlusClient) connects to Siemens S7-1200/S7-1500 PLCs using the reverse-engineered S7CommPlus protocol, subscribes to alarm events, writes alarm state, text, timestamps, acknowledgement status, and associated data values into a dedicated MongoDB collection, and sends acknowledgement commands back to the PLC. A dedicated S7Plus Alarms Viewer in AdminUI provides operators with a TIA Portal-equivalent interface.

**Shipped:** v1.1 — Full alarm management loop validated live with PLCSIM Advanced V8 + TIA Portal v21 on 2026-03-19. Tag read/write continues in parallel.

## Current Milestone: v1.2 Alarm Origin & Cleanup

**Goal:** Enrich alarm events with PLC origin (DB/FB name) resolved from a live relationID lookup at driver startup, and give operators the ability to delete alarm history from the viewer.

**Target features:**
- Per-row Delete and Bulk "Delete Filtered" in S7PlusAlarmsViewerPage
- RelationID stored in s7plusAlarmEvents MongoDB documents
- Driver queries PLC at startup to map relationID → DB/FB Name
- DB/FB Name column(s) added to alarm viewer table

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
- ✓ MongoDB ackState field correctly reflects PLC acknowledgement state — v1.1
- ✓ MongoDB alarm documents include resolved alarmClassName string — v1.1
- ✓ Acknowledgement command sent from json-scada received and applied by PLC — v1.1
- ✓ Dedicated S7Plus Alarms Viewer page in AdminUI with navigation — v1.1
- ✓ Alarm viewer displays TIA Portal-equivalent columns (11 columns) — v1.1
- ✓ Alarm viewer auto-refreshes every 5 seconds — v1.1
- ✓ Per-row Acknowledge button — ack loop from UI to PLC — v1.1
- ✓ Status and alarm class filters — v1.1

### Active

- ✓ Store relationID (BsonInt64) and dbNumber (BsonInt32) in s7plusAlarmEvents MongoDB documents — Validated in Phase 5: Driver RelationId Fields
- ✓ Driver queries PLC at startup to build relationID → DB/FB Name map — Validated in Phase 6: driver-startup-db-name-map
- [ ] DB Name (and FB Name if available) returned by alarm list API
- [ ] Alarm viewer displays DB/FB origin as new column(s)
- [ ] Per-row Delete button in alarm viewer (mirrors Ack button)
- [ ] Bulk "Delete Filtered" button removes all currently visible rows immediately

### Out of Scope

- Polling-based alarm detection — native subscription only (carried from v1.0)
- Upstream contribution to json-scada — internal use only
- Polling-based alarm detection — native subscription only
- Upstream contribution to json-scada — internal use only
- Multi-PLC load distribution or HA redundancy — single connection PoC
- Multi-language alarm text — LCID hardcoded to 1033 (English)

## Context

**Current state (v1.1):**
- S7CommPlusClient: AlarmThread.cs (~280 LOC), MongoCommands.cs with ack branch, AlarmAck.cs, Common.cs
- S7CommPlusDriver submodule: AlarmsHandler.cs with 5 bug fixes + WaitForAlarmNotification + SendAlarmAck
- MongoDB collection: `s7plusAlarmEvents` — alarm event documents with 15+ fields (ackState fixed, alarmClassName added)
- AdminUI: S7PlusAlarmsViewerPage.vue (11-column table, auto-refresh, filters, per-row Ack button)
- Backend: `GET /Invoke/auth/listS7PlusAlarms` + `POST /Invoke/auth/ackS7PlusAlarm` in server_realtime_auth
- Validated live: PLCSIM Advanced V8, TIA Portal v21, S7-1515 PLC, Windows VM

**Known technical debt:**
- Spinner resolution edge case: stillPending.add(id) missing in fetchAlarms — spinner always clears after one 5s poll regardless of ackState; unobservable in normal <1s ack scenario
- connectionId (MongoDB field name) vs connectionNumber (API body key) naming inconsistency
- LCID hardcoded to 1033 (English only)
- No auto-resubscribe after alarm subscription failure mid-session
- No automated tests (PoC — all Nyquist VALIDATION.md files are draft status)
- Phase 4 missing VERIFICATION.md (UAT.md covers functional testing)

## Constraints

- **Simplicity**: PoC — minimal code, no over-engineering
- **Protocol**: S7CommPlus only. Target: S7-1200 and S7-1500 PLCs
- **Language**: C#/.NET (consistent with existing S7CommPlusClient)
- **Storage**: MongoDB (consistent with json-scada architecture)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Separate MongoDB collection `s7plusAlarmEvents` | Alarm events are time-series log entries, not live point state | ✓ Good — clean separation confirmed |
| Extend S7CommPlusClient (not new project) | Cohesive PoC; alarm subscription belongs alongside tag driver | ✓ Good — single build, shared config |
| Read-only ack (v1.0) | Reduces PoC scope; write-back deferred | ✓ Resolved — v1.1 delivered full ack write-back |
| Credit-limit replenishment inside WaitForAlarmNotification | Keeps alarm thread minimal; protocol knowledge stays in driver layer | ✓ Good — alarm thread is simple loop |
| Separate S7CommPlusConnection for alarm thread | Avoids shared state and PDU interleaving with tag connection | ✓ Good — no threading issues observed |
| InsertOneAsync + GetAwaiter().GetResult() for alarm writes | Sync-over-async acceptable for low-frequency events in sync thread | ✓ Good — no performance issues |
| LCID 1033 (English) hardcoded | Matches project language; multi-language out of scope for PoC | ⚠ Revisit — if multi-language needed |
| Treat WaitForNewS7plusReceived timeout as non-fatal | 5s idle window is normal when no alarms are active | ✓ Good — loop continues correctly |
| AckJob uses CreateObjectRequest (not SetVariableRequest) | Decoded via Wireshark; DynObjX7 dynamic RelId is definitive marker | ✓ Good — confirmed working against PLCSIM |
| SendAlarmAck routed through alarmConn | Avoids polling contention; AckJob completion notification arrives naturally on alarm subscription connection | ✓ Good — no contention issues |
| AckJob completion notification (ClassId=3636) skipped in AlarmThread | P2ReturnValue=0x81 handled correctly but PObject attributes are AckJob-specific; skipping avoids KeyNotFoundException in AlarmsDai | ✓ Good — correct fix |
| PendingAcks ConcurrentQueue for ack dispatch | AlarmThread dequeues and sends ack; MongoCommands awaits TaskCompletionSource; no dedicated third connection | ✓ Good — clean architecture |

---
## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-03-23 — Phase 6 complete: driver builds RelationId→DB name map at startup and writes originDbName to every alarm document*

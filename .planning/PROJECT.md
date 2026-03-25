# S7CommPlus Alarm Subscriptions for json-scada

## What This Is

A proof-of-concept C#/.NET driver extension that integrates native S7CommPlus alarm subscriptions into json-scada with full alarm management capability. The driver (S7CommPlusClient) connects to Siemens S7-1200/S7-1500 PLCs using the reverse-engineered S7CommPlus protocol, subscribes to alarm events, writes alarm state, text, timestamps, acknowledgement status, origin DB name, and associated data values into a dedicated MongoDB collection, and sends acknowledgement commands back to the PLC. A dedicated S7Plus Alarms Viewer in AdminUI provides operators with a TIA Portal-equivalent interface including origin columns and alarm history deletion.

**Shipped:** v1.2 — Alarm origin enrichment and history deletion validated 2026-03-24. Full alarm management loop (subscribe → display → ack → delete) complete.

## Current Milestone: v1.3 Alarm Viewer Enhancements & Priority

**Goal:** Enrich alarm data with priority and ackability, improve the viewer with better UX (timestamp, bulk ack, pagination, sorting, source filter), and fix the 200-alarm display cap.

**Target features:**
- isAcknowledgeable flag derived from AlarmClass (class 33 = true) stored in MongoDB
- alarmPriority number stored in every alarm document
- Single timestamp column replacing separate date+time columns (format: 2026-03-24_12:57:10.758)
- Ack All button — acks all unacked alarms matching current filter
- Placeholder substitution extended to alarm text and info text (@N%x@ format)
- Sortable priority column in alarm viewer
- Source PLC filter (based on connectionName)
- Pagination in viewer (no API limit, paged display in UI)
- Fix 200-alarm cap in listS7PlusAlarms API

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
- ✓ Store relationId (BsonInt64) and dbNumber (BsonInt32) in s7plusAlarmEvents MongoDB documents — v1.2
- ✓ Driver queries PLC at startup to build relationId → DB Name map (GetListOfDatablocks) — v1.2
- ✓ originDbName (TIA Portal datablock name) stored in every alarm document at write time — v1.2
- ✓ `_id` exposed in alarm list API response for delete targeting — v1.2
- ✓ `POST /Invoke/auth/deleteS7PlusAlarms` endpoint with admin guard, ids-based and filter-based delete — v1.2
- ✓ Alarm viewer displays Origin DB Name and DB Number columns — v1.2
- ✓ Per-row Delete button in alarm viewer with Coming+unacked warning dialog — v1.2
- ✓ Bulk "Delete Filtered" button removes all currently visible rows with confirmation dialog — v1.2
- ✓ `isAcknowledgeable` boolean stored in every alarm document (true for alarmClass 33/37/39) — v1.3 Phase 9
- ✓ `alarmText` and `infoText` resolved from `@N%x@` placeholder templates at write time — v1.3 Phase 9
- ✓ `listS7PlusAlarms` returns all alarm documents (no 200-alarm ceiling) — Validated in Phase 10: api-cap-removal
- ✓ `{ createdAt: -1 }` index on `s7plusAlarmEvents` created at server startup for query performance — Validated in Phase 10: api-cap-removal
- ✓ Single Timestamp column (format `2026-03-24_12:57:10.758`) replaces separate Date+Time columns — Validated in Phase 11: vue-ui-enhancements
- ✓ Priority column in alarm viewer, sortable by clicking the header — Validated in Phase 11: vue-ui-enhancements
- ✓ Ack indicator: alarms with `isAcknowledgeable === false` show `-` in Acknowledge column — Validated in Phase 11: vue-ui-enhancements
- ✓ Source PLC filter dropdown, consistent with Status and Alarm Class filters — Validated in Phase 11: vue-ui-enhancements
- ✓ Ack All bulk action with confirmation dialog showing count; sequential loop, single failure non-blocking — Validated in Phase 11: vue-ui-enhancements
- ✓ Page preserved across 5-second auto-refresh via `v-model:page` — Validated in Phase 11: vue-ui-enhancements

### Active

*(v1.3 milestone complete — see REQUIREMENTS.md for any carry-forward items)*

### Out of Scope

- Polling-based alarm detection — native subscription only (carried from v1.0)
- Upstream contribution to json-scada — internal use only
- Multi-PLC load distribution or HA redundancy — single connection PoC
- Multi-language alarm text — LCID hardcoded to 1033 (English)
- RelationId (raw) column in alarm viewer — D-02: raw 64-bit integer adds no operator value (dropped v1.2)
- FB type name column — requires GetTypeInformation browse per DB; high complexity
- Alarm log retention policy (auto-expire) — not yet prioritized
- Server-side guard preventing active-alarm deletion — client-side confirmation only

## Context

**Current state (v1.3 complete — all 11 phases done):**
- Phase 11 (Vue UI Enhancements) complete — timestamp column, priority sort, ack indicator, source filter, Ack All bulk action, page preservation all live in S7PlusAlarmsViewerPage.vue
- v1.3 milestone fully shipped: Driver Enrichment (Phase 9) + API Cap Removal (Phase 10) + Vue UI Enhancements (Phase 11)

**Last state (v1.2):**
- S7CommPlusClient: AlarmThread.cs, MongoCommands.cs (queued ack via PendingAcks ConcurrentQueue), Common.cs (RelationIdNameMap + PendingAlarmAck), Program.cs (GetListOfDatablocks browse at startup)
- S7CommPlusDriver submodule: AlarmsHandler.cs with 5 bug fixes + WaitForAlarmNotification + SendAlarmAck
- MongoDB collection: `s7plusAlarmEvents` — alarm event documents with 18 fields (incl. relationId, dbNumber, originDbName)
- AdminUI: S7PlusAlarmsViewerPage.vue (14-column table, auto-refresh, filters, Ack button, per-row Delete button, bulk Delete Filtered button, confirmation dialogs)
- Backend: `GET /Invoke/auth/listS7PlusAlarms` (returns all fields) + `POST /Invoke/auth/ackS7PlusAlarm` + `POST /Invoke/auth/deleteS7PlusAlarms` in server_realtime_auth
- Validated live: PLCSIM Advanced V8, TIA Portal v21, S7-1515 PLC, Windows VM

**Known technical debt:**
- Spinner resolution edge case: stillPending.add(id) missing in fetchAlarms — spinner always clears after one 5s poll regardless of ackState; unobservable in normal <1s ack scenario
- connectionId (MongoDB field name) vs connectionNumber (API body key) naming inconsistency
- LCID hardcoded to 1033 (English only)
- No auto-resubscribe after alarm subscription failure mid-session
- No automated tests (PoC — all Nyquist VALIDATION.md files are draft status)
- Phase 4 missing VERIFICATION.md (UAT.md covers functional testing)
- listS7PlusAlarms error paths return HTTP 200 (not 4xx/5xx) — pre-existing json-scada pattern; Vue silently ignores non-array response

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
| dbNumber = `relationId & 0xFFFF` (lower 16 bits) | Confirmed by S7CommPlusConnection.cs:1247 — `area = relid >> 16`, `num = relid & 0xffff`; ORIGIN-02 formula in REQUIREMENTS.md was wrong (`>> 16`) | ✓ Good — correct extraction confirmed |
| Browse on tag connection (srv.connection) before alarm thread start | alarmConn does not exist yet at this point; browse is a one-time startup operation | ✓ Good — no ordering issue |
| Empty-string fallback for originDbName (not null) | Consistent schema for API consumers; `""` signals unknown DB without requiring null checks | ✓ Good — clean consumer contract |
| HTTP 204 No Content on delete success | Standard REST pattern for delete; no body needed | ✓ Good — Phase 8 ignores response body |
| Empty filter guard rejects `{ filter: {} }` with 400 | Prevents accidental full-collection wipe | ✓ Good — defensive guard in place |
| RelationId column intentionally omitted from alarm viewer (D-02) | Raw 64-bit integer adds no operator value; origin is already human-readable via originDbName | ✓ Good — confirmed right call |
| NativeDoubleSerializer removed from rtCommand | ae59194e approach: standard BSON double deserializer sufficient for command fields | ✓ Good — no issues observed |

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
*Last updated: 2026-03-25 (Phase 11 complete — v1.3 milestone done)

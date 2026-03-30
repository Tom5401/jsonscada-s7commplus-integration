# S7CommPlus Alarm Subscriptions for json-scada

## What This Is

A proof-of-concept C#/.NET driver extension that integrates native S7CommPlus alarm subscriptions into json-scada with full alarm management capability. The driver (S7CommPlusClient) connects to Siemens S7-1200/S7-1500 PLCs using the reverse-engineered S7CommPlus protocol, subscribes to alarm events, writes alarm state, text, timestamps, acknowledgement status, origin DB name, and associated data values into a dedicated MongoDB collection, and sends acknowledgement commands back to the PLC. A dedicated S7Plus Alarms Viewer in AdminUI provides operators with a TIA Portal-equivalent interface including origin columns and alarm history deletion.

**Shipped:** v1.4 — Tag Tree Browser milestone complete 2026-03-27. Full operator workflow: alarms → origin DB name → tag tree with live values.
**Phase 17 complete (2026-03-30):** TagTreeBrowserPage rewritten with lazy load-children (one level per expand via listS7PlusChildNodes), scoped value refresh (only expanded paths), and TRUE/FALSE boolean display.
**Phase 18 complete (2026-03-30):** PushValueDialog.vue added to TagTreeBrowser — operators can push new values to writable PLC tags directly from the tag tree; Modify button shows only on writable leaves (commandOfSupervised != 0); OPC WriteRequest (ServiceId 671) POSTed to /Invoke/; digital tags use TRUE/FALSE select, analog/string use text field; command stub tags deduplicated from tree view.

## Current Milestone: v1.5 TagTreeBrowser Overhaul

**Goal:** Transform TagTreeBrowser from a proof-of-concept into a production-ready tool that handles 100k+ tag databases with lazy loading at every tree level, real value display, value writing, and non-datablock tag support.

**Target features:**
- Lazy loading at every tree level — new backend endpoint returning direct children per path; Vuetify load-children; value refresh scoped to visible/expanded leaves
- Real value display (TRUE/FALSE instead of 0/1) — matching tabular view behavior
- Write/push values — open existing tabular view push-value window from TagTreeBrowser leaf nodes
- Non-datablock tags (MArea, QArea) — surfaced alongside datablocks in DatablockBrowser

## Shipped: v1.4 Tag Tree Browser (2026-03-27)

**Goal:** Mirror TIA Portal's hierarchical tag browsing in AdminUI — operators can navigate datablock structure, see live values for configured tags, and open a tag tree directly from the alarms viewer.

**Delivered:**
- DatablockBrowser page — lists all PLC datablocks in a table, added to AdminUI main menu
- TagTreeBrowser page — hierarchical tag tree (folder/leaf) with live values auto-refreshing every 5 seconds, touch-on-expand, auto-expand first level
- Backend: `s7plusDatablocks` MongoDB collection + 3 HTTP endpoints (listS7PlusDatablocks, listS7PlusTagsForDb, touchS7PlusActiveTagRequests)
- S7AlarmsViewer integration — `originDbName` cells render as clickable links opening TagTreeBrowser in a new tab

## Shipped: v1.3 Alarm Viewer Enhancements & Priority (2026-03-25)

All 10 requirements validated. The alarm viewer now shows enriched alarm data with full UX improvements:
isAcknowledgeable flag, resolved alarm text, no API cap, combined timestamp, sortable priority, ack indicator, source filter, Ack All bulk action, and page preservation.

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

### Validated (v1.4)

- ✓ Driver stores full PLC datablock list in `s7plusDatablocks` MongoDB collection at startup, upserted on `{connectionNumber, db_name}` — Validated in Phase 12: driver-datablock-persistence
- ✓ `GET /Invoke/auth/listS7PlusDatablocks` returns all datablocks for a connection, sorted by `db_name`, admin-guarded — Validated in Phase 13: backend-api-datablocks-tag-endpoints
- ✓ `GET /Invoke/auth/listS7PlusTagsForDb` returns realtimeData tags matching `protocolSourceBrowsePath` prefix for a given dbName, admin-guarded — Validated in Phase 13: backend-api-datablocks-tag-endpoints
- ✓ `POST /Invoke/auth/touchS7PlusActiveTagRequests` bulk-upserts tag requests into `activeTagRequests` with `source: 'tag-tree'` and TTL, admin-guarded — Validated in Phase 13: backend-api-datablocks-tag-endpoints
- ✓ `DatablockBrowserPage.vue` at `/s7plus-datablocks` — connection dropdown (starts empty), datablock table (db_name + db_number), per-row "Browse Tags" button opens TagTreeBrowser in new tab — Validated in Phase 14: datablockbrowser
- ✓ `TagTreeBrowserPage.vue` at `/s7plus-tag-tree` — hierarchical tree from `ungroupedDescription` full path, 5s in-place value refresh, touch-on-expand, auto-expand first level; structured datablocks (nested UDT structs) render folder/leaf hierarchy correctly — Validated in Phase 15: tagtreebrowser-integration
- ✓ `originDbName` cells in S7PlusAlarmsViewerPage render as clickable links (non-empty) opening TagTreeBrowser in new tab with `db=` and `connectionNumber=` params — Validated in Phase 15: tagtreebrowser-integration

### Validated (v1.5)

- ✓ Lazy loading at every tree level — `listS7PlusChildNodes` endpoint, Vuetify `:load-children` callback, `onLoadChildren` fetches one level per expand, Vuetify caches children after first open — Validated in Phase 17: lazy-tree-loading-boolean-display
- ✓ Real value display: digital tags (type=`"digital"`) show `TRUE`/`FALSE` via `formatLeafValue()` instead of 0/1 — Validated in Phase 17: lazy-tree-loading-boolean-display
- ✓ Value refresh scoped to expanded nodes only — `getExpandedParentPaths()` + `Promise.all` parallel `listS7PlusChildNodes` calls; only visible nodes refreshed — Validated in Phase 17: lazy-tree-loading-boolean-display

### Active

<!-- Current scope — v1.5 TagTreeBrowser Overhaul (phases 18–19) -->

- ✓ Write/push values from TagTreeBrowser leaf nodes — PushValueDialog.vue with OPC WriteRequest, Modify button on writable leaves only, digital/analog branching, inline success/error feedback, auto-close — Validated in Phase 18: value-write-dialog
- [ ] Non-datablock tags (MArea, QArea) alongside datablocks in DatablockBrowser

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

**Current state (v1.4 shipped 2026-03-27):**
- 15 phases complete across 5 milestones (v1.0–v1.4)
- Full operator workflow: subscribe alarms → display with metadata → ack/delete → click origin DB → browse tag tree with live values
- TagTreeBrowserPage.vue: hierarchical tree from `ungroupedDescription` full path; structured datablocks with nested UDT structs render as folder/leaf hierarchy; 5s in-place value refresh preserving expand state; touch-on-expand to extend TTL
- DatablockBrowserPage.vue: connection dropdown, datablock table, "Browse Tags" button
- S7PlusAlarmsViewerPage.vue: `originDbName` column clickable link → TagTreeBrowser in new tab
- Bug fix: `protocolSourceBrowsePath` is the parent path (not leaf path) — tree must use `ungroupedDescription` (= `varInfo.Name`) for correct folder/leaf construction
- Validated against: PLCSIM Advanced V8, TIA Portal v21, S7-1515 PLC, Windows VM

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
*Last updated: 2026-03-30 — Milestone v1.5 TagTreeBrowser Overhaul started*


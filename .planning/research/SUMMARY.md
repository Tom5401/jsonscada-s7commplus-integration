# Project Research Summary

**Project:** S7CommPlus Alarm Origin & Cleanup (v1.2)
**Domain:** Industrial SCADA alarm enrichment and history management — additions to existing json-scada S7CommPlusClient driver
**Researched:** 2026-03-23
**Confidence:** HIGH — all findings verified directly from submodule source code, live project files, and running codebase

## Executive Summary

v1.2 adds two self-contained capabilities to an already-validated stack: alarm origin enrichment (resolving the PLC datablock name that produced each alarm) and alarm history management (per-row and bulk delete from the MongoDB alarm log). Both features extend the existing pattern — the C# driver writes documents, Node.js serves them, Vue 3 displays them — without introducing new dependencies or new protocol functions. Every API required (`GetListOfDatablocks`, `ExploreASAlarms`, `deleteOne`/`deleteMany`, `v-btn :loading`) is already present in the installed versions of all components. The scope is additive: six files are modified, one new method is added, and the submodule is untouched.

The recommended approach is a four-phase build ordered by data dependency. The origin enrichment work (C# driver) must precede the frontend column addition because the `originDbName` field must exist in MongoDB documents before it can be displayed. The delete infrastructure (Node.js endpoint + `_id` exposure) is independent and can be built in parallel with origin enrichment, but both must complete before frontend wiring. Research confirmed that the entire DB-name lookup can use `GetListOfDatablocks`, which is already called at startup for tag browsing — no new protocol round-trips beyond the two existing `ExploreRequest` calls are needed.

The key risk is that the `RelationId`-to-block-name map built at startup becomes stale after a TIA Portal compile-and-download, which reassigns internal PLC object IDs without dropping the TCP connection. The mitigation is to treat the map as startup-time best-effort and store the resolved name in the alarm document at write time, making each event record self-contained and immutable. A secondary risk is a silent crash in `ExploreASAlarms` on PLCs with no user alarms configured (`.First()` LINQ call on an empty sequence). Both risks have straightforward defensive code patterns that do not require architectural changes — they require defensive coding and documentation.

## Key Findings

### Recommended Stack

No new dependencies are required for v1.2. The full stack is inherited from v1.1. The only changes are new fields in existing data structures and new methods on existing classes.

**Core technologies:**

- **S7CommPlusDriver (submodule, C#):** PLC protocol layer — `GetListOfDatablocks` (two `ExploreRequest` round-trips) provides the `RelationId → db_name` map; `PObject.RelationId` in notification PDUs provides the per-alarm identifier; both are already compiled into the project as a local submodule with no NuGet change needed
- **MongoDB.Driver (C#) 3.4.2:** Alarm document writes — two new fields (`relationId` as BsonInt64, `originDbName` as string) added to existing `BuildAlarmDocument` return value via the same `InsertOneAsync` call already in use
- **mongodb (Node.js) ^7.0.0:** `deleteOne`/`deleteMany` for single-row and bulk delete endpoints; `deletedCount` return is stable since driver v4
- **Vue 3 + Vuetify 3 (existing):** Per-row Delete button mirrors existing Ack button template slot pattern; bulk "Delete Filtered" button uses `filteredAlarms` computed property already present in v1.1
- **server_realtime_auth (Node.js, existing):** New `POST` endpoint for delete operations; auth guard (`authJwt.isAdmin`) mirrors existing ack endpoint pattern exactly

**Critical version note:** Store `relationId` as `BsonInt64` (Int64), not Int32. The area code in the upper 16 bits (e.g., `0x8a0e`) produces values such as `0x8a0e0005 = 2,317,140,997`, which exceeds Int32 max and would be silently truncated or sign-extended if stored incorrectly.

### Expected Features

All features listed below are P1 (required for v1.2 milestone completion). There are no optional P1 items.

**Must have (table stakes):**

- `relationId` stored in `s7plusAlarmEvents` documents — computed from `cpuAlarmId >> 32`; required foundation for origin display and makes the raw ID queryable independently of the resolved name
- Startup DB name map built via `GetListOfDatablocks` on the tag connection before `AlarmSubscriptionCreate` — produces `Dictionary<uint, string>` keyed by `db_block_relid`
- `originDbName` stored in every new alarm document at write time — map lookup in `BuildAlarmDocument` with graceful empty-string fallback; denormalized for zero-join reads
- "Origin DB" column in `S7PlusAlarmsViewerPage.vue` — one entry added to `headers` array; renders `item.originDbName`
- `_id` included in `listS7PlusAlarms` response — remove `{ _id: 0 }` projection exclusion; ObjectId serialized as string for JSON transport
- `POST /Invoke/auth/deleteS7PlusAlarms` endpoint — single endpoint handles both single-row (body: `{ ids: [...] }`) and bulk-by-filter (body: `{ filter: { alarmState, alarmClassName } }`)
- Per-row Delete button in viewer — mirrors Ack button template slot; calls delete endpoint with `item._id`; removes row optimistically from `alarms.value` on success
- "Delete Filtered (N)" button in viewer — renders visible row count; sends current `statusFilter` + `alarmClassFilter` to bulk delete endpoint; calls `fetchAlarms()` after success

**Should have (UX quality):**

- Confirmation dialog for active/unacked (`alarmState === 'Coming'` + `ackState === false`) rows before delete — prevents silent deletion of alarms still active on the PLC
- "(unavailable)" fallback display in the Origin column when origin browse failed at startup, with tooltip context

**Defer (v2+):**

- FB type name column — requires `GetTypeInformation` browse per DB after `GetListOfDatablocks`; engineering-level detail operators do not need
- Symbolic variable path within DB — requires matching `Alid` to variable type info; full DB browse; high complexity; not in alarm notification metadata
- Alarm log retention policy (auto-expire after N days) — operational need, not PoC
- Auto-resubscribe alarm subscription after connection failure — known technical debt, out of scope for v1.2

### Architecture Approach

The architecture is a strict layered extension: the C# driver enriches documents at write time, Node.js exposes delete operations, and Vue consumes both. No new services, no new collections, no new inter-process communication. The in-memory `Dictionary<uint, string>` on `S7CP_connection` (named `RelationIdNameMap`) follows the existing `AddressCache` field pattern in `Common.cs` and is marked `[BsonIgnore]` so it is never serialized. The browse runs once in `ConnectionThread` on `srv.connection` (the tag connection) before `AlarmThread.Start()`, ensuring the map is populated before the first notification arrives. The submodule (`BrowseAlarms.cs`, `AlarmsHandler.cs`, `AlarmsDai.cs`, `PObject.cs`) is used as-is with zero modifications.

**Major components and what changes:**

1. **`Common.cs` S7CP_connection** — gains `Dictionary<uint, string> RelationIdNameMap` field (`[BsonIgnore]`)
2. **`Program.cs` ConnectionThread()** — calls new `BrowseRelationIdNames(srv)` after `Connect()` succeeds, before `AlarmThread.Start()`
3. **`AlarmThread.cs` AlarmThread()** — reads `noti.P2Objects[0].RelationId` before `FromNotificationObject`; passes to `BuildAlarmDocument`
4. **`AlarmThread.cs` BuildAlarmDocument()** — accepts `uint relationId` parameter; writes `relationId` (BsonInt64) and `originDbName` (string) fields
5. **`AlarmThread.cs` or new `AlarmOrigin.cs`** — `BrowseRelationIdNames(S7CP_connection srv)` method; calls `GetListOfDatablocks`, builds map; wrapped in try-catch with empty-map fallback
6. **`server_realtime_auth/index.js`** — remove `_id: 0` from `listS7PlusAlarms` projection; add `POST /Invoke/auth/deleteS7PlusAlarms` with `authJwt.isAdmin`
7. **`S7PlusAlarmsViewerPage.vue`** — add `originDbName` column, Delete column with per-row button, "Delete Filtered" toolbar button, `deleteAlarm()` and `deleteFiltered()` functions

### Critical Pitfalls

1. **RelationId instability after TIA Portal download** — `RelationId` is a compile-time PLC object-tree identifier; a TIA Portal compile-and-download reassigns it without dropping the TCP connection, silently staling the in-memory map. Store `originDbName` in the alarm document at write time (immutable log entry); accept the map as startup-best-effort; document as known behavior; driver restart forces a fresh browse.

2. **ExploreASAlarms throws on PLCs with no user alarms** — `BrowseAlarms.cs` uses `.First(o => ...)` LINQ which throws `InvalidOperationException` on an empty response. Wrap the entire browse call in try-catch; replace `.First()` with `.FirstOrDefault()` + null check; treat empty or error result as an empty map and continue — alarm subscription must not depend on browse success.

3. **Browse called on the wrong connection** — `ExploreASAlarms` must run on `srv.connection` (tag connection), not on `alarmConn` (alarm subscription connection). Calling it on `alarmConn` risks PDU interleaving with the subscription protocol. Build the map in `ConnectionThread` before `AlarmThread.Start()` using `srv.connection` exclusively.

4. **Delete race with 5-second auto-refresh** — the auto-refresh fires independently of in-flight delete operations, causing deleted rows to reappear for up to one refresh cycle. Optimistically remove the deleted row from `alarms.value` immediately on successful delete response; treat `deletedCount === 0` as idempotent success in the backend.

5. **Active/unacked alarm deleted silently** — deleting a `Coming`+unacked alarm removes the MongoDB document while the PLC alarm state remains active; the next "Going" notification creates an orphaned document. Surface a confirmation dialog in Vue when `alarmState === 'Coming'` and `ackState === false`; do not add a server-side guard (requires PLC query in the delete path, out of scope).

## Implications for Roadmap

The architecture research explicitly defines a four-phase build order based on data dependency. That order should be preserved in the roadmap.

### Phase 1: Driver — RelationId in MongoDB

**Rationale:** Everything downstream depends on `relationId` and `originDbName` being present in alarm documents. This phase has no external dependencies and can be verified independently by inspecting MongoDB documents before any UI work begins. Starting here validates the bit-extraction logic and the BSON type choice (Int64, not Int32) before any UI work begins.

**Delivers:** All new alarm events written to `s7plusAlarmEvents` contain `relationId` (BsonInt64) and `originDbName` (string, initially `""` until Phase 2 completes the map).

**Addresses:** `relationId` stored in MongoDB (table stakes), `originDbName` field presence in MongoDB (table stakes — value populated in Phase 2).

**Files changed:** `Common.cs` (new field), `AlarmThread.cs` BuildAlarmDocument (new parameters + two new BsonDocument fields).

**Avoids:** RelationId Int32 truncation pitfall — must use BsonInt64 from the start.

### Phase 2: Driver — Startup DB Name Map

**Rationale:** Depends on Phase 1 (the `RelationIdNameMap` field and the `BuildAlarmDocument` parameter must exist before map population is wired). Once Phase 2 completes, new alarm events carry a populated `originDbName`. Existing documents from Phase 1 remain `""` — acceptable; no back-fill required.

**Delivers:** `GetListOfDatablocks` (or targeted `ExploreRequest`) called in `ConnectionThread` before `AlarmThread.Start()`; `srv.RelationIdNameMap` populated; new alarm events show DB name in `originDbName` field.

**Addresses:** Startup DB name map (table stakes).

**Files changed:** `Program.cs` ConnectionThread (one call added), `AlarmThread.cs` or new `AlarmOrigin.cs` (new `BrowseRelationIdNames` method with defensive try-catch + empty-map fallback).

**Avoids:** Browse-on-wrong-connection pitfall (browse on `srv.connection`, not `alarmConn`); ExploreASAlarms-throws-on-empty pitfall (defensive `.FirstOrDefault()` + null guard).

### Phase 3: Backend — Delete Endpoint + _id Exposure

**Rationale:** Independent of Phases 1 and 2 at the code level; can be built in parallel. Must complete before Phase 4 (Vue needs the endpoint and the `_id` field in list responses). Testing the endpoint in isolation via curl or Postman before frontend wiring catches auth and filter-translation bugs early.

**Delivers:** `_id` returned in `listS7PlusAlarms` response; `POST /Invoke/auth/deleteS7PlusAlarms` endpoint working with both `ids` body (single-row) and `filter` body (bulk).

**Addresses:** `_id` in list response (table stakes), delete endpoint (table stakes).

**Files changed:** `server_realtime_auth/index.js` (one-line projection change + new route).

**Avoids:** Missing `authJwt.isAdmin` guard pitfall; filter-values-passed-as-literals pitfall (map "All" → omit the field from the MongoDB filter, never pass "All" as a literal `alarmState` value).

### Phase 4: Frontend — Delete Buttons + Origin Column

**Rationale:** Depends on Phases 1 and 2 (`originDbName` field in documents) and Phase 3 (`_id` in response + delete endpoint). All driver and backend work must complete before this phase. Vue changes are low-risk once the data is in place — they follow the established Ack button template slot pattern already present in the component.

**Delivers:** Full v1.2 feature set visible in the viewer: "Origin DB" column, per-row Delete button with optimistic row removal, "Delete Filtered (N)" toolbar button (disabled when zero filtered rows), confirmation dialog for Coming+unacked rows.

**Addresses:** Origin DB column (table stakes), per-row Delete button (table stakes), Delete Filtered button (table stakes), active-alarm delete warning (UX quality).

**Files changed:** `S7PlusAlarmsViewerPage.vue` (headers array, template slots, new ref/computed additions, two new async functions).

**Avoids:** Delete-race-with-refresh pitfall (optimistic removal on success); active-alarm-silent-delete pitfall (confirmation dialog for Coming+unacked rows); "Delete Filtered" button active when zero rows visible (`:disabled="filteredAlarms.length === 0"`).

### Phase Ordering Rationale

- Phase 1 before Phase 2: the `RelationIdNameMap` field and the `BuildAlarmDocument` signature change (Phase 1) must exist before the map population call (Phase 2) is wired; they can be a single PR or sequential commits.
- Phase 3 is code-independent from Phases 1/2 and can be worked in parallel, but all three must land before Phase 4.
- Phase 4 last: it is the only phase that requires integration-level validation across the full stack (driver → MongoDB → API → Vue).
- The four phases map cleanly to the four component layers (driver schema, driver browse, backend API, frontend UI), minimising cross-cutting changes within any single phase.

### Research Flags

Phases with well-documented patterns — research-phase not needed:

- **Phase 1 (RelationId in MongoDB):** Bit extraction and BSON type choice are confirmed from source; `BuildAlarmDocument` pattern is established and verified in the live codebase.
- **Phase 3 (Delete endpoint):** `deleteOne`/`deleteMany` API is stable and documented; auth pattern is a direct copy of the ack endpoint; filter-to-MongoDB mapping has clear rules derived from existing viewer filter logic.
- **Phase 4 (Frontend):** All Vue/Vuetify patterns (template slots, `:loading`, `ref`, `computed`) are established in the existing component; the Ack button is a direct template to follow.

Phase that merits a brief validation checkpoint before full implementation:

- **Phase 2 (Startup browse):** `BrowseRelationIdNames` calls `ExploreASAlarms` (or `GetListOfDatablocks`) on `srv.connection` at a point in `ConnectionThread` not yet exercised for alarm-origin purposes. A short spike to confirm the connection is in the correct state at that point — and to validate the `RelationId`-to-name mapping against a live PLCSIM instance — is recommended before committing the full implementation. This is a quick validation spike, not a full research cycle.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All technologies verified from `package.json`, `.csproj`, and source files; no speculative dependencies; version compatibility confirmed |
| Features | HIGH | Feature list derived from existing v1.1 codebase; MVP definition is explicit and bounded; all table-stakes features have direct implementation paths confirmed in source |
| Architecture | HIGH | Component boundaries and data flow traced through source; integration points verified; build order derived from explicit code dependencies, not inference |
| Pitfalls | HIGH (code) / MEDIUM (protocol) | Code-level pitfalls confirmed from source analysis; protocol-level claim (RelationId reassignment after TIA Portal download) is based on reverse-engineered driver knowledge and general ICS engineering — no public Siemens specification confirms this |

**Overall confidence:** HIGH

### Gaps to Address

- **RelationId stability after TIA Portal download (MEDIUM confidence):** The claim that a compile-and-download reassigns RelationIds without dropping the TCP connection is based on reverse-engineering and general ICS knowledge, not a public specification. During Phase 2 validation, test this scenario with PLCSIM Advanced: perform a TIA Portal download while the driver is running and observe whether the map becomes stale or whether the TCP connection drops (which would trigger an automatic re-browse on reconnect). If the connection drops, this becomes a non-issue.

- **ExploreASAlarms DB-name extraction path (confirmed but two valid options):** Research identifies two equivalent approaches — call `GetListOfDatablocks` as-is (recommended; simpler; already proven in `S7CommPlusGUIBrowser`) or add `ObjectVariableTypeName` to the AP explore request within a custom method. The `GetListOfDatablocks` second pass (reading `LID=1` for `db_block_ti_relid`) is unnecessary for origin mapping but harmless. Validate the mapping against a live PLCSIM instance in Phase 2 before committing to either approach.

- **`alarmConn` / `srv.connection` state at the point `BrowseRelationIdNames` is called:** Architecture confirms the two connections are separate TCP sockets so there is no PDU interleaving between them. The browse-before-AlarmThread-start sequencing has been reasoned from source but not explicitly tested. The Phase 2 validation spike should confirm this timing is safe in practice.

## Sources

### Primary (HIGH confidence — source code)

- `S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs` lines 1183–1314 — `GetListOfDatablocks` implementation, `DatablockInfo` struct
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/BrowseAlarms.cs` — `ExploreASAlarms`, `AlarmData.GetCpuAlarmId()`, `RelationId << 32` encoding confirmed
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsDai.cs` — `FromNotificationObject`, `CpuAlarmId` origin from `DAI_CPUAlarmID`
- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — `BuildAlarmDocument` pattern, alarm subscription loop
- `json-scada/src/S7CommPlusClient/Common.cs` line 93 — `AddressCache` pattern confirming per-connection in-memory dict convention; `[BsonIgnore]` usage
- `json-scada/src/S7CommPlusClient/Program.cs` — `ConnectionThread` ordering: `AlarmThread.Start()` then `BrowseAndCreateTags`
- `json-scada/src/server_realtime_auth/index.js` lines 350–407 — `listS7PlusAlarms` projection `{ _id: 0 }`, `ackS7PlusAlarm` pattern with `authJwt.isAdmin`
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — `pendingAcks` Set pattern, `filteredAlarms` computed, `fetchAlarms` loop, Ack button template slot
- `json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` — MongoDB.Driver 3.4.2 confirmed
- `json-scada/src/server_realtime_auth/package.json` — mongodb ^7.0.0 confirmed
- `S7CommPlusGUIBrowser/Form1.cs` lines 60–75 — `GetListOfDatablocks()` proven usage reference

### Secondary (MEDIUM confidence — reverse-engineered protocol / general ICS)

- S7CommPlus reverse-engineered driver (Thomas Wiens) — `RelationId` is a compile-time PLC object-tree identifier, not a user-visible stable name
- General ICS engineering practice — TIA Portal compile-and-download reassigns internal PLC object IDs; no public Siemens specification confirms the exact TCP session behaviour during download

### Tertiary (documentation references)

- MongoDB docs: `deleteMany` — `deletedCount === 0` is valid success (idempotent delete)
- MongoDB docs: race conditions in concurrent operations — basis for bulk delete filter drift pitfall documentation

---
*Research completed: 2026-03-23*
*Ready for roadmap: yes*

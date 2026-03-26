# Project Research Summary

**Project:** v1.4 Tag Tree Browser (DatablockBrowser + TagTreeBrowser)
**Domain:** TIA Portal-equivalent hierarchical PLC tag browser — S7CommPlus / Vue 3 / MongoDB SCADA
**Researched:** 2026-03-26
**Confidence:** HIGH

## Executive Summary

v1.4 adds a TIA Portal-style tag browser to the existing json-scada S7CommPlus SCADA stack. The milestone splits into two new AdminUI pages: a `DatablockBrowserPage` listing all datablocks on the connected PLC, and a `TagTreeBrowserPage` providing lazy, on-demand expansion of each datablock's type hierarchy. No new npm packages are required — `v-treeview` ships with the already-installed Vuetify 3.10.12, and `router.resolve() + window.open()` handles new-tab navigation using the existing Vue Router 4. The backend pattern is equally conservative: the C# driver writes the datablock list to a new MongoDB collection at startup, and per-node type-info is fetched on demand via the existing `commandsQueue` change-stream mechanism, with results cached in a second new collection. Node.js reads only from MongoDB, keeping the event loop non-blocking.

The recommended architecture separates all PLC browse traffic onto a **third, dedicated `S7CommPlusConnection`**, following the v1.2 precedent of an isolated `alarmConn`. This is not optional — reusing either the tag or alarm connection for browse PDUs causes PDU interleaving that can silently corrupt tag values or terminate the connection. The full feature set follows a strict dependency chain: driver startup writes must precede the API endpoint, which must precede the Vue pages, which must precede the alarms-viewer integration. Sequencing phases along this chain is the primary constraint on roadmap structure.

The two highest risks for this milestone are (1) the PDU-interleaving problem if the browse connection is not isolated, and (2) a cluster of Vuetify `v-treeview` bugs that can cause infinite loops (duplicate node names), duplicate children (rapid expand-collapse), and performance degradation on large PLCs. Both risks have concrete, low-cost mitigations that must be baked in at the component-design level before writing the first line of Vue code.

---

## Key Findings

### Recommended Stack

The core stack is entirely unchanged from v1.3. Vuetify 3.10.12 includes `v-treeview` with a `load-children` prop that supports async lazy loading — this is the correct component for the TagTreeBrowser. The DatablockBrowser flat list should use `v-list` with `v-virtual-scroll` rather than `v-treeview` to avoid a confirmed Vuetify 3 performance regression on flat lists. New-window navigation uses `router.resolve().href` + `window.open()` against the existing `createWebHashHistory` router; no URL construction needs to be manual.

**Core technologies (all already installed):**
- `v-treeview` (Vuetify 3.10.12, built-in): lazy tree rendering via `load-children` async prop — no new packages
- `v-list` + `v-virtual-scroll` (Vuetify 3.10.12, built-in): flat DatablockBrowser list — avoids v-treeview perf regression on large lists
- `vue-router` 4.4.x: `router.resolve().href` + `window.open()` for new-tab launch — resilient to hash-history base-path changes
- MongoDB `commandsQueue` (existing): async dispatch of browse requests to C# driver — same pattern as alarm ack
- Two new MongoDB collections (`s7plusDatablocks`, `s7plusTypeInfoCache`): decouple PLC latency from Node.js response time

See `.planning/research/STACK.md` for full version evidence, code snippets, and alternatives considered.

### Expected Features

**Must have (table stakes — all P1):**
- `s7plusDatablocks` MongoDB collection populated by C# driver at startup — all browser features depend on this
- `GET /Invoke/auth/listS7PlusDatablocks` endpoint — identical pattern to `listS7PlusAlarms`
- `DatablockBrowserPage.vue` with DB name, DB number, and Browse button per row
- Connection selector in DatablockBrowserPage (multi-PLC deployments)
- AdminUI nav drawer entry for DatablockBrowser
- `TagTreeBrowserPage.vue` with lazy tree expansion via `load-children` and `getTypeInfoByRelId`
- Variable name and data type columns in TagTreeBrowser (Softdatatype from `PVartypeListElement`)
- 1-dimensional array member expansion with `varName[0]` index notation (GUIBrowser Form1.cs is reference)
- Live value column from `realtimeData` for configured tags; "—" for unconfigured tags; 5 s auto-refresh
- `S7PlusAlarmsViewerPage.vue`: `originDbName` cell becomes a clickable link opening TagTreeBrowser in new tab
- TagTreeBrowser accepts `?db=<name>` query parameter; resolves `ti_relid` on page load

**Should have (differentiators — add after P1 validation):**
- Node type icons per Softdatatype category (MDI icons — matches TIA Portal visual language)
- Client-side datablock name filter in DatablockBrowser (computed filter, zero backend change)
- Last-updated timestamp next to live value (uses existing `updatedAt` field in `realtimeData`)

**Defer (v2+):**
- Write-back / force tag values from browser (requires SBO logic, access control, command queue extension)
- `activeTagRequests` insertion from browser (on-demand polling trigger for unconfigured tags)
- Multi-dimensional array expansion (1-dim covers the majority of real PLC programs)
- Export tag list to CSV/Excel

See `.planning/research/FEATURES.md` for full dependency graph and per-feature implementation notes.

### Architecture Approach

The architecture is a four-layer pipeline: PLC -> C# driver (reads and caches to MongoDB) -> Node.js API (reads MongoDB only) -> Vue UI (fetches API, polls every 5 s). The key design decision is **on-demand browse via command queue** rather than startup pre-caching: the driver does not fetch type info at boot (which would delay alarm subscription), but instead responds to `commandsQueue` inserts with ASDU `"s7plus-browse-request"`. Results are persisted to `s7plusTypeInfoCache` and are instant on second access. The Vue `load-children` callback issues a POST to request type info, then polls `getS7PlusTypeInfo` every 500 ms (max 10 s, 20 retries) until `status === "ok"`.

**Major components:**
1. `ConnectionThread` (Program.cs, MODIFIED) — writes `s7plusDatablocks` snapshot on each successful PLC connect
2. `ProcessMongoCmd` (MongoCommands.cs, MODIFIED) — adds `s7plus-browse-request` ASDU branch; serializes PObject to `s7plusTypeInfoCache`
3. Dedicated browse `S7CommPlusConnection` (NEW) — isolated third connection; never shared with tag or alarm PDU traffic
4. Three new endpoints in `server_realtime_auth/index.js` (NEW): `listS7PlusDatablocks`, `getS7PlusTypeInfo`, `requestS7PlusTypeInfo`
5. `DatablockBrowserPage.vue` (NEW) — flat list with connection selector and row-level Browse navigation
6. `TagTreeBrowserPage.vue` (NEW) — lazy tree with dedup guards, live value polling, URL-param bootstrap
7. `S7PlusAlarmsViewerPage.vue` (MODIFIED) — `originDbName` as `window.open` link

See `.planning/research/ARCHITECTURE.md` for full system diagram, MongoDB schemas, data flow sequences, and build order.

### Critical Pitfalls

1. **PDU interleaving on shared S7CommPlusConnection** — Browse PDUs mixed with tag or alarm responses corrupt data or terminate the connection. Mitigation: open a dedicated third `S7CommPlusConnection` for all browse calls, following the `alarmConn` isolation pattern from v1.2. Must be decided before any browse endpoint is written.

2. **v-treeview infinite loop on duplicate node names** — Vuetify 3.7.0+ freezes the browser tab when a node and its parent share the same display name (confirmed Vuetify issue #20389; common TIA Portal pattern). Mitigation: always set `:item-value` to a unique key derived from RelId + parent path, never from the display name.

3. **`load-children` fires on node select, not only expand; race on rapid expand-collapse** — Both cause duplicate tree children. Mitigation: maintain a `loadedRelIds` Set and a `loadingRelIds` Set; gate all HTTP requests through both sets.

4. **JWT not available in new browser window** — Auth token in `sessionStorage` is not shared across tabs; `window.open` opens an unauthenticated tab that redirects to login and loses the `?db=` param. Mitigation: verify token is in `localStorage`; encode `db` param through the login redirect cycle.

5. **Blocking browse calls hang Node.js event loop** — `getTypeInfoByRelId` is a synchronous PLC call (up to 5 s timeout). Node.js must never call it directly. Mitigation: all browse results flow through the MongoDB command-queue cache; Node.js reads cache only, never touches the PLC connection.

See `.planning/research/PITFALLS.md` for all 14 pitfalls with phase-specific warning tables.

---

## Implications for Roadmap

Based on the dependency chain discovered in research, the natural structure is eight sequential steps grouped into four logical phases.

### Phase 1: Driver — Datablock List Persistence
**Rationale:** Every other feature depends on `s7plusDatablocks` existing in MongoDB. This is the mandatory foundation; nothing else can be built or tested without it.
**Delivers:** `s7plusDatablocks` collection populated at driver startup with `db_name`, `db_number`, `db_block_relid`, `db_block_ti_relid`, `connectionNumber`; `ReplaceOneAsync` upsert keyed on `{connectionNumber, db_name}`.
**Addresses:** P1 feature "s7plusDatablocks MongoDB collection"
**Avoids:** Pitfall 2 (Node.js event loop block) — data pre-cached at driver startup, not at HTTP request time; Pitfall 13 (inline collection name) — define named constant before writing any collection access code

### Phase 2: API — listS7PlusDatablocks Endpoint
**Rationale:** Low-risk, pattern-match step (identical to `listS7PlusAlarms`). Enables end-to-end API verification before building the Vue layer.
**Delivers:** `GET /Invoke/auth/listS7PlusDatablocks?connectionNumber=N` returning sorted datablock list; `[authJwt.isAdmin]` guard consistent with other S7Plus endpoints.
**Addresses:** P1 feature "listS7PlusDatablocks API endpoint"

### Phase 3: Vue — DatablockBrowserPage + Router
**Rationale:** With the API available, the flat DatablockBrowser can be built and demo'd before tackling the more complex tree component. TagTreeBrowserPage starts as a stub showing only the `db` query param.
**Delivers:** `DatablockBrowserPage.vue` (v-list + v-virtual-scroll, connection selector, client-side name filter), two new routes (`/s7plus-datablocks`, `/s7plus-tag-tree`), AdminUI nav entry, DashboardPage shortcut, TagTreeBrowserPage stub.
**Uses:** `v-list` + `v-virtual-scroll` for flat list (avoids Pitfall 4 — v-treeview perf regression on large flat lists)
**Addresses:** P1 features: DatablockBrowserPage, nav entry, Browse action; P2 feature: datablock name filter

### Phase 4: Driver — On-Demand Browse via Command Queue
**Rationale:** The TagTreeBrowser's lazy expansion requires the driver to respond to browse commands and write type info to `s7plusTypeInfoCache`. This driver-side work must be complete before Vue expansion logic can be wired up. Can be developed in parallel with Phase 3 (no shared files).
**Delivers:** Dedicated third `S7CommPlusConnection` for browse; `ProcessMongoCmd` ASDU branch for `s7plus-browse-request`; PObject serialization (varnames + vartypes + relIds) to `s7plusTypeInfoCache` with upsert.
**Addresses:** P1 feature "Type-info browse backend (C# side)"
**Avoids:** Pitfall 1 (PDU interleaving) — dedicated connection is the solution; Pitfall 12 (blocking thread) — wrap browse call in `Task.Run` with `CancellationToken`; Pitfall 7 (null PObject crash) — null-guard copied from Form1.cs line 136; Pitfall 14 (GetListOfDatablocks timeout) — document `timeoutMs` config for large PLCs

### Phase 5: API — getS7PlusTypeInfo + requestS7PlusTypeInfo Endpoints
**Rationale:** Thin endpoints over the now-populated `s7plusTypeInfoCache` and `commandsQueue`. Idempotency check prevents duplicate PLC requests.
**Delivers:** `GET /Invoke/auth/getS7PlusTypeInfo?tiRelId=X`, `POST /Invoke/auth/requestS7PlusTypeInfo`; compound `{ protocolSourceObjectAddress: 1, protocolConnectionNumber: 1 }` index on `realtimeData` at startup.
**Addresses:** P1 feature "Type-info browse backend (Node.js side)"
**Avoids:** Pitfall 8 (realtimeData full collection scan) — compound index created in this phase

### Phase 6: Vue — TagTreeBrowserPage Lazy Expand
**Rationale:** With all backend infrastructure in place, the full tree component is built with correct deduplication guards from day one.
**Delivers:** `TagTreeBrowserPage.vue` with `v-treeview`, `load-children` async handler, `loadedRelIds`/`loadingRelIds` dedup guards, unique `:item-value` keys (RelId + parent path), 1-dim array expansion, UDT child placeholders, 500 ms poll loop with 10 s timeout.
**Addresses:** P1 features: TagTreeBrowserPage lazy expand, 1-dim array expansion
**Avoids:** Pitfall 3 (load-children fires on select), Pitfall 5 (duplicate node name infinite loop), Pitfall 10 (rapid expand-collapse race)

### Phase 7: Vue + API — Live Values Column
**Rationale:** Self-contained addition to the tree page; depends on the tree rendering correctly first.
**Delivers:** Batch `getRealtimeValues` endpoint (or reuse existing pattern); live value column in TagTreeBrowserPage showing `valueString` from `realtimeData` for leaf nodes; 5 s auto-refresh for visible leaf nodes; "—" for unconfigured tags.
**Addresses:** P1 feature "Live value column"
**Avoids:** Pitfall 5 variant — only query realtimeData for leaf nodes (hasRelation=false) that are currently expanded, not the entire tree

### Phase 8: Integration — Alarms Viewer Link + Auth in New Window
**Rationale:** End-to-end integration of the two milestone goals. Auth-in-new-window must be verified before wiring up `window.open`.
**Delivers:** `originDbName` as clickable chip/link in S7PlusAlarmsViewerPage; `encodeURIComponent` on DB name; auth token storage in `localStorage` verified; `db` param preserved through any login redirect.
**Addresses:** P1 feature "originDbName click opens TagTreeBrowser"
**Avoids:** Pitfall 6 (JWT not in new tab), Pitfall 9 (navigation guard loses db param), Pitfall 11 (special chars in DB name URL)

### Phase Ordering Rationale

- Phases 1-2 are strictly sequential (driver writes MongoDB -> API reads MongoDB -> Vue fetches API).
- Phase 3 can deliver a working DatablockBrowser with a stub TagTreeBrowserPage, providing an early integration checkpoint and demo milestone.
- Phase 4 (driver browse) can be developed in parallel with Phase 3 (Vue DatablockBrowser) — no file overlap.
- Phases 5-6 are tightly coupled (API endpoint schema and Vue `load-children` tree node schema must agree with the MongoDB document structure defined in Phase 4).
- Phase 7 (live values) is additive and can be treated as an optional sub-milestone if time is short.
- Phase 8 (alarms integration) is last because it depends on TagTreeBrowserPage being fully functional.

### Research Flags

Phases likely needing deeper research or a validation spike during planning:

- **Phase 4 (Driver browse connection):** The dedicated third `S7CommPlusConnection` initialization sequence (connect timing, error recovery, reconnect behavior when tag/alarm connections are also reconnecting) should be validated against `AlarmThread.cs` as the reference implementation before coding begins. The pattern is architecturally clear but the exact C# lifecycle is not fully spelled out in research.
- **Phase 7 (Live values address resolution):** Mapping tree node paths to `realtimeData.protocolSourceObjectAddress` strings requires the `escapeTiaString` path-composition logic from GUIBrowser Form1.cs lines 334-348 to match the address format the driver actually stores. This is a MEDIUM-confidence area — verify empirically against a real `realtimeData` document before writing the lookup logic.

Phases with standard patterns (skip research-phase):
- **Phase 1:** Direct extension of existing `GetListOfDatablocks` + `ReplaceOneAsync` pattern in Program.cs — HIGH confidence.
- **Phase 2:** Pattern-match with `listS7PlusAlarms` — HIGH confidence.
- **Phase 3:** Standard Vuetify v-list + Vue Router patterns — HIGH confidence.
- **Phase 5:** Standard Express endpoint + `insertOne` into commandsQueue — HIGH confidence.
- **Phase 8:** Vue Router `router.resolve()` + `window.open()` — HIGH confidence.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All versions confirmed from installed node_modules and package.json; v-treeview availability verified against Vuetify issue history and official docs; router pattern confirmed from project source code |
| Features | HIGH | Grounded in live codebase (GUIBrowser Form1.cs, S7CommPlusConnection.cs, server_realtime_auth/index.js); UX conventions validated against Ignition, OPC UA, and TIA Portal WinCC patterns |
| Architecture | HIGH | Based on direct code inspection of all affected files; command-queue browse pattern verified against existing alarm ack implementation in MongoCommands.cs |
| Pitfalls | HIGH | All 14 pitfalls grounded in codebase inspection and/or confirmed Vuetify GitHub issues with specific issue numbers and affected version ranges |

**Overall confidence:** HIGH

### Gaps to Address

- **`db_block_ti_relid` not in alarm documents:** Alarm documents store `dbNumber` and `originDbName`, not `db_block_ti_relid`. TagTreeBrowser opened from an alarm must call `listS7PlusDatablocks` on mount to resolve `ti_relid` by name match. This is a known, accepted design choice — implement explicitly in Phase 8.

- **Live value address format match:** `realtimeData.protocolSourceObjectAddress` may use AccessSequence hex strings (e.g. `8a0e0001.1.2.3`) rather than TIA Portal symbolic strings (e.g. `"AlarmData_DB".AlarmSlot[0].Active`). The exact format for already-configured tags must be confirmed against a real `realtimeData` document before Phase 7 implementation. The `escapeTiaString` logic in GUIBrowser Form1.cs is the reference candidate.

- **Browse connection reconnect lifecycle:** How the dedicated browse `S7CommPlusConnection` should behave during PLC reconnect events is not detailed in research. Validate the reconnect sequence against `AlarmThread.cs` before Phase 4 coding begins.

- **Large array display cap:** 1-dim arrays with many elements (e.g. 500-element arrays of UDTs) could generate hundreds of child nodes in a single `load-children` call. A display cap (e.g. first 100 elements + "Show more") should be decided before Phase 6 implementation begins.

---

## Sources

### Primary (HIGH confidence — direct source code)
- `S7CommPlusGUIBrowser/Form1.cs` — lazy browse, array handling, UDT recursion, SetImageKey() type mapping, null-guard (line 136)
- `S7CommPlusDriver/S7CommPlusConnection.cs` — `DatablockInfo` struct, `GetListOfDatablocks`, `getTypeInfoByRelId`, PDU queue / mutex / timeout constants
- `S7CommPlusClient/Program.cs` — ConnectionThread structure, existing `GetListOfDatablocks` call, collection name constants
- `S7CommPlusClient/MongoCommands.cs` — ASDU dispatch pattern; alarm ack as template for browse command handler
- `S7CommPlusClient/AlarmThread.cs` — `alarmConn` isolation pattern; dedicated connection reference
- `S7CommPlusClient/Common.cs` — `S7CP_connection.timeoutMs` default 5000 ms
- `server_realtime_auth/index.js` — existing S7Plus alarm endpoints; `authJwt.isAdmin` guard; commandsQueue pattern; `createIndex` idempotent pattern (Phase 10)
- `AdminUI/src/router/index.js` — `createWebHashHistory` confirmed; existing route registration pattern
- `AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — existing viewer structure, filter patterns, auto-refresh
- `AdminUI/src/components/DashboardPage.vue` — shortcut card pattern
- `AdminUI/package.json` — Vuetify `"3.10"`, Vue `^3.4.31`, vue-router `^4.4.0` confirmed
- `.planning/PROJECT.md` — v1.4 target features, KEY DECISIONS, constraints, validated history

### Secondary (MEDIUM confidence)
- Vuetify official docs (vuetifyjs.com) — `v-treeview` component, `load-children` prop async behavior
- Vue Router official docs (router.vuejs.org) — `router.resolve()` API
- Vuetify GitHub issues #19919, #20389, #21387, #5543, #8791, #20450 — confirmed bugs with affected version ranges
- Ignition Tag Browser documentation — filter, live value, data type column UX conventions
- OPC UA address space browser patterns (UaExpert) — lazy expand, node caching, expand-on-click conventions

### Tertiary (LOW confidence — needs validation)
- devicewise.com Siemens S7 driver docs — `GetListOfDatablocks` timeout recommendation for PLCs with >300 datablocks
- Pinia cross-tab isolation community docs — `localStorage` vs `sessionStorage` partitioning (verify against AdminUI auth store implementation)

---

*Research completed: 2026-03-26*
*Ready for roadmap: yes*

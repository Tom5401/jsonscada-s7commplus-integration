# Roadmap: S7CommPlus Alarm Subscriptions for json-scada

## Milestones

- ✅ **v1.0 — S7CommPlus Alarm Subscriptions PoC** — Phase 1 (shipped 2026-03-18)
- ✅ **v1.1 — Alarm Management & Viewer** — Phases 2–4 (shipped 2026-03-23)
- ✅ **v1.2 — Alarm Origin & Cleanup** — Phases 5–8 (shipped 2026-03-24)
- ✅ **v1.3 — Alarm Viewer Enhancements & Priority** — Phases 9–11 (shipped 2026-03-25)
- ✅ **v1.4 — Tag Tree Browser** — Phases 12–15 (shipped 2026-03-27)
- 🚧 **v1.5 — TagTreeBrowser Overhaul** — Phases 16–19 (in progress)

## Phases

<details>
<summary>✅ v1.0 — S7CommPlus Alarm Subscriptions PoC (Phase 1) — SHIPPED 2026-03-18</summary>

- [x] Phase 1: End-to-End Alarm Pipeline (2/2 plans) — completed 2026-03-18

Full details: [milestones/v1.0-ROADMAP.md](milestones/v1.0-ROADMAP.md)

</details>

<details>
<summary>✅ v1.1 — Alarm Management & Viewer (Phases 2–4) — SHIPPED 2026-03-23</summary>

- [x] Phase 2: Driver Fixes (2/2 plans) — completed 2026-03-18
- [x] Phase 3: Read-Only Alarm Viewer (2/2 plans) — completed 2026-03-19
- [x] Phase 4: Ack Write-Back (2/2 plans) — completed 2026-03-19

Full details: [milestones/v1.1-ROADMAP.md](milestones/v1.1-ROADMAP.md)

</details>

<details>
<summary>✅ v1.2 — Alarm Origin & Cleanup (Phases 5–8) — SHIPPED 2026-03-24</summary>

- [x] Phase 5: Driver — RelationId Fields (1/1 plans) — completed 2026-03-23
- [x] Phase 6: Driver — Startup DB Name Map (1/1 plans) — completed 2026-03-23
- [x] Phase 7: Backend — Delete Endpoint + _id Exposure (1/1 plans) — completed 2026-03-24
- [x] Phase 8: Frontend — Delete Buttons + Origin Columns (1/1 plans) — completed 2026-03-24

Full details: [milestones/v1.2-ROADMAP.md](milestones/v1.2-ROADMAP.md)

</details>

<details>
<summary>✅ v1.3 — Alarm Viewer Enhancements & Priority (Phases 9–11) — SHIPPED 2026-03-25</summary>

- [x] Phase 9: Driver Enrichment (1/1 plans) — completed 2026-03-25
- [x] Phase 10: API Cap Removal (1/1 plans) — completed 2026-03-25
- [x] Phase 11: Vue UI Enhancements (2/2 plans) — completed 2026-03-25

Full details: [milestones/v1.3-ROADMAP.md](milestones/v1.3-ROADMAP.md)

</details>

<details>
<summary>✅ v1.4 — Tag Tree Browser (Phases 12–15) — SHIPPED 2026-03-27</summary>

- [x] Phase 12: Driver — Datablock Persistence (1/1 plans) — completed 2026-03-26
- [x] Phase 13: Backend API — Datablocks & Tag Endpoints (1/1 plans) — completed 2026-03-26
- [x] Phase 14: DatablockBrowser (1/1 plans) — completed 2026-03-26
- [x] Phase 15: TagTreeBrowser & Integration (1/1 plans) — completed 2026-03-27

Full details: [milestones/v1.4-ROADMAP.md](milestones/v1.4-ROADMAP.md)

</details>

### v1.5 TagTreeBrowser Overhaul (In Progress)

**Milestone Goal:** Transform TagTreeBrowser from a proof-of-concept into a production-ready tool that handles 100k+ tag databases with lazy loading at every tree level, real boolean value display, value writing from the tree, and non-datablock tag support.

#### Phases

- [x] **Phase 16: Backend — `listS7PlusChildNodes` Endpoint + Index** — New auth-guarded endpoint returning direct children of a path; compound index for fast child queries (completed 2026-03-30)
- [x] **Phase 17: Lazy Tree Loading + Boolean Display** — Rewrite TagTreeBrowserPage.vue around Vuetify `load-children`; scope value refresh to expanded leaves; show TRUE/FALSE for digital tags (completed 2026-03-30)
- [ ] **Phase 18: Value Write Dialog** — New `PushValueDialog.vue` component; write button on leaf nodes; routes through existing OPC WriteRequest handler
- [ ] **Phase 19: Non-Datablock Tags** — Append virtual area rows (IArea, QArea, MArea, S7Timers, S7Counters) to DatablockBrowserPage; browsable via existing lazy endpoint

## Phase Details

### Phase 16: Backend — `listS7PlusChildNodes` Endpoint + Index
**Goal**: A new admin-guarded HTTP endpoint returns the direct children of a given `protocolSourceBrowsePath` for a connection, backed by a compound index — enabling every other v1.5 feature to fetch exactly one level of the tag tree per expand
**Depends on**: Nothing (first v1.5 phase; built on existing realtimeData collection)
**Requirements**: PERF-01, PERF-02
**Success Criteria** (what must be TRUE):
  1. `GET /Invoke/auth/listS7PlusChildNodes?connectionNumber=N&path=P` returns only direct children whose `protocolSourceBrowsePath` equals `P` exactly — not subtree matches
  2. Each returned document includes a `hasChildren` boolean derived from a secondary `distinct` query, enabling the frontend to distinguish folder from leaf nodes
  3. A compound index `{protocolSourceConnectionNumber: 1, protocolSourceBrowsePath: 1}` is created on `realtimeData` at server startup idempotently (no crash on duplicate)
  4. The endpoint rejects unauthenticated requests consistent with the existing `[authJwt.isAdmin]` guard pattern
  5. A direct MongoDB query on a live connection confirms that area tags (MArea, QArea) have `protocolSourceBrowsePath` values matching the pattern the endpoint will query
**Plans:** 1/1 plans complete
Plans:
- [x] 16-01-PLAN.md — Add listS7PlusChildNodes endpoint and compound index to index.js

### Phase 17: Lazy Tree Loading + Boolean Display
**Goal**: TagTreeBrowserPage loads the root level on open and fetches one additional level per expand via Vuetify `load-children`, scopes the 5-second value refresh to only currently-expanded leaf tags, and displays digital tags as TRUE/FALSE instead of 0/1
**Depends on**: Phase 16
**Requirements**: PERF-03, PERF-04, DISP-01
**Success Criteria** (what must be TRUE):
  1. Opening TagTreeBrowserPage with `?db=<name>&connectionNumber=<N>` loads only the root-level children of that DB — not the full tag subtree
  2. Expanding a tree node triggers exactly one `listS7PlusChildNodes` call for that node's path and renders its children; the expand fires only once per node (subsequent opens use cached children)
  3. The 5-second value refresh calls `touchS7PlusActiveTagRequests` only for tags whose tree nodes are currently expanded and visible, not all tags in the DB
  4. Leaf tags with `protocolSourceDataType` of boolean/digital display `TRUE` or `FALSE` in the value column; non-digital tags are unaffected
  5. Deeply nested datablocks (3+ levels, e.g. UDT structs) expand correctly at every level — no silently unreachable nodes
**Plans:** 1/1 plans complete
Plans:
- [x] 17-01-PLAN.md — Rewrite TagTreeBrowserPage.vue with load-children, scoped refresh, and formatLeafValue

### Phase 18: Value Write Dialog
**Goal**: Operators can push a new value to any writable PLC tag directly from a TagTreeBrowser leaf node via a write dialog that routes through the existing OPC WriteRequest infrastructure
**Depends on**: Phase 17
**Requirements**: WRITE-01, WRITE-02
**Success Criteria** (what must be TRUE):
  1. A write button appears on leaf nodes where `commandOfSupervised != 0`; nodes without a paired command point show no write button
  2. Clicking the write button opens a `PushValueDialog.vue` modal pre-populated with the tag's name, current value, and data type
  3. The dialog renders a `v-select` (TRUE/FALSE) for digital tags and a `v-text-field` for analog/string tags
  4. Submitting the dialog posts an OPC WriteRequest (ServiceId 676) to `/Invoke/` — the existing handler enforces `canSendCommands`, creates a SOE log entry, and records to `userActions`
  5. A successful write closes the dialog and shows a brief confirmation; a failed write shows the error message without closing
**Plans:** 0/1 plans complete
Plans:
- [ ] 18-01-PLAN.md — Create PushValueDialog.vue and wire write button to leaf nodes in TagTreeBrowserPage.vue

### Phase 19: Non-Datablock Tags
**Goal**: MArea, QArea, IArea, S7Timers, and S7Counters appear as browsable entries alongside datablocks in DatablockBrowserPage, openable in TagTreeBrowserPage via the same lazy-expand mechanism
**Depends on**: Phase 17 (lazy endpoint handles area paths with no special-casing)
**Requirements**: AREA-01
**Success Criteria** (what must be TRUE):
  1. DatablockBrowserPage appends 5 virtual rows (IArea, QArea, MArea, S7Timers, S7Counters) after the real datablock list; virtual rows are visually consistent with datablock rows
  2. Clicking "Browse Tags" on a virtual area row opens TagTreeBrowserPage in a new tab with `?db=<areaName>&connectionNumber=<N>`
  3. TagTreeBrowserPage successfully lazy-loads the root children for an area path using the same `listS7PlusChildNodes` endpoint (no area-specific code path required)
  4. If an area has no tags in `realtimeData` for this connection, the tree renders empty (no crash, no infinite spinner)
**Plans:** 0/1 plans complete
Plans:
- [ ] 19-01-PLAN.md — Add virtual area rows to DatablockBrowserPage.vue

## Progress

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. End-to-End Alarm Pipeline | v1.0 | 2/2 | Complete | 2026-03-18 |
| 2. Driver Fixes | v1.1 | 2/2 | Complete | 2026-03-18 |
| 3. Read-Only Alarm Viewer | v1.1 | 2/2 | Complete | 2026-03-19 |
| 4. Ack Write-Back | v1.1 | 2/2 | Complete | 2026-03-19 |
| 5. Driver — RelationId Fields | v1.2 | 1/1 | Complete | 2026-03-23 |
| 6. Driver — Startup DB Name Map | v1.2 | 1/1 | Complete | 2026-03-23 |
| 7. Backend — Delete Endpoint + _id Exposure | v1.2 | 1/1 | Complete | 2026-03-24 |
| 8. Frontend — Delete Buttons + Origin Columns | v1.2 | 1/1 | Complete | 2026-03-24 |
| 9. Driver Enrichment | v1.3 | 1/1 | Complete | 2026-03-25 |
| 10. API Cap Removal | v1.3 | 1/1 | Complete | 2026-03-25 |
| 11. Vue UI Enhancements | v1.3 | 2/2 | Complete | 2026-03-25 |
| 12. Driver — Datablock Persistence | v1.4 | 1/1 | Complete | 2026-03-26 |
| 13. Backend API — Datablocks & Tag Endpoints | v1.4 | 1/1 | Complete | 2026-03-26 |
| 14. DatablockBrowser | v1.4 | 1/1 | Complete | 2026-03-26 |
| 15. TagTreeBrowser & Integration | v1.4 | 1/1 | Complete | 2026-03-27 |
| 16. Backend — listS7PlusChildNodes Endpoint + Index | v1.5 | 1/1 | Complete    | 2026-03-30 |
| 17. Lazy Tree Loading + Boolean Display | v1.5 | 1/1 | Complete   | 2026-03-30 |
| 18. Value Write Dialog | v1.5 | 0/1 | Not started | — |
| 19. Non-Datablock Tags | v1.5 | 0/1 | Not started | — |

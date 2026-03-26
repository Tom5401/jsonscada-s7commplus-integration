# Roadmap: S7CommPlus Alarm Subscriptions for json-scada

## Milestones

- ✅ **v1.0 — S7CommPlus Alarm Subscriptions PoC** — Phase 1 (shipped 2026-03-18)
- ✅ **v1.1 — Alarm Management & Viewer** — Phases 2–4 (shipped 2026-03-23)
- ✅ **v1.2 — Alarm Origin & Cleanup** — Phases 5–8 (shipped 2026-03-24)
- ✅ **v1.3 — Alarm Viewer Enhancements & Priority** — Phases 9–11 (shipped 2026-03-25)
- 🚧 **v1.4 — Tag Tree Browser** — Phases 12–15 (in progress)

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

### 🚧 v1.4 Tag Tree Browser (In Progress)

**Milestone Goal:** Mirror TIA Portal's hierarchical tag browsing in AdminUI — operators can navigate datablock structure, see live values for configured tags, and open a tag tree directly from the alarms viewer.

#### Phases

- [x] **Phase 12: Driver — Datablock Persistence** - C# driver writes the full PLC datablock list to MongoDB at startup (completed 2026-03-26)
- [ ] **Phase 13: Backend API — Datablocks & Tag Endpoints** - Three new HTTP endpoints expose datablock list, realtimeData tags, and activeTagRequests upsert
- [ ] **Phase 14: DatablockBrowser** - New AdminUI page listing all PLC datablocks with connection selector and row-level navigation to TagTreeBrowser
- [ ] **Phase 15: TagTreeBrowser & Integration** - Lazy tag tree with live values, URL-param bootstrap, and alarms viewer link integration

## Phase Details

### Phase 12: Driver — Datablock Persistence
**Goal**: The C# driver populates a dedicated MongoDB collection with the full PLC datablock list at startup, making it available to all downstream features
**Depends on**: Nothing (first v1.4 phase; extends existing GetListOfDatablocks startup call)
**Requirements**: DRIVER-03
**Success Criteria** (what must be TRUE):
  1. After driver startup, the `s7plusDatablocks` MongoDB collection contains one document per datablock on the connected PLC
  2. Each document includes `db_name`, `db_number`, `db_block_relid`, `db_block_ti_relid`, and `connectionNumber`
  3. Restarting the driver upserts (not duplicates) the datablock documents — keyed on `{connectionNumber, db_name}`
**Plans:** 1/1 plans complete
Plans:
- [x] 12-01-PLAN.md — Add DatablocksCollectionName, EnsureDatablockIndexes, and UpsertDatablocks to S7CommPlusClient

### Phase 13: Backend API — Datablocks & Tag Endpoints
**Goal**: Three new admin-guarded HTTP endpoints expose the datablock list, realtimeData tags filtered by DB name, and activeTagRequests upsert — all as thin MongoDB reads/writes with no PLC calls
**Depends on**: Phase 12
**Requirements**: API-03, API-04, API-05
**Success Criteria** (what must be TRUE):
  1. `GET /Invoke/auth/listS7PlusDatablocks?connectionNumber=N` returns all datablock documents for that connection sorted by `db_name`
  2. `GET /Invoke/auth/listS7PlusTagsForDb?connectionNumber=N&dbName=X` returns all `realtimeData` documents whose `protocolSourceObjectAddress` begins with `"X"` (quoted DB name prefix)
  3. `POST /Invoke/auth/touchS7PlusActiveTagRequests` accepts an array of `{connectionNumber, protocolSourceObjectAddress}` pairs and upserts them into `activeTagRequests` with a refreshed TTL timestamp
  4. All three endpoints reject unauthenticated requests consistent with the existing `[authJwt.isAdmin]` guard pattern
**Plans**: TBD

### Phase 14: DatablockBrowser
**Goal**: Operators can navigate to a new AdminUI page that lists all PLC datablocks with a connection filter, and open the TagTreeBrowser for any selected datablock in a new browser tab
**Depends on**: Phase 13
**Requirements**: DBBROWSER-01, DBBROWSER-02, DBBROWSER-03
**Success Criteria** (what must be TRUE):
  1. A "Datablock Browser" entry appears in the AdminUI navigation menu and opens `DatablockBrowserPage`
  2. The page displays all datablocks for the selected connection, showing `db_name` and `db_number` per row
  3. A connection selector dropdown lets the operator filter the list to a specific PLC connection
  4. Clicking a datablock row opens `TagTreeBrowserPage` in a new browser tab, pre-scoped to that datablock and connection
**Plans**: TBD
**UI hint**: yes

### Phase 15: TagTreeBrowser & Integration
**Goal**: Operators can lazy-expand any datablock's tag hierarchy with live values auto-refreshing every 5 seconds, and can reach the tree directly by clicking an alarm's origin DB name in the alarms viewer
**Depends on**: Phase 14
**Requirements**: TAGTREE-01, TAGTREE-02, TAGTREE-03, TAGTREE-04, INTEGRATION-01
**Success Criteria** (what must be TRUE):
  1. `TagTreeBrowserPage` loads when navigated to with `?db=<name>&connectionNumber=<N>` and displays the tag hierarchy for that datablock
  2. Expanding a tree node shows child tags and structs derived by parsing `protocolSourceObjectAddress` strings (e.g. `"DBName".SubStruct.Tag`) into a depth-matched hierarchy
  3. Leaf tag nodes show the data type from the corresponding `realtimeData` document
  4. Live tag values update automatically every 5 seconds for all currently expanded leaf tags; expanding a node triggers a `touchS7PlusActiveTagRequests` call for those tags
  5. In `S7PlusAlarmsViewerPage`, each `originDbName` cell is a clickable link that opens `TagTreeBrowserPage` in a new browser tab pre-filtered to that datablock and connection
**Plans**: TBD
**UI hint**: yes

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
| 12. Driver — Datablock Persistence | v1.4 | 1/1 | Complete    | 2026-03-26 |
| 13. Backend API — Datablocks & Tag Endpoints | v1.4 | 0/? | Not started | - |
| 14. DatablockBrowser | v1.4 | 0/? | Not started | - |
| 15. TagTreeBrowser & Integration | v1.4 | 0/? | Not started | - |

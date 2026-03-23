# Roadmap: S7CommPlus Alarm Subscriptions for json-scada

## Milestones

- ✅ **v1.0 — S7CommPlus Alarm Subscriptions PoC** — Phase 1 (shipped 2026-03-18)
- ✅ **v1.1 — Alarm Management & Viewer** — Phases 2–4 (shipped 2026-03-23)
- 📋 **v1.2 — Alarm Origin & Cleanup** — Phases 5–8 (active)

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

<details open>
<summary>📋 v1.2 — Alarm Origin & Cleanup (Phases 5–8) — ACTIVE</summary>

- [ ] **Phase 5: Driver — RelationId Fields** - Store relationId and dbNumber in every new alarm document
- [ ] **Phase 6: Driver — Startup DB Name Map** - Populate originDbName by resolving relationId against a startup-built PLC browse map
- [ ] **Phase 7: Backend — Delete Endpoint + _id Exposure** - Expose _id in list API and implement delete endpoint for single-row and bulk operations
- [ ] **Phase 8: Frontend — Delete Buttons + Origin Columns** - Display origin columns and enable per-row and bulk delete from the alarm viewer

</details>

## Phase Details

### Phase 5: Driver — RelationId Fields
**Goal**: Every new alarm event written to MongoDB carries the raw PLC origin identifiers
**Depends on**: Nothing (first phase of v1.2; builds on completed v1.1 driver)
**Requirements**: ORIGIN-01, ORIGIN-02
**Success Criteria** (what must be TRUE):
  1. A newly triggered alarm event document in `s7plusAlarmEvents` contains a `relationId` field stored as BsonInt64 (Int64, not Int32)
  2. A newly triggered alarm event document contains a `dbNumber` field (uint) correctly extracted as `relationId >> 16 & 0xFFFF`
  3. The driver compiles and runs without error after the field additions; alarm subscription loop continues normally
**Plans:** 0/1 plans executed

Plans:
- [ ] 05-01-PLAN.md — Add relationId (BsonInt64) and dbNumber (BsonInt32) to BuildAlarmDocument

---

### Phase 6: Driver — Startup DB Name Map
**Goal**: New alarm events carry a human-readable DB name resolved at write time from a startup-built PLC browse map
**Depends on**: Phase 5 (RelationIdNameMap field and BuildAlarmDocument signature must exist)
**Requirements**: ORIGIN-03, ORIGIN-04
**Success Criteria** (what must be TRUE):
  1. Driver calls `GetListOfDatablocks` (or equivalent browse) on the tag connection before `AlarmThread.Start()` executes; the call completes without blocking alarm subscription startup
  2. A newly triggered alarm event document contains an `originDbName` field; when the DB is in the PLC browse, the value matches the TIA Portal datablock name (e.g., `"AlarmDB"`)
  3. When the startup browse fails or returns no blocks, the driver continues without error and new alarm documents contain `originDbName: ""`
  4. Browse runs on `srv.connection` (tag connection), not `alarmConn`; no PDU interleaving observed
**Plans:** 1 plan

Plans:
- [ ] 06-01-PLAN.md — Add RelationIdNameMap field, browse call, and originDbName document field

---

### Phase 7: Backend — Delete Endpoint + _id Exposure
**Goal**: The alarm list API exposes document IDs and a new delete endpoint allows authenticated admins to remove alarm history
**Depends on**: Nothing (code-independent from Phases 5 and 6; must complete before Phase 8)
**Requirements**: DELETE-01, DELETE-02, DELETE-03, ORIGIN-05
**Success Criteria** (what must be TRUE):
  1. `GET /Invoke/auth/listS7PlusAlarms` response includes `_id` (ObjectId serialized as string) for each document; `relationId`, `dbNumber`, and `originDbName` fields are also present in each document
  2. `POST /Invoke/auth/deleteS7PlusAlarms` with body `{ "ids": ["<id>"] }` deletes the matching document and returns success; a second call with the same ID returns success (idempotent)
  3. `POST /Invoke/auth/deleteS7PlusAlarms` with body `{ "filter": { "alarmState": "Going", "alarmClassName": "Errors" } }` deletes all matching documents
  4. Calling the delete endpoint without a valid admin JWT returns 401/403; the document is not deleted
**Plans**: TBD
**UI hint**: no

---

### Phase 8: Frontend — Delete Buttons + Origin Columns
**Goal**: Operators can see the PLC origin of each alarm and delete individual or filtered alarm history from the viewer
**Depends on**: Phase 5, Phase 6 (originDbName in documents), Phase 7 (_id in list response + delete endpoint)
**Requirements**: ORIGIN-06, ORIGIN-07, ORIGIN-08, DELETE-04, DELETE-05, DELETE-06
**Success Criteria** (what must be TRUE):
  1. The alarm viewer table displays three new columns: "Origin DB Name", "DB Number", and "RelationId"; values are populated for alarms triggered after Phase 6 is deployed
  2. Clicking the per-row Delete button removes that row from the table immediately (optimistic removal); the document is absent from MongoDB on next page load
  3. The "Delete Filtered (N)" toolbar button shows the count of currently visible rows, is disabled when zero rows are visible, and deletes all visible rows when clicked
  4. Attempting to delete a row where `alarmState === 'Coming'` and `ackState === false` shows a confirmation warning before proceeding
**Plans**: TBD
**UI hint**: yes

---

## Progress

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. End-to-End Alarm Pipeline | v1.0 | 2/2 | Complete | 2026-03-18 |
| 2. Driver Fixes | v1.1 | 2/2 | Complete | 2026-03-18 |
| 3. Read-Only Alarm Viewer | v1.1 | 2/2 | Complete | 2026-03-19 |
| 4. Ack Write-Back | v1.1 | 2/2 | Complete | 2026-03-19 |
| 5. Driver — RelationId Fields | v1.2 | 0/1 | Planned    |  |
| 6. Driver — Startup DB Name Map | v1.2 | 0/1 | Planned | - |
| 7. Backend — Delete Endpoint + _id Exposure | v1.2 | 0/? | Not started | - |
| 8. Frontend — Delete Buttons + Origin Columns | v1.2 | 0/? | Not started | - |

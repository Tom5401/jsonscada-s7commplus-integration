# Roadmap: S7CommPlus Alarm Subscriptions for json-scada

## Milestones

- ✅ **v1.0 — S7CommPlus Alarm Subscriptions PoC** — Phase 1 (shipped 2026-03-18)
- ✅ **v1.1 — Alarm Management & Viewer** — Phases 2–4 (shipped 2026-03-23)
- ✅ **v1.2 — Alarm Origin & Cleanup** — Phases 5–8 (shipped 2026-03-24)
- 🔄 **v1.3 — Alarm Viewer Enhancements & Priority** — Phases 9–11 (active)

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

<details open>
<summary>🔄 v1.3 — Alarm Viewer Enhancements & Priority (Phases 9–11) — ACTIVE</summary>

- [x] **Phase 9: Driver Enrichment** - Store isAcknowledgeable and resolve alarmText/infoText placeholders in C# (completed 2026-03-25)
- [ ] **Phase 10: API Cap Removal** - Remove 200-alarm limit and add MongoDB index in same change
- [ ] **Phase 11: Vue UI Enhancements** - Timestamp, priority sort, ack indicator, source filter, Ack All, page preservation

</details>

## Phase Details

### Phase 9: Driver Enrichment
**Goal**: Every new alarm event stored in MongoDB carries a correct `isAcknowledgeable` flag and fully-resolved alarm text fields
**Depends on**: Nothing (first phase of v1.3; C# changes are self-contained)
**Requirements**: DRIVER-01, DRIVER-02
**Success Criteria** (what must be TRUE):
  1. A new alarm event document in MongoDB has `isAcknowledgeable: true` when `alarmClass == 33` and `isAcknowledgeable: false` for any other alarm class
  2. The `alarmText` field in a new alarm document contains the resolved text (e.g. "Motor speed: 1500 rpm") not the raw template (e.g. "Motor speed: @N%1@")
  3. The `infoText` field in a new alarm document contains the resolved text, not the raw `@N%x@` placeholder template
  4. Existing (pre-v1.3) alarm documents are unaffected — no migration performed
**Plans:** 1 plan
Plans:
- [x] 09-01-PLAN.md — Add isAcknowledgeable field and resolve alarmText/infoText placeholders (completed 2026-03-25)

### Phase 10: API Cap Removal
**Goal**: The alarm list API returns the full alarm collection and MongoDB query performance is protected by an index
**Depends on**: Phase 9
**Requirements**: API-01, API-02
**Success Criteria** (what must be TRUE):
  1. A MongoDB collection with more than 200 alarm documents returns all documents (up to the raised ceiling) when the alarm viewer loads
  2. The `{ createdAt: -1 }` index exists on `s7plusAlarmEvents` after server startup — verifiable via `db.s7plusAlarmEvents.getIndexes()`
  3. The API change and index creation ship as a single atomic change (same commit / same deployment)
**Plans**: TBD

### Phase 11: Vue UI Enhancements
**Goal**: Operators can view, filter, sort, and bulk-acknowledge alarms through a more informative and usable alarm viewer
**Depends on**: Phase 9, Phase 10
**Requirements**: VIEWER-01, VIEWER-02, VIEWER-03, VIEWER-04, VIEWER-05, VIEWER-06
**Success Criteria** (what must be TRUE):
  1. The alarm table shows a single combined timestamp column formatted as `2026-03-24_12:57:10.758`; separate Date and Time columns are gone
  2. Operator can click the Priority column header to sort alarms ascending or descending by numeric priority value
  3. Each alarm row displays a visible indicator (e.g. icon or label) showing whether the alarm requires acknowledgement or is information-only
  4. Operator can select a source PLC from a dropdown to filter the alarm table to alarms from that connection only, consistent with the existing Status and AlarmClass filters
  5. Operator can click "Ack All" to acknowledge all currently unacked, acknowledgeable alarms matching the active filter; a confirmation dialog shows the count before proceeding; a single failed ack does not block remaining acks
  6. After the 5-second auto-refresh, the operator's current page in the alarm table is preserved (no jump back to page 1)
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
| 10. API Cap Removal | v1.3 | 0/? | Not started | - |
| 11. Vue UI Enhancements | v1.3 | 0/? | Not started | - |

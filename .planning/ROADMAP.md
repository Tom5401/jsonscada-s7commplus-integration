# Roadmap: S7CommPlus Alarm Subscriptions for json-scada

## Milestones

- ✅ **v1.0 — S7CommPlus Alarm Subscriptions PoC** — Phase 1 (shipped 2026-03-18)
- 🚧 **v1.1 — Alarm Management & Viewer** — Phases 2–4 (in progress)

## Phases

<details>
<summary>✅ v1.0 — S7CommPlus Alarm Subscriptions PoC (Phase 1) — SHIPPED 2026-03-18</summary>

- [x] Phase 1: End-to-End Alarm Pipeline (2/2 plans) — completed 2026-03-18

Full details: [milestones/v1.0-ROADMAP.md](milestones/v1.0-ROADMAP.md)

</details>

### 🚧 v1.1 — Alarm Management & Viewer (In Progress)

**Milestone Goal:** Close the ack loop (fix ackState bug + write-back to PLC), enrich alarm data with alarm class names, and add a TIA Portal-style Alarms Viewer in AdminUI with per-row acknowledge capability.

- [x] **Phase 2: Driver Fixes** - Fix ackState always-true bug and add alarmClassName resolution to MongoDB alarm documents (completed 2026-03-18)
- [x] **Phase 3: Read-Only Alarm Viewer** - REST endpoints + S7PlusAlarmsViewerPage.vue with TIA Portal columns and auto-refresh (completed 2026-03-19)
- [ ] **Phase 4: Ack Write-Back** - Wireshark spike + full ack pipeline (Vue → Express → commandsQueue → C# → PLC) + Acknowledge button

## Phase Details

### Phase 2: Driver Fixes
**Goal**: MongoDB alarm documents contain correct, operator-readable data — ackState reflects actual PLC acknowledgement state and alarmClassName is a human-readable string
**Depends on**: Phase 1
**Requirements**: DRVR-01, DRVR-02
**Success Criteria** (what must be TRUE):
  1. A fresh unacknowledged alarm appears in MongoDB with `ackState: false`; acknowledging it on the PLC causes the next event to arrive with `ackState: true`
  2. Every new alarm document in `s7plusAlarmEvents` contains an `alarmClassName` string field (e.g., "Acknowledgment required") derived from the numeric alarm class ID
  3. Existing tag read/write and alarm subscription pipeline continue to function without regression after the C# changes
**Plans**: 2 plans

Plans:
- [ ] 02-01-PLAN.md — DRVR-01: PLCSIM spike + surgical ackState fix (AckTimestamp sentinel confirmed via live trace)
- [ ] 02-02-PLAN.md — DRVR-02: AlarmClassNames dictionary + alarmClassName BSON field (IDs from Plan 01 trace)

### Phase 3: Read-Only Alarm Viewer
**Goal**: An operator can navigate to a dedicated S7Plus Alarms Viewer page in AdminUI that displays alarms with TIA Portal-equivalent columns, auto-refreshes every 5 seconds, and supports filtering — independent of the existing tag-based alarm viewer
**Depends on**: Phase 2
**Requirements**: VIEW-01, VIEW-02, VIEW-04, VIEW-05, VIEW-06
**Success Criteria** (what must be TRUE):
  1. Navigating to `/s7plus-alarms` in AdminUI opens the new S7Plus Alarms Viewer page; the existing tag-based viewer at its original route loads correctly and is unchanged
  2. The viewer displays alarms in a table with columns: Source, Date, Time, Status, Acknowledge, Alarm class name, Event text, ID, Additional text 1–3
  3. The alarm table refreshes automatically every 5 seconds — a new alarm triggered on the PLC appears in the viewer within one refresh cycle without a manual page reload
  4. The operator can filter the displayed alarms by status (Incoming / Outgoing / All) and by alarm class, with the table updating to show only matching rows
**Plans**: 2 plans

Plans:
- [ ] 03-01-PLAN.md — Backend endpoint + S7PlusAlarmsViewerPage.vue component (table, filters, auto-refresh)
- [ ] 03-02-PLAN.md — AdminUI wiring: route registration, Dashboard tile, i18n keys (13 locales)

### Phase 4: Ack Write-Back
**Goal**: An operator can acknowledge an unacknowledged alarm from the viewer and the PLC receives and applies the acknowledgement command — closing the full ack loop from UI to PLC
**Depends on**: Phase 3
**Requirements**: DRVR-03, VIEW-03
**Success Criteria** (what must be TRUE):
  1. A Wireshark capture of TIA Portal sending an ack command to a live PLC is captured and the `SetVariableRequest` payload (object ID, attribute ID, value encoding) is decoded — this is the first deliverable of this phase; no ack send code is written before this spike is complete
  2. Clicking Acknowledge on an unacknowledged alarm row in the viewer sends a POST request; the PLC receives and applies the ack command; the alarm document in MongoDB is subsequently updated with `ackState: true`
  3. The Acknowledge button is enabled only for rows with `ackState: false`; it shows a pending state after click and prevents duplicate sends until the round-trip confirms acknowledgement
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 2 → 3 → 4

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. End-to-End Alarm Pipeline | v1.0 | 2/2 | Complete | 2026-03-18 |
| 2. Driver Fixes | 2/2 | Complete   | 2026-03-18 | - |
| 3. Read-Only Alarm Viewer | 2/2 | Complete   | 2026-03-19 | - |
| 4. Ack Write-Back | v1.1 | 0/? | Not started | - |

# Milestones

## v1.4 Tag Tree Browser (Shipped: 2026-03-27)

**Phases completed:** 4 phases, 4 plans, 2 tasks

**Key accomplishments:**

- Bulk-upsert of full PLC datablock list to MongoDB s7plusDatablocks collection at driver startup, with unique compound index and idempotent restart behavior
- One-liner:

---

## v1.3 Alarm Viewer Enhancements (Shipped: 2026-03-25)

**Phases completed:** 3 phases, 4 plans, 3 tasks

**Key accomplishments:**

- Single Timestamp column, sortable Priority column, ack indicator for non-acknowledgeable alarms, and page preservation across auto-refresh added to S7Plus Alarms Viewer.
- Source PLC filter dropdown and Ack All bulk action added to S7Plus Alarms Viewer — operators can narrow alarms by connection and bulk-acknowledge all unacked+acknowledgeable alarms matching active filters.

---

## v1.2 Alarm Origin & Cleanup (Shipped: 2026-03-24)

**Phases completed:** 4 phases, 4 plans, 5 tasks

**Key accomplishments:**

- Common.cs
- S7PlusAlarmsViewerPage.vue updated with origin columns, delete functionality, and Ack button restored via cherry-pick.

---

## v1.2 Alarm Origin & Cleanup (Shipped: 2026-03-24)

**Phases completed:** 4 phases, 4 plans, 5 tasks

**Key accomplishments:**

- Common.cs
- S7PlusAlarmsViewerPage.vue updated with origin columns, delete functionality, and Ack button restored via cherry-pick.

---

## v1.1 — Alarm Management & Viewer (Shipped: 2026-03-23)

**Phases:** 2–4 | **Plans:** 6 | **Timeline:** 2026-03-18 → 2026-03-19 (2 days)
**Archive:** [milestones/v1.1-ROADMAP.md](milestones/v1.1-ROADMAP.md)

**Delivered:** Full alarm management loop — fixed ackState bug, added alarm class names, TIA Portal-style viewer in AdminUI with per-row acknowledge capability wired to PLC.

**Key accomplishments:**

- Fixed ackState always-true bug (DateTime.UnixEpoch sentinel confirmed via live PLCSIM trace)
- Added alarmClassName field to MongoDB alarm documents (AlarmClass 33 = "Acknowledgment required"; Unknown (N) fallback)
- S7PlusAlarmsViewerPage.vue — 11-column TIA Portal-equivalent table with 5s auto-refresh and status/class filters; i18n across 13 locales
- Full ack pipeline: Vue Ack button → POST /ackS7PlusAlarm → commandsQueue → C# AlarmThread.SendAlarmAck via alarmConn → PLC → ackState updated in MongoDB
- AckJob PDU decoded via Wireshark spike (CreateObjectRequest with 13 numeric IDs from s7comm_plus.dll)

---

## v1.0 — S7CommPlus Alarm Subscriptions PoC (Shipped: 2026-03-18)

**Phases:** 1 | **Plans:** 2 | **Timeline:** 2026-03-16 → 2026-03-18 (3 days)
**Archive:** [milestones/v1.0-ROADMAP.md](milestones/v1.0-ROADMAP.md)

**Delivered:** Native S7CommPlus alarm subscription pipeline for json-scada — PLC alarms arrive in MongoDB `s7plusAlarmEvents` via push subscription, not polling, with full metadata.

**Key accomplishments:**

- Fixed 5 protocol bugs blocking alarm subscriptions (credit-limit renewal, subscription slot leak, RelationId collision, unknown PDU crash, ack-only null dereference)
- Dedicated AlarmThread with full subscription lifecycle, graceful error handling, clean shutdown
- Alarm events persisted to s7plusAlarmEvents with 14 fields including resolved additionalTexts (SD_1–SD_9 with @N%x@ substitution) and typed associatedValues (SD_1–SD_10)
- Live PLC validated with PLCSIM Advanced V8 + TIA Portal v21: Coming/Going events confirmed; tag polling unaffected

---

# Milestones

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

---
phase: 2
slug: driver-fixes
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-18
---

# Phase 2 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | none — .NET 8 console; no unit test project |
| **Config file** | none |
| **Quick run command** | `dotnet build S7CommPlusClient/S7CommPlusClient.sln` |
| **Full suite command** | manual PLCSIM + MongoDB query (see Manual-Only Verifications) |
| **Estimated runtime** | ~10 min (manual PLCSIM session) |

---

## Sampling Rate

- **After every task commit:** Run `dotnet build S7CommPlusClient/S7CommPlusClient.sln`
- **After every plan wave:** Run manual PLCSIM verification session
- **Before `/gsd:verify-work`:** Full manual verification must pass
- **Max feedback latency:** build: ~30s; full verification: ~10 min

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 2-01-01 | 01 | 1 | DRVR-01 | manual | `dotnet build` (compile check) | ✅ | ⬜ pending |
| 2-01-02 | 01 | 2 | DRVR-01 | manual | PLCSIM trace + MongoDB query | N/A | ⬜ pending |
| 2-02-01 | 02 | 1 | DRVR-02 | manual | `dotnet build` (compile check) | ✅ | ⬜ pending |
| 2-02-02 | 02 | 2 | DRVR-02 | manual | PLCSIM + MongoDB query | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

*Existing infrastructure covers all phase requirements (build-only automated check; full verification is manual PLCSIM + MongoDB).*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `ackState: false` for unacknowledged alarm | DRVR-01 | Requires live PLC/PLCSIM — no unit test harness | Trigger unacknowledged alarm; query `db.s7plusAlarmEvents.find({},{ackState:1})` — expect `false` |
| `ackState: true` after acknowledgement | DRVR-01 | Requires live PLC acknowledgement action | Acknowledge alarm on PLC; verify next event has `ackState: true` |
| `alarmClassName` string present in document | DRVR-02 | Requires live alarm event from PLCSIM | Trigger alarm; query `db.s7plusAlarmEvents.find({},{alarmClassName:1})` — expect non-null string |
| `alarmClassName: "Unknown (N)"` for unmapped ID | DRVR-02 | Requires alarm with non-standard class ID | Trigger alarm with non-standard class; verify fallback string format |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 600s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending

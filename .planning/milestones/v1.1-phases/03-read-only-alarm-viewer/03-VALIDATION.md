---
phase: 3
slug: read-only-alarm-viewer
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-18
---

# Phase 3 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | None configured (AdminUI: no vitest/jest; server_realtime_auth: no test runner) |
| **Config file** | None — manual smoke testing only |
| **Quick run command** | Manual browser smoke test against running AdminUI |
| **Full suite command** | Full manual acceptance checklist (all 5 requirements) |
| **Estimated runtime** | ~5 minutes manual walkthrough |

---

## Sampling Rate

- **After every task commit:** Manual browser smoke test against running AdminUI
- **After every plan wave:** Full manual acceptance checklist (all 5 requirements)
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** ~300 seconds (manual)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 3-01-01 | 01 | 1 | VIEW-01 | manual smoke | Manual: GET /Invoke/auth/listS7PlusAlarms returns JSON array | N/A | ⬜ pending |
| 3-02-01 | 02 | 1 | VIEW-01 | manual smoke | Manual: navigate to /#/s7plus-alarms, page loads | N/A | ⬜ pending |
| 3-02-02 | 02 | 1 | VIEW-02 | manual smoke | Manual: alarm table shows Source, Date, Time, Status, Acknowledge, Alarm class name, Event text, ID, Additional text 1-3 columns | N/A | ⬜ pending |
| 3-02-03 | 02 | 1 | VIEW-04 | manual smoke | Manual: wait 5s, observe table refresh without manual reload | N/A | ⬜ pending |
| 3-02-04 | 02 | 1 | VIEW-05 | manual smoke | Manual: set Status filter to Incoming, verify only Coming rows shown | N/A | ⬜ pending |
| 3-02-05 | 02 | 1 | VIEW-06 | manual smoke | Manual: alarm class dropdown populated from dataset, filter works | N/A | ⬜ pending |
| 3-03-01 | 03 | 2 | VIEW-01 | manual smoke | Manual: /alarms-viewer route still loads unchanged | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

No automated test infrastructure exists for AdminUI or server_realtime_auth. All phase requirements are verified manually against a running stack. No Wave 0 test file creation needed.

*Existing infrastructure covers all phase requirements (via manual smoke testing).*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Navigate to `/s7plus-alarms` → viewer loads; `/alarms-viewer` unchanged | VIEW-01 | No e2e test framework | Open browser, navigate to /#/s7plus-alarms, confirm S7Plus viewer; then navigate to /#/alarms-viewer, confirm existing page unchanged |
| Alarm table shows all 9 required columns | VIEW-02 | No component test framework | Verify Source, Date, Time, Status (v-chip), Acknowledge (icon), Alarm class name, Event text, ID, Additional text 1-3 columns visible |
| Auto-refresh: new alarm appears within 5s without reload | VIEW-04 | Requires live PLC/PLCSIM connection | Trigger a new alarm on PLCSIM, observe it appears in table within one 5-second cycle |
| Status filter: Incoming/Outgoing/All filters rows | VIEW-05 | No component test framework | Set Status dropdown to Incoming, verify only Coming rows; set to Outgoing, verify only Going rows; set to All, verify all rows |
| Alarm class filter: populated from data, filters correctly | VIEW-06 | Dynamic options require live data | Confirm dropdown shows distinct class names from loaded alarms; select one, verify table shows only matching rows |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 300s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending

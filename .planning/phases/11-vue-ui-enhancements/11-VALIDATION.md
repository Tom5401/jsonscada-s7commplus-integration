---
phase: 11
slug: vue-ui-enhancements
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-25
---

# Phase 11 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | none — manual smoke testing (no Vitest installed) |
| **Config file** | none |
| **Quick run command** | `npm run dev` (visual inspection in browser) |
| **Full suite command** | Manual smoke test checklist |
| **Estimated runtime** | ~5 minutes manual |

---

## Sampling Rate

- **After every task commit:** Visual check in browser at localhost
- **After every plan wave:** Full manual smoke checklist
- **Before `/gsd:verify-work`:** All 6 success criteria verified manually
- **Max feedback latency:** ~5 minutes

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 11-01-01 | 01 | 1 | VIEWER-01 | manual | visual inspect timestamp column | ✅ | ⬜ pending |
| 11-01-02 | 01 | 1 | VIEWER-02 | manual | click Priority header, verify sort | ✅ | ⬜ pending |
| 11-01-03 | 01 | 1 | VIEWER-03 | manual | verify ack indicator icons visible | ✅ | ⬜ pending |
| 11-01-04 | 01 | 1 | VIEWER-04 | manual | select PLC from dropdown, verify filter | ✅ | ⬜ pending |
| 11-01-05 | 01 | 1 | VIEWER-05 | manual | click Ack All, verify dialog + bulk ack | ✅ | ⬜ pending |
| 11-01-06 | 01 | 1 | VIEWER-06 | manual | navigate to page 2, wait refresh, verify page preserved | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. All changes are isolated to a single Vue component — no test framework setup needed for PoC.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Timestamp column shows `2026-03-24_12:57:10.758` format | VIEWER-01 | No test framework; UI-only | Load app, confirm single combined column with underscore separator |
| Priority sort ascending/descending | VIEWER-02 | UI interaction | Click Priority header once (asc), again (desc), verify numeric sort |
| Ack indicator shown per row | VIEWER-03 | Visual rendering | Confirm each row shows icon/label for ack-required vs info-only |
| PLC source filter dropdown | VIEWER-04 | UI interaction | Select a PLC, confirm table filters to that source only |
| Ack All with confirmation dialog | VIEWER-05 | Multi-step interaction | Click Ack All, verify dialog shows count, confirm, verify all acked |
| Page preserved after auto-refresh | VIEWER-06 | Timing-dependent | Navigate to page 2+, wait 5 seconds, verify page number unchanged |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 300s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending

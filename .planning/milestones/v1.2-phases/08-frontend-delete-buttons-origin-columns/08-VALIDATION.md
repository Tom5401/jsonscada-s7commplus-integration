---
phase: 8
slug: frontend-delete-buttons-origin-columns
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-24
---

# Phase 8 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | None — no automated test infrastructure in AdminUI project |
| **Config file** | none |
| **Quick run command** | Manual: load alarm viewer in browser, verify columns/buttons |
| **Full suite command** | Manual: full verification checklist (see PLAN.md verification section) |
| **Estimated runtime** | ~5 minutes (manual browser verification) |

---

## Sampling Rate

- **After every task commit:** Browser visual check — load alarm viewer, verify columns/buttons render
- **After every plan wave:** Full manual verification checklist
- **Before `/gsd:verify-work`:** Full manual suite must pass
- **Max feedback latency:** ~5 minutes

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 8-01-01 | 01 | 1 | ORIGIN-06 | manual | — | ❌ Wave 0 | ⬜ pending |
| 8-01-02 | 01 | 1 | ORIGIN-07 | manual | — | ❌ Wave 0 | ⬜ pending |
| 8-01-03 | 01 | 1 | DELETE-04 | manual | — | ❌ Wave 0 | ⬜ pending |
| 8-01-04 | 01 | 1 | DELETE-05 | manual | — | ❌ Wave 0 | ⬜ pending |
| 8-01-05 | 01 | 1 | DELETE-06 | manual | — | ❌ Wave 0 | ⬜ pending |

*ORIGIN-08 skipped per decision D-02 (RelationId column deferred).*

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

No automated test infrastructure — AdminUI has no vitest/jest config or tests directory. All prior phases follow browser/runtime verification pattern. Adding a full test stack is out of scope for this phase.

*Manual verification is the established project pattern for AdminUI changes.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| "Origin DB Name" column renders in table | ORIGIN-06 | No test infra in AdminUI | Load alarm viewer; verify "Origin DB Name" column header and cell values appear |
| "DB Number" column renders in table | ORIGIN-07 | No test infra in AdminUI | Load alarm viewer; verify "DB Number" column header and numeric values appear |
| Per-row delete removes row optimistically | DELETE-04 | No test infra in AdminUI | Click row delete button; verify row disappears immediately; reload page and confirm row absent |
| "Delete Filtered (N)" button count and disable | DELETE-05 | No test infra in AdminUI | Filter table; verify button shows row count; clear filter; verify button disabled at 0 |
| Coming+unacked row shows warning dialog | DELETE-06 | No test infra in AdminUI | Click delete on Coming+unacked row; verify confirmation dialog appears before delete proceeds |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 300s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending

---
phase: 15
slug: tagtreebrowser-integration
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-26
---

# Phase 15 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | none — Vue SFC; validation is manual browser testing (PoC, no vitest configured) |
| **Config file** | none |
| **Quick run command** | `cd json-scada/src/AdminUI && npm run build 2>&1 | tail -5` |
| **Full suite command** | `cd json-scada/src/AdminUI && npm run build` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd json-scada/src/AdminUI && npm run build 2>&1 | tail -5`
- **After every plan wave:** Run `cd json-scada/src/AdminUI && npm run build`
- **Before `/gsd:verify-work`:** Full build must be green
- **Max feedback latency:** 20 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 15-01-01 | 01 | 1 | TAGTREE-01,04 | build | `npm run build` | ✅ | ⬜ pending |
| 15-01-02 | 01 | 1 | TAGTREE-01,02 | build | `npm run build` | ✅ | ⬜ pending |
| 15-01-03 | 01 | 1 | TAGTREE-02 | build | `npm run build` | ✅ | ⬜ pending |
| 15-01-04 | 01 | 1 | TAGTREE-03 | build | `npm run build` | ✅ | ⬜ pending |
| 15-01-05 | 01 | 2 | INTEGRATION-01 | build | `npm run build` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements (no new test files — PoC project with no automated test suite).

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Tag tree renders with correct hierarchy depth | TAGTREE-01 | Requires live PLC or PLCSIM data in MongoDB | Navigate to `/s7plus-tag-tree?db=<name>&connectionNumber=N`; confirm tree shows structs and leaf tags at correct depth |
| Live values update every 5 seconds | TAGTREE-02 | Requires live tag polling | Expand a leaf tag, wait 10s, confirm value changes if PLC updates the tag |
| touchS7PlusActiveTagRequests fires on expand | TAGTREE-02 | Requires browser DevTools / network tab | Open DevTools > Network; expand a node; confirm POST to `/Invoke/auth/touchS7PlusActiveTagRequests` |
| Leaf type column shows correct type | TAGTREE-03 | Requires real realtimeData docs | Expand leaf tag; confirm type matches `protocolSourceASDU` mapping in TagMapping.cs |
| URL params bootstrap the page correctly | TAGTREE-04 | Requires browser navigation | Open `/#/s7plus-tag-tree?db=DB1&connectionNumber=1`; confirm page loads for that DB without manual selection |
| originDbName link opens TagTreeBrowser | INTEGRATION-01 | Requires running AdminUI | In S7PlusAlarmsViewerPage, click an originDbName cell; confirm new tab opens at correct URL |
| Empty originDbName shows plain text | INTEGRATION-01 | Requires alarm with empty originDbName | Confirm cells with `""` render as `-` or blank without a link |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 20s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending

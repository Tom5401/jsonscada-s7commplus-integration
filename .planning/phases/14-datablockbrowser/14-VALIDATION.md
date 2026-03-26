---
phase: 14
slug: datablockbrowser
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-26
---

# Phase 14 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | None (no automated test framework in AdminUI) |
| **Config file** | none |
| **Quick run command** | `cd json-scada/src/AdminUI && npm run build` |
| **Full suite command** | `cd json-scada/src/AdminUI && npm run build` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run build` (zero TypeScript/ESLint errors)
- **After every plan wave:** Run `npm run build`
- **Before `/gsd:verify-work`:** Build must succeed + manual browser check
- **Max feedback latency:** ~30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 14-01-01 | 01 | 1 | DBBROWSER-01 | build | `cd json-scada/src/AdminUI && npm run build` | ❌ W0 | ⬜ pending |
| 14-01-02 | 01 | 1 | DBBROWSER-02 | build | `cd json-scada/src/AdminUI && npm run build` | ❌ W0 | ⬜ pending |
| 14-01-03 | 01 | 1 | DBBROWSER-03 | build | `cd json-scada/src/AdminUI && npm run build` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `json-scada/src/AdminUI/src/components/DatablockBrowserPage.vue` — new component file (created in Wave 1)
- [ ] `json-scada/src/AdminUI/src/router/index.js` — route registration (modified in Wave 1)
- [ ] `json-scada/src/AdminUI/src/components/DashboardPage.vue` — shortcut entry (modified in Wave 1)
- [ ] `json-scada/src/AdminUI/src/locales/en.json` — i18n key (modified in Wave 1)

*All Wave 0 artifacts are created/modified in Wave 1 — no separate setup wave needed.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Dashboard card visible, navigates to DatablockBrowserPage | DBBROWSER-01 | No E2E framework | Open AdminUI, login, verify "Datablock Browser" card on dashboard; click it |
| Connection dropdown loads all connections, selecting one populates datablock table | DBBROWSER-02 | No E2E framework | On DatablockBrowserPage, open connection dropdown; select a connection; verify datablocks appear |
| "Browse Tags" button opens TagTreeBrowserPage in new tab at correct URL | DBBROWSER-03 | No E2E framework | Click "Browse Tags" on a row; verify new tab opens at `/#/s7plus-tag-tree?db=...&connectionNumber=N` |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending

---
phase: 7
slug: backend-delete-endpoint-id-exposure
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-23
---

# Phase 7 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual HTTP / curl (no automated test suite detected) |
| **Config file** | none |
| **Quick run command** | `curl -s http://localhost:3000/Invoke/auth/listS7PlusAlarms -H "Authorization: Bearer <token>"` |
| **Full suite command** | Manual verification per success criteria |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Manual smoke test of the modified endpoint
- **After every plan wave:** Full manual verification of all success criteria
- **Before `/gsd:verify-work`:** All 4 success criteria must be confirmed green
- **Max feedback latency:** ~30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 7-01-01 | 01 | 1 | ORIGIN-05, DELETE-01 | manual | `curl GET /listS7PlusAlarms` includes `_id`, `relationId`, `dbNumber`, `originDbName` | ✅ | ⬜ pending |
| 7-02-01 | 02 | 2 | DELETE-02 | manual | `curl POST /deleteS7PlusAlarms` with `ids` array deletes document | ✅ | ⬜ pending |
| 7-02-02 | 02 | 2 | DELETE-02 | manual | Second delete call with same ID returns success (idempotent) | ✅ | ⬜ pending |
| 7-02-03 | 02 | 2 | DELETE-03 | manual | `curl POST /deleteS7PlusAlarms` with `filter` object deletes matching docs | ✅ | ⬜ pending |
| 7-02-04 | 02 | 2 | DELETE-01 | manual | Delete call without valid admin JWT returns 401/403; doc not deleted | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements.

*No new test files required — validation is manual curl-based against the running server.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `_id` exposed as hex string in list response | ORIGIN-05 | No automated test framework | `curl GET /Invoke/auth/listS7PlusAlarms` with admin JWT; confirm `_id` field present as 24-char hex string |
| `relationId`, `dbNumber`, `originDbName` in list response | ORIGIN-05 | No automated test framework | Same curl as above; confirm all three fields present per document |
| Delete by IDs array — success + idempotent | DELETE-02 | No automated test framework | POST `{ "ids": ["<id>"] }` twice; both return success; document absent after first |
| Delete by filter object | DELETE-03 | No automated test framework | POST `{ "filter": { "alarmState": "Going", "alarmClassName": "Errors" } }`; matching docs deleted |
| 401/403 without valid admin JWT | DELETE-01 | No automated test framework | POST without token → expect 401; POST with non-admin token → expect 403 |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending

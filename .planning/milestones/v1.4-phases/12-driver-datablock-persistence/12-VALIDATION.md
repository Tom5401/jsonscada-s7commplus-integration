---
phase: 12
slug: driver-datablock-persistence
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-26
---

# Phase 12 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | None (PoC — manual validation only per PROJECT.md) |
| **Config file** | none |
| **Quick run command** | `dotnet build src/S7CommPlusClient/` |
| **Full suite command** | Manual: restart driver twice, verify no duplicate documents in `s7plusDatablocks` |
| **Estimated runtime** | ~30 seconds build; ~5 minutes manual smoke |

---

## Sampling Rate

- **After every task commit:** Run `dotnet build src/S7CommPlusClient/`
- **After every plan wave:** Manual inspection of `s7plusDatablocks` collection after driver restart
- **Before `/gsd:verify-work`:** All 3 success criteria verified manually
- **Max feedback latency:** ~30 seconds (build)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 12-01-01 | 01 | 1 | DRIVER-03 | build | `dotnet build src/S7CommPlusClient/` | ✅ | ⬜ pending |
| 12-01-02 | 01 | 1 | DRIVER-03 | build | `dotnet build src/S7CommPlusClient/` | ✅ | ⬜ pending |
| 12-01-03 | 01 | 1 | DRIVER-03 | manual-smoke | `mongosh --eval "db.s7plusDatablocks.find({connectionNumber:1}).count()"` | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

None — Existing infrastructure covers all phase requirements. No new test files needed (PoC project, manual validation only per PROJECT.md).

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `s7plusDatablocks` contains one doc per PLC datablock after startup | DRIVER-03 | PoC — no automated test infrastructure | Restart driver; run `mongosh --eval "db.s7plusDatablocks.find({connectionNumber:1}).count()"` and compare to TIA Portal DB count |
| Documents contain all 5 required fields | DRIVER-03 | PoC — no automated test infrastructure | Run `mongosh --eval "db.s7plusDatablocks.findOne({connectionNumber:1})"` and verify `db_name`, `db_number`, `db_block_relid`, `db_block_ti_relid`, `connectionNumber` are present |
| Restart produces upsert, not duplicate | DRIVER-03 | PoC — no automated test infrastructure | Record doc count after first start; restart driver; verify count is unchanged (not doubled) |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 60s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending

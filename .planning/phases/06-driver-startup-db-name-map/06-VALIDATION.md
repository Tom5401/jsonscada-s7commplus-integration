---
phase: 6
slug: driver-startup-db-name-map
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-23
---

# Phase 6 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | None — no automated test suite exists in S7CommPlusClient |
| **Config file** | none |
| **Quick run command** | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` |
| **Full suite command** | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` |
| **Estimated runtime** | ~10 seconds |

---

## Sampling Rate

- **After every task commit:** Run `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj`
- **After every plan wave:** Run `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj`
- **Before `/gsd:verify-work`:** Clean build + manual PLCSIM verification of `originDbName` field in MongoDB
- **Max feedback latency:** ~10 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 6-01-01 | 01 | 1 | ORIGIN-03 | Build | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` | ✅ | ⬜ pending |
| 6-01-02 | 01 | 1 | ORIGIN-03 | Build | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` | ✅ | ⬜ pending |
| 6-01-03 | 01 | 1 | ORIGIN-04 | Build | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

None — no new test files are needed. The phase is three targeted edits to existing files. Existing build infrastructure covers all automated checks.

*Existing infrastructure covers all phase requirements.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Map builds from `GetListOfDatablocks` before alarm start | ORIGIN-03 | No unit test project; requires live PLC or PLCSIM connection | Connect driver to PLCSIM, trigger alarm, confirm `originDbName` matches TIA Portal DB name in MongoDB alarm document |
| Browse failure leaves empty map; alarm thread starts normally | ORIGIN-03 | Requires simulated PLC failure or connection disruption | Force `GetListOfDatablocks` to fail (e.g., disconnect/reconnect during startup), confirm `AlarmThread started` log appears and `originDbName: ""` in alarm docs |
| `originDbName` field present on every alarm document | ORIGIN-04 | Requires live alarm event from PLCSIM | Trigger alarm in PLCSIM, query MongoDB, confirm `originDbName` field exists with correct string value |
| Empty string fallback when DB not in map | ORIGIN-04 | Requires specific PLC state (load-memory-only DB) | Verify that alarms from a DB not returned by `GetListOfDatablocks` have `originDbName: ""` |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending

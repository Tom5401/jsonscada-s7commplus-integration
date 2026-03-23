---
phase: 5
slug: driver-relationid-fields
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-23
---

# Phase 5 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | None (no xUnit/NUnit project in S7CommPlusClient) |
| **Config file** | N/A — no test config file |
| **Quick run command** | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` |
| **Full suite command** | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` (compile is only automated gate) |
| **Estimated runtime** | ~10 seconds |

---

## Sampling Rate

- **After every task commit:** Run `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj`
- **After every plan wave:** Run `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj`
- **Before `/gsd:verify-work`:** Full build must be green (0 errors)
- **Max feedback latency:** ~10 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 5-01-01 | 01 | 1 | ORIGIN-01, ORIGIN-02 | compile + manual | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` | ✅ existing | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. No new test files needed.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `relationId` field present in alarm document as BsonInt64 with correct value | ORIGIN-01 | Requires live PLC/PLCSIM to produce `AlarmsDai` objects; no mock harness exists | Trigger an alarm event, then inspect `s7plusAlarmEvents` document in MongoDB Compass or mongosh — verify `relationId` is a positive Int64 value |
| `dbNumber` field present as BsonInt32 with value = `relationId & 0xFFFF` | ORIGIN-02 | Same — requires live alarm event | Inspect same document — verify `dbNumber` is a small Int32 (≤ 65535) equal to the lower 16 bits of `relationId` |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending

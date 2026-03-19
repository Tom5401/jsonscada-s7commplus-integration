---
phase: 4
slug: ack-write-back
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-19
---

# Phase 4 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | none — PoC; manual verification + Wireshark capture |
| **Config file** | none |
| **Quick run command** | `curl -s http://localhost:3000/Invoke/auth/ackAlarm -X POST -H "Content-Type: application/json" -d '{}' \| head -c 200` |
| **Full suite command** | Manual: Wireshark capture + PLC ack round-trip + MongoDB ackState check |
| **Estimated runtime** | ~5 minutes (manual) |

---

## Sampling Rate

- **After every task commit:** Verify the file(s) touched compile / start without errors
- **After every plan wave:** Run manual round-trip test: POST → commandsQueue → C# branch → PLC → MongoDB `ackState: true`
- **Before `/gsd:verify-work`:** Full manual test suite must pass
- **Max feedback latency:** One poll cycle (~5 seconds) for ackState confirmation

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 4-01-01 | 01 | 1 | DRVR-03 | manual | Wireshark capture shows `SetVariableRequest` payload decoded | ❌ W0 | ⬜ pending |
| 4-01-02 | 01 | 1 | DRVR-03 | manual | Prototype C# method sends ack PDU to live PLC without error | ❌ W0 | ⬜ pending |
| 4-02-01 | 02 | 2 | DRVR-03 | manual | `curl POST /Invoke/auth/ackAlarm` returns 200; commandsQueue document inserted with ASDU `s7plus-alarm-ack` | ❌ W0 | ⬜ pending |
| 4-02-02 | 02 | 2 | DRVR-03 | manual | C# Change Stream detects insert; `protocolSourceASDU == "s7plus-alarm-ack"` branch executes; PLC receives ack | ❌ W0 | ⬜ pending |
| 4-02-03 | 02 | 2 | DRVR-03 | manual | MongoDB alarm document updated with `ackState: true` after successful round-trip | ❌ W0 | ⬜ pending |
| 4-03-01 | 03 | 3 | VIEW-03 | manual | Vue Ack button visible only for `ackState: false` rows; not visible for `ackState: true` rows | ❌ W0 | ⬜ pending |
| 4-03-02 | 03 | 3 | VIEW-03 | manual | Clicking Ack button shows spinner and disables button (pending state); no duplicate POSTs sent | ❌ W0 | ⬜ pending |
| 4-03-03 | 03 | 3 | VIEW-03 | manual | After poll returns `ackState: true`, spinner clears and row shows `mdi-check` green icon | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

None — no automated test framework in use for this PoC phase. All verifications are manual.

*Existing infrastructure covers all phase requirements.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Wireshark spike: decode `SetVariableRequest` PDU for alarm ack | DRVR-03 | Requires live PLC + Wireshark; no automated OPC UA ack test tooling | Capture TIA Portal ack → filter S7CommPlus → decode InObjectId, Address, value encoding |
| PLC receives and applies ack command | DRVR-03 | Requires live PLC hardware | POST ack → verify PLC alarm state clears in TIA Portal diagnostics |
| Vue pending state prevents duplicate sends | VIEW-03 | Browser interaction testing | Click Ack rapidly multiple times; verify only one POST sent via network tab |
| Poll-based confirmation: ackState transitions false→true | VIEW-03 | Requires full stack running | Wait for 5s poll; verify spinner clears and icon switches to mdi-check |
| Failure path: re-enables button on POST error | VIEW-03 | Requires forced error condition | Shut down Express or PLC; click Ack; verify button re-enables after error |

---

## Validation Sign-Off

- [ ] All tasks have manual verify steps documented above
- [ ] Sampling continuity: manual check after each plan wave
- [ ] Wave 0: N/A (no test framework for this PoC)
- [ ] No watch-mode flags
- [ ] Feedback latency < 5s (one poll cycle)
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending

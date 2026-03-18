# Retrospective

## Milestone: v1.0 — S7CommPlus Alarm Subscriptions PoC

**Shipped:** 2026-03-18
**Phases:** 1 | **Plans:** 2

### What Was Built

- Fixed 5 protocol-level bugs in S7CommPlusDriver alarm subscription code that had never been exercised in combination with a real PLC
- Built AlarmThread.cs: dedicated per-connection alarm thread with full lifecycle, receive loop, graceful error handling, and MongoDB write
- Validated end-to-end with live PLC (PLCSIM Advanced V8 + TIA Portal v21): Coming/Going events flowing, tag polling unaffected
- Extended alarm document schema beyond v1 requirements: infoText, additionalTexts with @N%x@ placeholder resolution, typed associatedValues (SD_1–SD_10)

### What Worked

- **Single delivery phase for tightly coupled requirements** — all 17 v1 requirements were interdependent (bugs, lifecycle, threading, data, MongoDB must all work together). Grouping them in one phase avoided artificial splits that would have left intermediate states untestable.
- **Submodule separation** — keeping S7CommPlusDriver as a submodule made bug fixes cleanly separable from client code, and each was committable independently.
- **Live PLC as acceptance gate** — `dotnet build` as automated gate + live PLC as behavioral gate was the right trade-off for a protocol-heavy PoC. Unit tests would have required mocking the entire S7CommPlus PDU stack.
- **Post-UAT fix loop** — discovering the timeout-exit bug during live UAT and fixing it inline (rather than creating a new phase) was the right call for a PoC. The fix was surgical.

### What Was Inefficient

- **SUMMARY artifacts predate post-execution enhancements** — alarm timeout fix, additionalTexts, and placeholder resolution were committed after the phase SUMMARY was written. Future phases should hold the SUMMARY open until all live-test follow-ups are complete.
- **Stale ROADMAP progress table** — never updated from "0/2 Not started" during execution. The table should be updated as part of each plan completion commit.

### Patterns Established

- Alarm subscription uses a separate `S7CommPlusConnection` instance from the tag connection — avoids PDU interleaving and shared-state threading issues
- WaitForAlarmNotification timeout (errTCPDataReceive) cleared to 0 in driver layer — consumer checks `m_LastError == 0` to distinguish idle timeout (continue) from real error (break)
- AlarmThread as `partial class` in separate file — keeps ConnectionThread focused on tag polling lifecycle
- MongoDB alarm document schema: 14 fields with typed SD values and resolved text placeholders

### Key Lessons

- The S7CommPlusDriver alarming code was experimental/untested — assume any "TODO Unknown" comment in protocol code hides a latent bug. Fix eagerly before live test.
- `WaitForNewS7plusReceived` ERROR logging was hardcoded and misleading — library code that logs "ERROR" unconditionally on expected conditions causes false alarm fatigue. Always check whether the log is appropriate for all callers.
- Associated data values (SD_x) are already parsed in `AsCgs.AssociatedValues` — the alarm text template substitution was a one-function addition that makes the data immediately useful without any protocol work.

### Cost Observations

- Sessions: 2 (execution + live UAT + follow-up fixes)
- Execution was fast (~25 min total for both plans) — the research and planning phases did the heavy lifting
- Live PLC validation caught the only real bug (timeout exit); static analysis and planning missed it

---

## Cross-Milestone Trends

| Milestone | Phases | Plans | Bugs Found in UAT | Post-Execution Fixes |
|-----------|--------|-------|-------------------|----------------------|
| v1.0 PoC  | 1      | 2     | 1 (timeout exit)  | 4 (timeout, log level, additionalTexts, placeholder resolution) |

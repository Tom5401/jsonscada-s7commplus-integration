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

## Milestone: v1.2 — Alarm Origin & Cleanup

**Shipped:** 2026-03-24
**Phases:** 4 | **Plans:** 4

### What Was Built

- Driver stores `relationId` (BsonInt64) and `dbNumber` in every alarm document — PLC origin identifiers captured at write time without protocol overhead
- Driver builds a RelationId→DB-name map at startup via `GetListOfDatablocks`; alarm viewer gains `originDbName` (TIA Portal datablock name) without any per-alarm PLC query
- `listS7PlusAlarms` API now returns `_id`; new `POST /Invoke/auth/deleteS7PlusAlarms` with admin guard handles both single-row and bulk delete (HTTP 204)
- Alarm viewer gains Origin DB Name + DB Number columns; per-row delete (optimistic removal) and bulk "Delete Filtered (N)" with confirmation dialogs
- Phase 4 Ack button recovered via cherry-pick after branch divergence — three commits stranded on an unmerged branch since Phase 3 were identified and cherry-picked onto the Phase 8 tip

### What Worked

- **Phase isolation at the right granularity** — splitting driver fields (Phase 5), startup browse (Phase 6), backend API (Phase 7), and frontend (Phase 8) allowed each to be planned, executed, and verified independently; no cross-phase rework needed
- **Empty-string fallback pattern** — choosing `""` over `null` for `originDbName` when DB not in map gave Phase 7/8 consumers a clean contract (no null checks needed anywhere downstream)
- **Integration checker** caught the DELETE-03 asymmetry (filter-path backend with no UI caller) cleanly — confirmed it was intentional before archiving
- **VERIFICATION.md human-gating** worked well: the 5 human items identified across phases (build, live PLC, auth guard) were exactly the right things to flag; none blocked milestone completion since code correctness was separately confirmed

### What Was Inefficient

- **Phase 4 branch divergence** — the biggest time sink of the milestone. Phase 4's ack commits existed on an unmerged submodule branch; Phases 5–8 were built on a tip that never included them. Cherry-picking onto Phase 8 required resolving 5 conflict blocks across 3 files and re-verifying correctness. Prevention: run `git submodule status` for `+` prefix before starting each phase; run `/gsd:progress` between milestones.
- **Merge conflict markers committed to fork** — during conflict resolution, `git add` accepted files with unresolved markers (git does not validate conflict-marker absence). The markers were committed and pushed before being caught in the IDE. Prevention: grep for `<<<<<<<` after resolving before staging.
- **REQUIREMENTS.md checkboxes not updated post-Phase 8** — 5 checkboxes remained `[ ]` after Phase 8 execution; had to note them as "stale" in the audit rather than simply reading the file. Prevention: update REQUIREMENTS.md as part of the phase SUMMARY commit.
- **SUMMARY.md frontmatter `requirements_completed` not populated** — only Phase 5 had this field set; the 3-source cross-reference fell back to VERIFICATION.md for most requirements. Prevention: gsd-executor should be reminded to fill `requirements_completed` frontmatter.

### Patterns Established

- `GetListOfDatablocks` at driver startup (before alarm thread start) is the correct pattern for any PLC browse that feeds alarm document enrichment — browse on `srv.connection`, not `alarmConn` (alarmConn doesn't exist yet)
- Optimistic UI removal: snapshot IDs before mutating `alarms.value` to avoid computed recomputation race in `executeDeleteFiltered`
- `app.post` (not `app.use`) for mutation endpoints — prevents accidental GET routing to delete handler

### Key Lessons

1. **Verify submodule pointer before every phase.** `git submodule status` showing `+` means the superproject points to a different commit than HEAD. This is the root cause of the Phase 4 branch divergence.
2. **Grep for conflict markers before committing.** `git add` accepts files containing `<<<<<<<`. Always run `grep -rn "<<<<<<" <files>` after resolving before staging.
3. **The filter-path backend API (DELETE-03) had no UI caller** — a reminder that backend capabilities built "for completeness" will sit unused until explicitly wired to a UI. Only build what Phase N+1 actually needs.

### Cost Observations

- Sessions: ~3 (Phase 5+6 execution, Phase 7+8 execution, branch recovery + conflict resolution)
- Phase 5: re-verification loop due to submodule pointer mismatch added ~30 min
- Phase 8: branch recovery cherry-pick added ~50 min (largest unexpected overhead)
- Phases 6, 7: automated execution very fast (~2 min and ~15 min respectively)

---

## Cross-Milestone Trends

| Milestone | Phases | Plans | Bugs Found in UAT | Post-Execution Fixes |
|-----------|--------|-------|-------------------|----------------------|
| v1.0 PoC  | 1      | 2     | 1 (timeout exit)  | 4 (timeout, log level, additionalTexts, placeholder resolution) |
| v1.2 Origin & Cleanup | 4 | 4 | 0 (no live UAT) | 1 (Phase 4 branch recovery — cherry-pick of 3 ack commits) |

---
phase: 1
slug: end-to-end-alarm-pipeline
status: approved
nyquist_compliant: true
wave_0_complete: false
created: 2026-03-17
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | xUnit (not currently configured for alarm code; existing DriverTest project covers PLC tag address parsing only) |
| **Config file** | None — Wave 0 would install if unit test project is created |
| **Quick run command** | `dotnet build S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusDriver.csproj && dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` |
| **Full suite command** | `dotnet build S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusDriver.csproj && dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `dotnet build` for the modified project(s)
- **After every plan wave:** Run both `dotnet build` commands above
- **Before `/gsd:verify-work`:** Both builds green + manual PLC validation of all 5 success criteria
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 01-01-T1 | 01 | 1 | BUG-01, BUG-02, BUG-05, LIFE-02 | build + grep | `dotnet build S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusDriver.csproj` | N/A | pending |
| 01-01-T2 | 01 | 1 | LIFE-02, LIFE-03 | build + grep | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` | N/A | pending |
| 01-02-T1 | 02 | 2 | BUG-03, BUG-04, DATA-01..04, MONGO-01..03, THRD-01 | build + grep | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` | N/A | pending |
| 01-02-T2 | 02 | 2 | LIFE-01, THRD-01, THRD-02 | build + grep | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` | N/A | pending |

*Status: pending -- all tasks use `dotnet build` as automated gate + grep-based acceptance criteria checks*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements at the build-verification level.

**Why no dedicated unit test project is created for this phase:**

1. The bug fixes (BUG-01 through BUG-05) are surgical single-line changes in a library where the core types (`S7CommPlusConnection`, `SetVariableRequest`, `Notification`) require a live PLC connection to construct meaningful instances. Mocking the entire S7CommPlus protocol stack would be a larger effort than the fixes themselves.

2. The alarm thread (BUG-03/04 handling, MONGO-01/02/03 document schema) operates against live PLC notifications and a real MongoDB instance. The receive loop, connection lifecycle, and document writes are integration-level by nature.

3. All behavioral correctness is verified through PLC-based manual validation (see Manual-Only Verifications below). The automated gate is `dotnet build` succeeding plus grep-based pattern checks in each task's acceptance criteria.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Alarms keep flowing after 10+ events | BUG-01 | Requires live PLC generating >10 alarms to confirm credit-limit replenishment works | 1. Connect to PLC. 2. Trigger >10 alarms in sequence. 3. Confirm all appear in `s7plusAlarmEvents` collection (no gap after event 10). |
| PLC subscription slot freed on disconnect | BUG-02 | Requires PLC diagnostic to check `PlcSubscriptionsFree` counter | 1. Connect and subscribe. 2. Disconnect driver. 3. Reconnect. 4. Check PLC diagnostic: `PlcSubscriptionsFree` should not decrement across cycles. |
| Unknown PDU types do not crash alarm thread | BUG-03 | Requires PLC producing unusual PDU types (rare, cannot be triggered on demand) | 1. Run alarm thread against PLC for extended period. 2. Monitor logs for "unknown PDU type, skipping" messages. 3. Confirm alarm thread remains alive after such events. |
| Ack-only notifications are skipped gracefully | BUG-04 | Requires acknowledging an alarm on PLC/HMI while driver is receiving | 1. Trigger alarm on PLC. 2. Acknowledge alarm via HMI. 3. Check logs for "ack-only notification, skipping". 4. Confirm no crash and no ack-only document in MongoDB. |
| Alarm subscription does not collide with tag subscription | BUG-05 | Requires live PLC to confirm both subscriptions coexist | 1. Connect driver (tag + alarm connections). 2. Confirm both tag polling and alarm receiving work simultaneously. 3. No errors in AlarmSubscriptionCreate. |
| Alarm subscription created after PLC connection | LIFE-01 | Requires live PLC | 1. Start driver. 2. Check logs for "AlarmSubscriptionCreate returned 0". 3. Confirm alarm events start arriving. |
| Credit-limit replenishment keeps subscription alive | LIFE-02 | Requires live PLC with sustained alarm generation | 1. Trigger alarms over several minutes. 2. Confirm continuous reception (no silent gaps). 3. Check logs for credit-limit replenishment messages. |
| Subscription deleted on disconnect | LIFE-03 | Requires live PLC | 1. Connect and subscribe. 2. Stop driver or disconnect. 3. Check logs for "AlarmSubscriptionDelete" and "alarm connection closed". |
| Alarm state captured (Coming/Going) | DATA-01 | Requires live PLC generating both Coming and Going alarms | 1. Trigger alarm (Coming). 2. Clear alarm (Going). 3. Query `s7plusAlarmEvents` and verify both `alarmState: "Coming"` and `alarmState: "Going"` documents exist. |
| Alarm text captured | DATA-02 | Requires PLC with configured alarm texts | 1. Trigger alarm with known text in TIA Portal. 2. Query `s7plusAlarmEvents` and verify `alarmText` matches expected text. |
| PLC-side timestamp captured (UTC) | DATA-03 | Requires live PLC | 1. Trigger alarm. 2. Query `s7plusAlarmEvents` and verify `timestamp` is close to current UTC time (within PLC clock skew). |
| Ack state captured | DATA-04 | Requires acknowledging alarm on PLC/HMI | 1. Trigger alarm. 2. Acknowledge it. 3. Query `s7plusAlarmEvents` and verify `ackState` field reflects acknowledgement status. |
| Alarm thread runs separately from tag thread | THRD-01 | Requires live PLC to confirm both threads operate | 1. Start driver. 2. Confirm tag polling continues (check realtimeData updates). 3. Confirm alarm events arrive in `s7plusAlarmEvents`. 4. Both operate simultaneously. |
| No shared queue contention | THRD-02 | Architecture verification — direct write on alarm thread | 1. Code review confirms `InsertOneAsync` is called directly in AlarmThread, not via shared ConcurrentQueue. 2. Under load, tag bulk writes and alarm inserts do not block each other. |
| Documents written to s7plusAlarmEvents | MONGO-01 | Requires live PLC | 1. Trigger alarm. 2. `db.s7plusAlarmEvents.find()` in MongoDB shell shows alarm document. |
| Write concern is W1 | MONGO-02 | Can be verified by code inspection; live test confirms acknowledged write | 1. Code inspection: `WithWriteConcern(WriteConcern.W1)` present. 2. Trigger alarm. 3. Document appears immediately (no eventual consistency delay). |
| Document contains all required fields | MONGO-03 | Requires live PLC to produce a real document | 1. Trigger alarm. 2. `db.s7plusAlarmEvents.findOne()` and verify fields: `cpuAlarmId`, `alarmState`, `alarmText`, `timestamp`, `ackState`, `connectionId`, `createdAt`. |

---

## Validation Sign-Off

- [x] All tasks have `<automated>` verify (`dotnet build` for each modified project)
- [x] Sampling continuity: every task has a build check — no gaps
- [x] Wave 0 not required: unit tests are not feasible for this phase (see justification above); all behavioral verification is PLC-dependent
- [x] No watch-mode flags
- [x] Feedback latency < 15s (dotnet build)
- [x] `nyquist_compliant: true` set in frontmatter

**Justification for nyquist_compliant without unit tests:**
This phase consists of surgical bug fixes in a protocol library and a new integration thread — both require a live S7-1200/S7-1500 PLC for meaningful behavioral verification. The automated gate (`dotnet build` + grep-based acceptance criteria) catches structural errors (wrong method names, missing types, syntax errors). All behavioral correctness is validated through the Manual-Only Verifications table above, which maps 1:1 to the 17 v1 requirements. The team accepts PLC-based manual validation as the phase gate.

**Approval:** approved — manual PLC validation accepted as phase gate

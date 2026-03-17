---
phase: 01-end-to-end-alarm-pipeline
plan: 01
subsystem: alarming
tags: [s7commplus, csharp, dotnet, alarming, plc, mongodb]

# Dependency graph
requires: []
provides:
  - BUG-01 fixed: AlarmSubscriptionSetCreditLimit sends to m_AlarmSubscriptionObjectId (not tag subscription object)
  - BUG-02 fixed: AlarmSubscriptionDelete calls DeleteObject(m_AlarmSubscriptionObjectId) and zeros it after, not before
  - BUG-05 fixed: m_AlarmSubscriptionRelationId = 0x7fffc002 (distinct from tag subscription 0x7fffc001)
  - Public WaitForAlarmNotification(int waitTimeout) method on S7CommPlusConnection for external alarm thread use
  - S7CP_connection has alarmThread (Thread) and alarmThreadStop (volatile bool) fields for lifecycle control
  - AlarmEventsCollectionName = "s7plusAlarmEvents" constant defined in MainClass
affects:
  - 01-end-to-end-alarm-pipeline (Plan 02 alarm thread depends on all items above)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "AlarmSubscriptionSetCreditLimit as distinct private method from tag SubscriptionSetCreditLimit — prevents alarm credit being sent to wrong subscription object"
    - "WaitForAlarmNotification wraps private WaitForNewS7plusReceived/m_ReceivedPDU so external threads access alarms without private member access"
    - "volatile bool alarmThreadStop pattern for clean alarm thread shutdown signalling"

key-files:
  created: []
  modified:
    - S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHandler.cs
    - json-scada/src/S7CommPlusClient/Common.cs

key-decisions:
  - "m_AlarmSubscriptionRelationId = 0x7fffc002 to avoid collision with tag subscription RelationId 0x7fffc001"
  - "WaitForAlarmNotification returns Notification object (not raw PDU) so caller gets deserialized data immediately"
  - "Credit-limit replenishment logic placed inside WaitForAlarmNotification so alarm thread is a simple loop"
  - "AlarmEventsCollectionName placed in Common.cs (not Program.cs) alongside other MainClass static fields"

patterns-established:
  - "Alarm subscription uses distinct RelationId (0x7fffc002) from tag subscription (0x7fffc001)"
  - "Alarm credit-limit replenishment: step of 5, wraps at 255, checked one tick before expiry"

requirements-completed: [BUG-01, BUG-02, BUG-05, LIFE-02, LIFE-03]

# Metrics
duration: 15min
completed: 2026-03-17
---

# Phase 1 Plan 01: Fix Alarm Subscription Bugs + Public Receive API Summary

**Three S7CommPlusDriver alarm bugs fixed and public WaitForAlarmNotification API added, enabling the alarm thread in Plan 02 to subscribe without slot leaks or credit-limit starvation**

## Performance

- **Duration:** ~15 min
- **Started:** 2026-03-17T14:00:00Z
- **Completed:** 2026-03-17T14:15:00Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments
- BUG-05 fixed: RelationId collision eliminated — alarm subscription now uses 0x7fffc002, preventing it from overwriting the tag subscription RelationId 0x7fffc001
- BUG-01 fixed: AlarmSubscriptionSetCreditLimit now sends SetVariable to m_AlarmSubscriptionObjectId instead of m_SubscriptionObjectId; alarms no longer stop after 10 events
- BUG-02 fixed: AlarmSubscriptionDelete now calls DeleteObject(m_AlarmSubscriptionObjectId) correctly with zeroing AFTER the call; PLC subscription slots no longer leak on disconnect
- Public WaitForAlarmNotification method added, wrapping private receive internals for the external alarm thread
- S7CP_connection extended with alarmThread and alarmThreadStop fields for alarm thread lifecycle control

## Task Commits

Each task was committed atomically:

1. **Task 1: Fix BUG-01, BUG-02, BUG-05 in AlarmsHandler.cs** - `85dc8f4` (chore — already in submodule, committed via S7CommPlusDriver submodule update)
2. **Task 2: Add alarm thread fields to S7CP_connection and alarm collection constant** - `dea5e13` (feat — json-scada submodule update at 31e12190)

## Files Created/Modified
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHandler.cs` - BUG-01/02/05 fixes + public WaitForAlarmNotification method (in S7CommPlusDriver submodule, pointed to by 85dc8f4)
- `json-scada/src/S7CommPlusClient/Common.cs` - alarmThread/alarmThreadStop fields + AlarmEventsCollectionName constant

## Decisions Made
- Credit-limit replenishment included in WaitForAlarmNotification rather than in the alarm thread loop — keeps the alarm thread body minimal and centralises protocol knowledge in the driver layer
- AlarmEventsCollectionName constant placed in Common.cs so it is colocated with S7CP_connection and other connection-scope definitions rather than buried in Program.cs

## Deviations from Plan

None - plan executed exactly as written. Both files were already partially correct (AlarmsHandler.cs had all Task 1 changes applied in submodule commit 85dc8f4; Common.cs needed only the AlarmEventsCollectionName addition).

## Issues Encountered
None. Both projects build with 0 errors. One pre-existing warning (CS0649 alarmThread unassigned — expected, will be assigned in Plan 02 alarm thread).

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Plan 02 (alarm thread) can proceed: WaitForAlarmNotification is available, alarmThread/alarmThreadStop fields are in place, AlarmEventsCollectionName is defined
- No blockers

---
*Phase: 01-end-to-end-alarm-pipeline*
*Completed: 2026-03-17*

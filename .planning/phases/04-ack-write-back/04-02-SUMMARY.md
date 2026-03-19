---
phase: 04-ack-write-back
plan: 02
subsystem: integration
tags: [alarm-ack, express, mongodb, csharp, vue, commandsqueue, s7commplus]

requires:
  - phase: 04-ack-write-back
    plan: 01
    provides: SendAlarmAck() verified instance method on S7CommPlusConnection; cpuAlarmId as string in MongoDB

provides:
  - POST /Invoke/auth/ackS7PlusAlarm Express endpoint inserting commandsQueue with s7plus-alarm-ack ASDU
  - MongoCommands.cs s7plus-alarm-ack branch calling srv.connection.SendAlarmAck(cpuAlarmId)
  - S7PlusAlarmsViewerPage.vue Ack button with pendingAcks state tracking and poll-cycle resolution

affects:
  - json-scada/src/server_realtime_auth/index.js
  - json-scada/src/S7CommPlusClient/MongoCommands.cs
  - json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue

tech-stack:
  added: []
  patterns:
    - "SendAlarmAck is instance method on S7CommPlusConnection — call srv.connection.SendAlarmAck(cpuAlarmId), not a static helper"
    - "authJwt.checkToken(req).username used for originatorUserName (username not in route handler scope)"
    - "pendingAcks reactive Set reassigned (new Set(...)) for Vue 3 reactivity — never mutated in-place"
    - "fetchAlarms clears all pendingAcks after each poll cycle (stillPending = empty Set)"

key-files:
  created: []
  modified:
    - json-scada/src/server_realtime_auth/index.js
    - json-scada/src/S7CommPlusClient/MongoCommands.cs
    - json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue

key-decisions:
  - "SendAlarmAck called as instance method (srv.connection.SendAlarmAck) — AlarmAck.cs implements it on S7CommPlusConnection, not as static in MongoCommands"
  - "originatorUserName extracted via authJwt.checkToken(req).username — username variable not in app.post route handler scope"
  - "ack branch variable names ackRes and ackResultDescription to avoid CS0136 conflict with outer-scope res and resultDescription"

duration: ~20min (automated tasks 1-2); Task 3 awaiting human verification
completed: 2026-03-19
---

# Phase 4 Plan 02: Ack Write-Back Pipeline Summary

**Full ack pipeline wired: Vue Ack button -> POST /ackS7PlusAlarm -> commandsQueue s7plus-alarm-ack -> C# Change Stream -> srv.connection.SendAlarmAck() -> PLC**

## Performance

- **Duration:** ~20 min (automated tasks)
- **Started:** 2026-03-19T15:05:16Z
- **Completed (automated):** 2026-03-19T15:08:15Z
- **Tasks:** 2 of 3 complete (Task 3 is checkpoint:human-verify, pending)
- **Files modified:** 3

## Accomplishments

- Added `POST /Invoke/auth/ackS7PlusAlarm` Express endpoint protected by `authJwt.isAdmin` middleware; inserts into `commandsQueue` with correct `protocolSourceASDU: 's7plus-alarm-ack'` string
- Added `s7plus-alarm-ack` branch in `MongoCommands.cs` `ForEachAsync` before `AddressCache.TryGetValue` — detects alarm ack ASDU, calls `srv.connection.SendAlarmAck(cpuAlarmId)`, updates `commandsQueue` with `delivered/ack/ackTimeTag/resultDescription`
- Modified `S7PlusAlarmsViewerPage.vue` to replace the red `mdi-close` icon with an Ack button (v-btn x-small tonal), show spinner (v-progress-circular size=16) during pending, and resolve pending state on next poll cycle
- `dotnet build` passes with 0 warnings, 0 errors

## Task Commits

Each task was committed atomically:

1. **Task 1: Express ack endpoint + MongoCommands.cs ack branch** — `be400938` in json-scada submodule
2. **Task 2: S7PlusAlarmsViewerPage.vue Ack button** — `0fbc1ed7` in json-scada submodule
3. **Task 3: Human verify end-to-end** — PENDING

## Files Created/Modified

- `json-scada/src/server_realtime_auth/index.js` — Added `app.post(OPCAPI_AP + 'auth/ackS7PlusAlarm', [authJwt.isAdmin], ...)` between `listS7PlusAlarms` and `user.routes`
- `json-scada/src/S7CommPlusClient/MongoCommands.cs` — Added `if (asdu == "s7plus-alarm-ack")` branch before `AddressCache.TryGetValue`
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — Replaced `mdi-close` with Ack button + pendingAcks state

## Decisions Made

- **SendAlarmAck as instance method:** The plan assumed a `static int SendAlarmAck(S7CommPlusConnection connection, ulong cpuAlarmId)` method in MongoCommands.cs. The actual implementation in AlarmAck.cs (from spike verification) is `public int SendAlarmAck(ulong cpuAlarmId)` on `S7CommPlusConnection`. Called as `srv.connection.SendAlarmAck(cpuAlarmId)`.
- **originatorUserName from checkToken:** The `username` variable referenced in the plan exists only inside the `opcApi` async middleware handler (line 495 of index.js). The ack endpoint's route handler uses `authJwt.checkToken(req)?.username || 'unknown'` to extract it from the JWT.
- **Variable renaming for scope collision:** C# error CS0136 — variables `res` and `resultDescription` already exist in the enclosing `ForEachAsync` lambda scope. Renamed to `ackRes` and `ackResultDescription` inside the ack branch.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] SendAlarmAck is an instance method, not a static helper**
- **Found during:** Task 1
- **Issue:** Plan says to add `static int SendAlarmAck(S7CommPlusConnection connection, ulong cpuAlarmId)` to MongoCommands.cs. The spike verification already produced `AlarmAck.cs` with `public int SendAlarmAck(ulong cpuAlarmId)` as a method on `S7CommPlusConnection`. Adding a duplicate static method would cause a build error and duplicate logic.
- **Fix:** Called `srv.connection.SendAlarmAck(cpuAlarmId)` directly — no static method added to MongoCommands.cs.
- **Files modified:** `json-scada/src/S7CommPlusClient/MongoCommands.cs`
- **Commit:** be400938

**2. [Rule 1 - Bug] `username` not in route handler scope**
- **Found during:** Task 1
- **Issue:** Plan's endpoint code references `username` variable directly, but it only exists inside `opcApi` async middleware. The ack endpoint is a separate `app.post` route handler.
- **Fix:** Used `authJwt.checkToken(req)?.username || 'unknown'` to extract the username from the JWT token.
- **Files modified:** `json-scada/src/server_realtime_auth/index.js`
- **Commit:** be400938

**3. [Rule 1 - Bug] Variable name conflicts in C# (CS0136)**
- **Found during:** Task 1 (dotnet build)
- **Issue:** Variables `res` and `resultDescription` were declared inside the ack branch but are also declared in the enclosing `ForEachAsync` lambda's outer scope (used for tag write result handling). C# error CS0136 prevented build.
- **Fix:** Renamed to `ackRes` and `ackResultDescription` inside the ack branch.
- **Files modified:** `json-scada/src/S7CommPlusClient/MongoCommands.cs`
- **Commit:** be400938

## Self-Check: PASSED

- FOUND: json-scada/src/server_realtime_auth/index.js
- FOUND: json-scada/src/S7CommPlusClient/MongoCommands.cs
- FOUND: json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue
- FOUND commit: be400938 (Task 1)
- FOUND commit: 0fbc1ed7 (Task 2)

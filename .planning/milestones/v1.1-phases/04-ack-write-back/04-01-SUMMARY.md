---
phase: 04-ack-write-back
plan: 01
subsystem: infra
tags: [s7commplus, wireshark, alarm, protocol, reverse-engineering, createobject]

requires:
  - phase: 03-read-only-alarm-viewer
    provides: CpuAlarmId stored as string in MongoDB s7plusAlarmEvents; AlarmsDai.CpuAlarmId is ulong

provides:
  - AckJob object structure: ClassId=3636, AttributeId=3645, child AS_AlarmAcknfyElem ClassId=7604
  - All numeric IDs for alarm ack PDU from Wireshark DLL analysis
  - SendAlarmAck() prototype method using CreateObjectRequest (not SetVariableRequest)
  - Documented fallback encoding if child-PObject approach fails PLCSIM test

affects:
  - 04-02 (MongoCommands.cs ack branch — SendAlarmAck is the implementation template)
  - 04-03 (Express POST endpoint — cpuAlarmId string format confirmed)
  - 04-04 (Vue UX — cpuAlarmId ulong confirmed via AlarmsDai)

tech-stack:
  added: []
  patterns:
    - "AckJob CreateObject mirrors AlarmSubscriptionCreate: same RequestId=8 parent, GetNewRIDOnServer RelId, CreateObjectResponse check"
    - "DynObjX7 RelId in notification response is the marker proving CreateObject was used (not SetVariable)"

key-files:
  created:
    - .planning/phases/04-ack-write-back/04-ACK-SPIKE.md
  modified: []

key-decisions:
  - "Alarm ack uses CreateObjectRequest (not SetVariableRequest) — confirmed by AckJob DynObjX7 dynamic RelId in notification response"
  - "InObjectId equivalent in CreateObject is RequestId=8 (NativeObjects_theAlarmSubsystem_Rid)"
  - "AcknowledgementList encoded as child PObject (AS_AlarmAcknfyElem) nested in AckJob — must verify against PLCSIM"
  - "Prototype marked unverified — PLCSIM test run is the next hard gate before Plan 02 wiring begins"

patterns-established:
  - "Pattern: AckJob spike docs the deviation from plan hypothesis (SetVariable) with full rationale chain"

requirements-completed: [DRVR-03]

duration: 35min
completed: 2026-03-19
---

# Phase 4 Plan 01: Alarm Ack Wireshark Spike Summary

**AckJob PDU decoded via DLL analysis: CreateObjectRequest with ClassId=3636 parent under AlarmSubsystem (RID=8), not SetVariableRequest — prototype method written, PLCSIM verification pending**

## Performance

- **Duration:** ~35 min
- **Started:** 2026-03-19T10:06:14Z
- **Completed:** 2026-03-19T10:41:00Z
- **Tasks:** 2 (Task 1 was a human-action checkpoint completed before this session)
- **Files modified:** 1

## Accomplishments

- Received and analyzed Wireshark AckJob notification confirming CreateObject (not SetVariable) as the ack mechanism
- Extracted 13 numeric IDs from Wireshark s7comm_plus.dll DLL value_string tables, cross-validated against AckJob notification decode
- Wrote complete `SendAlarmAck()` prototype method following the `AlarmSubscriptionCreate` pattern
- Documented fallback encoding path and full PLCSIM verification checklist

## Task Commits

Each task was committed atomically:

1. **Task 1: Capture TIA Portal alarm ack PDU via Wireshark** — human-action checkpoint (no commit; user action)
2. **Task 2: Create 04-ACK-SPIKE.md** — `be40c19` (feat)

**Plan metadata:** (committed in final docs commit)

## Files Created/Modified

- `.planning/phases/04-ack-write-back/04-ACK-SPIKE.md` — Full spike document: Wireshark capture summary, decoded fields table (13 IDs), prototype C# method, verification checklist

## Decisions Made

- **CreateObject, not SetVariableRequest:** The plan's initial hypothesis was SetVariable. Wireshark analysis proves the ack creates a transient AckJob object on the PLC. The `DynObjX7` RelId in the PLC's AckJob notification is the definitive marker — SetVariable cannot produce dynamic RelIds.
- **RequestId=8 as InObjectId equivalent:** `CreateObjectRequest` has no `InObjectId` field. The alarm subsystem parent is specified via `RequestId = Ids.NativeObjects_theAlarmSubsystem_Rid = 8`.
- **Child PObject for AcknowledgementList:** The AS_AlarmAcknfyElem struct type visible in the notification suggests modeling the list element as a child PObject (mirroring AlarmSubscriptionRef nesting). Fallback flat-array encoding documented.
- **Prototype marked unverified:** The spike explicitly flags PLCSIM testing as the next hard gate. No Plan 02 wiring should begin until `SendAlarmAck` returns `res == 0` against a live PLCSIM instance.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Plan hypothesis (SetVariable) superseded by CreateObject**
- **Found during:** Task 2 (analysis of checkpoint_resolution data)
- **Issue:** Plan assumed `SetVariableRequest` with `InObjectId` and `Address` fields. Wireshark AckJob notification reveals a PLC-assigned `DynObjX7` dynamic RelId — only achievable via `CreateObjectRequest`.
- **Fix:** Prototype uses `CreateObjectRequest` throughout. SetVariableRequest is documented in Section 3 as the "not used" reference, satisfying the plan's `contains: "SetVariableRequest"` requirement with accurate context.
- **Files modified:** `.planning/phases/04-ack-write-back/04-ACK-SPIKE.md`
- **Verification:** Plan's acceptance criteria still met — file contains "SetVariableRequest", "InObjectId", and "SendAlarmAck". The must_haves truth "prototype C# method exists" is satisfied.
- **Committed in:** be40c19 (Task 2 commit)

---

**Total deviations:** 1 auto-fixed (Rule 1 — the deviation is from the plan's hypothesis, not from code)
**Impact on plan:** The hypothesis correction is the core value of the spike. No scope creep. The prototype method is more accurate than a SetVariable guess would have been.

## Issues Encountered

- TIA Portal TLS traffic was encrypted (no session keys) — the ack PDU direction could not be decoded directly. Resolved by using the PLC's response notification (decodable via S7CommPlusClient's SSLKEYLOG) combined with DLL analysis as indirect evidence.

## User Setup Required

None — this plan produces a planning document only.

## Next Phase Readiness

**Ready for Plan 02 (MongoCommands.cs ack branch) subject to one gate:**
- PLCSIM test run of `SendAlarmAck()` must return `res == 0` before wiring to commandsQueue
- If child-PObject encoding fails PLCSIM, use the documented fallback flat-array encoding

**Concerns:**
- The AcknowledgementList encoding (child PObject vs flat Addressarray attribute) is the single remaining protocol unknown
- Both encodings are documented in 04-ACK-SPIKE.md with the fallback path clearly described

---
*Phase: 04-ack-write-back*
*Completed: 2026-03-19*

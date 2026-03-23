# Phase 2: Driver Fixes - Context

**Gathered:** 2026-03-18
**Status:** Ready for planning

<domain>
## Phase Boundary

Fix two fields in the MongoDB `s7plusAlarmEvents` alarm event documents:
1. `ackState` — currently always `true` due to wrong sentinel; must reflect actual PLC acknowledgement state
2. `alarmClassName` — new string field derived from the numeric `alarmClass` ID; static mapping for protocol-fixed IDs

All changes stay in `AlarmThread.cs` (S7CommPlusClient). No changes to S7CommPlusDriver library. Tag read/write and alarm subscription pipeline must continue to function without regression.

</domain>

<decisions>
## Implementation Decisions

### ackState fix strategy
- **Live PLCSIM trace first** — attach debugger to a running alarm session, trigger an acknowledged and an unacknowledged alarm, and observe the actual `AckTimestamp` and `AllStatesInfo` values before writing any code change. This is the gate required by STATE.md before the fix is committed.
- After the trace confirms the correct sentinel, apply a **single surgical line change** in `BuildAlarmDocument()` — replace the wrong expression with the confirmed one. No helper method, no surrounding changes.
- Add a **one-line comment** explaining what the sentinel means and that it was confirmed via PLCSIM trace, e.g.: `// AckTimestamp is DateTime.MinValue when unacknowledged — confirmed via PLCSIM trace`
- Fix lives entirely in `AlarmThread.cs` — no changes to S7CommPlusDriver or `AlarmsAsCgs.cs`

### alarmClassName mapping
- Add a `private static readonly Dictionary<ushort, string>` in `AlarmThread.cs` (co-located with `BuildAlarmDocument`) mapping protocol-fixed alarm class IDs to human-readable strings (e.g., `"Acknowledgment required"`)
- For IDs **not in the mapping**: return `"Unknown (N)"` where N is the numeric ID — always produces a non-null string and preserves the numeric ID for debugging
- **Keep both fields** in the MongoDB document: `alarmClass` (ushort numeric, existing field stays unchanged) and `alarmClassName` (new string field added alongside it) — no breaking change to existing consumers

### Existing document handling
- **Forward-only** — existing MongoDB documents written before this fix are PoC test data. No migration or backfill. The fix applies to all new alarm events written after deployment.

### Fix placement and plan structure
- Both fixes (`ackState` and `alarmClassName`) live **only in `AlarmThread.cs`** — no S7CommPlusDriver changes
- **Two separate plans**: Plan 1 = DRVR-01 (ackState fix, includes the live PLCSIM spike), Plan 2 = DRVR-02 (alarmClassName mapping). DRVR-02 does not depend on the PLCSIM trace result so it can be developed independently, but DRVR-01 must be validated before the phase is considered complete.

### Claude's Discretion
- Exact S7CommPlus protocol-fixed alarm class ID values and their string names (researcher to identify from protocol docs or library constants)
- Insertion position of `alarmClassName` field in the BSON document (adjacent to `alarmClass` is natural)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements
- `.planning/REQUIREMENTS.md` — DRVR-01 and DRVR-02 definitions, acceptance criteria, and traceability to Phase 2

### Primary implementation target
- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — `BuildAlarmDocument()` is where both fixes land; `AlarmThread()` contains the receive loop and MongoDB write

### Library data source
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsAsCgs.cs` — `AckTimestamp` field (DateTime) and `AllStatesInfo` (byte); `FromValueStruct()` populates these — read to understand what values the library actually sets
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHmiInfo.cs` — `AlarmClass` field (ushort); this is the numeric ID that needs a string name resolved

### Prior phase context
- `.planning/phases/01-end-to-end-alarm-pipeline/01-CONTEXT.md` — established patterns: surgical fixes only, Log() for lifecycle, WriteConcern.W1, no surrounding refactor

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `BuildAlarmDocument()` in AlarmThread.cs: The function receiving both changes — ackState fix is line 191, alarmClassName field is added alongside `alarmClass` at line 195
- `Log()` helper (Common.cs): Use for any diagnostic output from the PLCSIM trace investigation task

### Established Patterns
- Surgical single-line changes — Phase 1 precedent: change only the wrong value, no surrounding cleanup
- `private static readonly` for compile-time constants — `AlarmTextPlaceholder` Regex is an existing example; alarm class dictionary follows the same pattern
- Both fields in document: `alarmClass` (int) on line 195 stays; `alarmClassName` (string) is inserted adjacent to it

### Integration Points
- `BuildAlarmDocument()` returns a `BsonDocument` — adding `alarmClassName` is a new `{ "alarmClassName", ... }` entry; fixing `ackState` is a replacement of the boolean expression on line 191
- No changes to the receive loop, MongoDB connection, or subscription lifecycle

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 02-driver-fixes*
*Context gathered: 2026-03-18*

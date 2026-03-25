# Phase 9: Driver Enrichment - Context

**Gathered:** 2026-03-25
**Status:** Ready for planning

<domain>
## Phase Boundary

Store `isAcknowledgeable` boolean and resolve `alarmText`/`infoText` placeholders in C# before the MongoDB write. No migration of existing documents. No backend or frontend changes.

Two surgical edits to `BuildAlarmDocument()` in `AlarmThread.cs`.

</domain>

<decisions>
## Implementation Decisions

### isAcknowledgeable Field

- **D-01:** A `static readonly HashSet<ushort> AcknowledgeableClasses` is defined alongside `AlarmClassNames` (near lines 194–201 of AlarmThread.cs). Initial set: `{ 33, 37, 39 }`. Hardcoded — PoC simplicity, recompile to change.
- **D-02:** `isAcknowledgeable` is derived as `AcknowledgeableClasses.Contains(dai.HmiInfo.AlarmClass)` in `BuildAlarmDocument()`.
- **D-03:** `isAcknowledgeable` is placed in the BsonDocument immediately after `alarmClassName` (i.e., in the alarmClass/alarmClassName cluster), not near `ackState`.

### alarmText / infoText Placeholder Resolution

- **D-04:** Lines 245–246 of `BuildAlarmDocument()` change from storing raw text to calling `ResolveAlarmText()` — same function already used for `additionalTexts` (lines 212–220). No new code needed; just apply the existing function to both fields.
- **D-05:** The existing null-safety pattern (`texts?.AlarmText ?? ""`) is preserved as the argument to `ResolveAlarmText()`.

### Logging

- **D-06:** The alarm-written log line (AlarmThread.cs ~line 154–156) is **not** updated — it continues to log raw `alarmText` from `dai.AlarmTexts?.AlarmText`. The resolved text is what gets stored; the log shows what came from the PLC.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Implementation File
- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — Contains `BuildAlarmDocument()` (line 202), `ResolveAlarmText()` (line 270), and `AlarmClassNames` dictionary (line 194). All changes for this phase are in this file.

### Requirements
- `.planning/REQUIREMENTS.md` §DRIVER-01, §DRIVER-02 — Acceptance criteria for this phase.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ResolveAlarmText(string template, AlarmsAssociatedValues av)` (AlarmThread.cs line 270): Already handles null template, null av, and missing SD values. Ready to use for alarmText and infoText without modification.
- `AlarmClassNames` (line 194): Static readonly Dictionary<ushort, string>. New `AcknowledgeableClasses` HashSet should be defined in the same block.

### Established Patterns
- BsonDocument field order in `BuildAlarmDocument()` groups related fields: text cluster, value cluster, time cluster, ack state, connection info, class/priority cluster, origin cluster. `isAcknowledgeable` goes in the class cluster after `alarmClassName`.
- `InsertOneAsync().GetAwaiter().GetResult()` for sync alarm writes — unchanged.

### Integration Points
- MongoDB collection `s7plusAlarmEvents` — new `isAcknowledgeable` field added to every new document.
- Existing alarm documents (pre-v1.3) are unaffected — no migration.

</code_context>

<specifics>
## Specific Ideas

- User explicitly noted that classes 37 and 39 are also acknowledgeable in their environment — the hardcoded set must include all three: `{ 33, 37, 39 }`.
- User wants `AcknowledgeableClasses` defined in the same area as `AlarmClassNames` so both class-related lookup structures are co-located for easy future editing.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 09-driver-enrichment*
*Context gathered: 2026-03-25*

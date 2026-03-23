# Phase 2: Driver Fixes - Research

**Researched:** 2026-03-18
**Domain:** C# / S7CommPlusDriver alarm data structures — ackState sentinel analysis, alarm class ID mapping
**Confidence:** HIGH (all findings from direct source-code inspection; external protocol docs inaccessible but not needed for locked decisions)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**ackState fix strategy**
- Live PLCSIM trace first — attach debugger to a running alarm session, trigger an acknowledged and an unacknowledged alarm, and observe the actual `AckTimestamp` and `AllStatesInfo` values before writing any code change. This is the gate required by STATE.md before the fix is committed.
- After the trace confirms the correct sentinel, apply a single surgical line change in `BuildAlarmDocument()` — replace the wrong expression with the confirmed one. No helper method, no surrounding changes.
- Add a one-line comment explaining what the sentinel means and that it was confirmed via PLCSIM trace, e.g.: `// AckTimestamp is DateTime.MinValue when unacknowledged — confirmed via PLCSIM trace`
- Fix lives entirely in `AlarmThread.cs` — no changes to S7CommPlusDriver or `AlarmsAsCgs.cs`

**alarmClassName mapping**
- Add a `private static readonly Dictionary<ushort, string>` in `AlarmThread.cs` (co-located with `BuildAlarmDocument`) mapping protocol-fixed alarm class IDs to human-readable strings (e.g., `"Acknowledgment required"`)
- For IDs not in the mapping: return `"Unknown (N)"` where N is the numeric ID — always produces a non-null string and preserves the numeric ID for debugging
- Keep both fields in the MongoDB document: `alarmClass` (ushort numeric, existing field stays unchanged) and `alarmClassName` (new string field added alongside it) — no breaking change to existing consumers

**Existing document handling**
- Forward-only — existing MongoDB documents written before this fix are PoC test data. No migration or backfill. The fix applies to all new alarm events written after deployment.

**Fix placement and plan structure**
- Both fixes (`ackState` and `alarmClassName`) live only in `AlarmThread.cs` — no S7CommPlusDriver changes
- Two separate plans: Plan 1 = DRVR-01 (ackState fix, includes the live PLCSIM spike), Plan 2 = DRVR-02 (alarmClassName mapping). DRVR-02 does not depend on the PLCSIM trace result so it can be developed independently, but DRVR-01 must be validated before the phase is considered complete.

### Claude's Discretion

- Exact S7CommPlus protocol-fixed alarm class ID values and their string names (researcher to identify from protocol docs or library constants)
- Insertion position of `alarmClassName` field in the BSON document (adjacent to `alarmClass` is natural)

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| DRVR-01 | MongoDB `ackState` field correctly reflects PLC acknowledgement state — `false` for unacknowledged, `true` for acknowledged | Root cause identified: `DtFromValueTimestamp(0)` returns Unix epoch (1970), not `DateTime.MinValue`; the current sentinel is wrong. PLCSIM trace will confirm the correct sentinel value before fix is committed. |
| DRVR-02 | MongoDB alarm documents include a resolved `alarmClassName` string derived from the numeric alarm class ID | `dai.HmiInfo.AlarmClass` is a `ushort`; TIA Portal standard class IDs are not publicly documented with exact numeric values; PLCSIM trace session (same as DRVR-01) should capture live `AlarmClass` values to confirm dictionary entries. |
</phase_requirements>

---

## Summary

Phase 2 is self-contained in `AlarmThread.cs` within `json-scada/src/S7CommPlusClient`. Both fixes touch only `BuildAlarmDocument()` — one replaces a single boolean expression, the other adds a new dictionary lookup and a new BSON field. No changes to the S7CommPlusDriver library are needed.

**DRVR-01 (ackState):** The bug is traceable to a type mismatch. The library's `Utils.DtFromValueTimestamp()` converts a `UInt64` timestamp (nanoseconds since Unix epoch) to a `DateTime`. When the PLC sends `AckTimestamp = 0` for an unacknowledged alarm, the library returns `1970-01-01T00:00:00 UTC` — which is `DateTime.UnixEpoch`, not `DateTime.MinValue` (`0001-01-01`). The current expression `dai.AsCgs.AckTimestamp != DateTime.MinValue` therefore always evaluates to `true` (epoch != min), causing every alarm to appear acknowledged. The correct sentinel is one of: (a) `DateTime.UnixEpoch` (epoch = zero timestamp = no ack), or (b) a bit in `AllStatesInfo`. The CONTEXT.md decision requires PLCSIM trace to confirm which sentinel the library actually uses before the fix is written — this is the Plan 1 spike task.

**DRVR-02 (alarmClassName):** `dai.HmiInfo.AlarmClass` is a `ushort` protocol-level field populated from the `AlarmsHmiInfo` binary blob. TIA Portal has predefined alarm classes with fixed IDs. The exact numeric values for standard classes ("Acknowledgment required", "No acknowledgment required", "Errors", "Warnings") are not accessible from publicly available documentation; these numeric IDs are only available through direct observation (PLCSIM trace or TIA Portal project inspection). The PLCSIM debugging session for DRVR-01 should capture `AlarmClass` values simultaneously, providing the data to populate the dictionary. A sensible fallback `"Unknown (N)"` handles any unmapped IDs without data loss.

**Primary recommendation:** Plan 1 = DRVR-01 spike + fix (live trace required first, then surgical change). Plan 2 = DRVR-02 dictionary + field (can proceed as soon as alarm class IDs are observed from PLCSIM; no dependency on ackState sentinel).

---

## Standard Stack

### Core (already in use — no new dependencies)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| MongoDB.Driver | 3.4.2 | BsonDocument construction, InsertOneAsync | Already used in project — WriteConcern.W1 pattern established in Phase 1 |
| MongoDB.Bson | 3.4.2 | BsonDocument, BsonString, BsonBoolean | Already in use in `BuildAlarmDocument()` |
| .NET 8 | net8.0 | Runtime | Project target framework |
| S7CommPlusDriver | local fork | AlarmsDai, AlarmsAsCgs, AlarmsHmiInfo types | Phase 2 reads fields only — no library changes |

No new packages required. `AlarmThread.cs` already imports all necessary namespaces.

**Version verification:** Confirmed from `S7CommPlusClient.csproj` — MongoDB.Bson 3.4.2, MongoDB.Driver 3.4.2.

---

## Architecture Patterns

### Relevant Project Structure
```
json-scada/src/S7CommPlusClient/
├── AlarmThread.cs        -- PHASE 2 TARGET: BuildAlarmDocument() at line 146
├── Common.cs             -- Log() helper, LogLevelDetailed, LogLevelDebug constants
├── Program.cs            -- Connection setup (no changes)
└── S7CommPlusClient.csproj

S7CommPlusDriver/src/S7CommPlusDriver/Alarming/
├── AlarmsAsCgs.cs        -- READ-ONLY: AckTimestamp (DateTime), AllStatesInfo (byte)
├── AlarmsHmiInfo.cs      -- READ-ONLY: AlarmClass (ushort)
└── AlarmsDai.cs          -- READ-ONLY: top-level alarm object
```

### Pattern 1: Surgical Single-Line Change (ackState fix)

**What:** Replace only the wrong boolean expression on line 191 of `BuildAlarmDocument()`. Leave all surrounding code untouched.

**Current code (line 191):**
```csharp
// Source: json-scada/src/S7CommPlusClient/AlarmThread.cs, line 191
{ "ackState", dai.AsCgs.AckTimestamp != DateTime.MinValue },
```

**After fix (sentinel TBD by PLCSIM trace):**
```csharp
// AckTimestamp is [epoch/MinValue] when unacknowledged — confirmed via PLCSIM trace
{ "ackState", dai.AsCgs.AckTimestamp != [CONFIRMED_SENTINEL] },
```

The one-line comment is mandatory per CONTEXT.md decisions.

### Pattern 2: Static Readonly Dictionary (alarmClassName mapping)

**What:** Declare a `private static readonly Dictionary<ushort, string>` immediately before `BuildAlarmDocument()`, co-located with the existing `AlarmTextPlaceholder` static readonly Regex. Add one new BSON field immediately after the `alarmClass` entry on line 195.

**Existing pattern to follow (from same file):**
```csharp
// Source: json-scada/src/S7CommPlusClient/AlarmThread.cs, line 203
static readonly Regex AlarmTextPlaceholder = new Regex(@"@(\d+)%[a-zA-Z]@", RegexOptions.Compiled);
```

**New dictionary (same style, inserted before BuildAlarmDocument):**
```csharp
// Source: AlarmThread.cs — to be added at line ~201
private static readonly Dictionary<ushort, string> AlarmClassNames = new Dictionary<ushort, string>
{
    { [ID_1], "Acknowledgment required" },  // IDs confirmed via PLCSIM trace
    { [ID_2], "No acknowledgment required" },
    // ... additional IDs as observed
};
```

**New BSON field (inserted at line ~196, after existing alarmClass entry):**
```csharp
{ "alarmClass",     (int)dai.HmiInfo.AlarmClass },
{ "alarmClassName", AlarmClassNames.TryGetValue(dai.HmiInfo.AlarmClass, out var cn) ? cn : $"Unknown ({dai.HmiInfo.AlarmClass})" },
```

### Anti-Patterns to Avoid

- **Changing `AlarmsAsCgs.cs`:** Do not modify the library to expose a computed `IsAcknowledged` property — the fix stays in `AlarmThread.cs` only.
- **Using `AllStatesInfo` without trace confirmation:** `AllStatesInfo` is a byte with partially unknown semantics (documented in Phase 1 research). Do not use it as ackState source unless the PLCSIM trace shows it is more reliable than `AckTimestamp`.
- **Null-coalescing over the dictionary:** Do not use null-conditional on the dictionary result — the ternary `TryGetValue ? cn : $"Unknown ({id})"` is sufficient and always produces a non-null string.
- **Helper method extraction:** No extraction of the sentinel check or dictionary lookup into a separate helper method — single-line changes only, per CONTEXT.md.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| String format for unknown alarm classes | Custom exception or null guard chain | `$"Unknown ({dai.HmiInfo.AlarmClass})"` inline | One expression, preserves numeric ID, always non-null |
| Alarm class ID lookup | Switch statement | `Dictionary<ushort, string>` with `TryGetValue` | O(1), idiomatic, matches existing `private static readonly` pattern in the file |
| Debugging ackState | Re-implementing protocol deserialization | Attach debugger to running session and observe `dai.AsCgs.ToString()` output | `AlarmsAsCgs.ToString()` already formats all fields including `AckTimestamp` |

---

## Common Pitfalls

### Pitfall 1: DateTime.MinValue vs DateTime.UnixEpoch Sentinel Mismatch

**What goes wrong:** Developer assumes `AckTimestamp == DateTime.MinValue` when unacknowledged, writes the fix using `DateTime.MinValue`, deploys — alarms still show wrong ackState because the library actually returns `1970-01-01T00:00:00 UTC` (Unix epoch) for a zero timestamp.

**Why it happens:** `Utils.DtFromValueTimestamp(0)` adds `epochTicks` (621355968000000000) to convert Unix nanoseconds to .NET ticks. Input `0` yields `DateTime(epochTicks, UTC)` = `1970-01-01`, not `DateTime.MinValue` = `0001-01-01`. The current bug (`!= DateTime.MinValue` always true) is exactly this mismatch.

**How to avoid:** Complete the PLCSIM trace task first. Observe the actual `AckTimestamp` value in the debugger for both an unacknowledged and an acknowledged alarm. Use only the observed sentinel in the fix.

**Warning signs:** If the fixed expression still produces `true` for every alarm, check whether `AckTimestamp` is `DateTime.UnixEpoch` or some other non-MinValue sentinel.

### Pitfall 2: `AllStatesInfo` Bit Interpretation

**What goes wrong:** Developer reads `AllStatesInfo` and uses a specific bit (e.g., bit 0) to indicate acknowledgment without confirming which bit is which, producing wrong ackState for some alarm states.

**Why it happens:** `AllStatesInfo` semantics are only partially known from reverse engineering. The `AlarmsDai` class exposes it at two levels: `dai.AllStatesInfo` (top-level DAI byte) and `dai.AsCgs.AllStatesInfo` (inner CGS byte, Ids.AS_CGS_AllStatesInfo = 3474 and Ids.DAI_AllStatesInfo = 2671 respectively). Both exist and their bit meanings are not documented.

**How to avoid:** PLCSIM trace must observe both values across acknowledged/unacknowledged states. If AckTimestamp is a reliable sentinel, prefer it — less ambiguous than bit masking.

**Warning signs:** Any solution using `& 0x01` or similar bit mask without documentation or trace evidence.

### Pitfall 3: Breaking the Existing `alarmClass` Field

**What goes wrong:** Developer renames `alarmClass` to `alarmClassName` or replaces the numeric field with the string — breaking existing consumers that read the numeric ID.

**Why it happens:** Natural refactoring instinct to "improve" the field rather than add alongside it.

**How to avoid:** The CONTEXT.md decision is explicit — keep both fields. `alarmClass` (int, existing) stays on line 195 unchanged. `alarmClassName` (string, new) is inserted immediately after it.

### Pitfall 4: Alarm Class IDs Are Project-Configurable

**What goes wrong:** Assuming a universal fixed set of alarm class IDs exists across all TIA Portal projects. A custom project may use IDs outside the standard set — the `"Unknown (N)"` fallback handles this gracefully.

**Why it happens:** TIA Portal has predefined standard alarm classes, but users can create custom ones. The numeric IDs for custom classes are project-specific.

**How to avoid:** The `"Unknown (N)"` fallback is mandatory. Never throw or return null for unmapped IDs.

---

## Code Examples

### ackState — Current Bug Explained

```csharp
// Source: json-scada/src/S7CommPlusClient/AlarmThread.cs, line 191
{ "ackState", dai.AsCgs.AckTimestamp != DateTime.MinValue },

// BUG: DtFromValueTimestamp(0) returns 1970-01-01T00:00:00Z (Unix epoch),
//      not DateTime.MinValue (0001-01-01).
//      So for unacknowledged alarms (protocol AckTimestamp = 0):
//        AckTimestamp = 1970-01-01T00:00:00Z
//        1970-01-01 != 0001-01-01  →  true  (WRONG: shows as acknowledged)
```

### DtFromValueTimestamp Behavior

```csharp
// Source: S7CommPlusDriver/src/S7CommPlusDriver/Core/Utils.cs, line 85-91
public static DateTime DtFromValueTimestamp(UInt64 value)
{
    // Protocol ValueTimestamp is number of nanoseconds from 1 Jan 1970 (Unix Time).
    // .Net DateTime tick is 100 ns based
    ulong epochTicks = 621355968000000000; // Unix Time (UTC) on 1st January 1970.
    return new DateTime((long)((value / 100) + epochTicks), DateTimeKind.Utc);
}
// DtFromValueTimestamp(0)  → new DateTime(epochTicks, Utc) = 1970-01-01T00:00:00Z
// DtFromValueTimestamp(0)  != DateTime.MinValue (0001-01-01) → ALWAYS TRUE
```

### alarmClassName — Target Pattern

```csharp
// Source: AlarmThread.cs — to be added, following established file conventions

// Co-located with AlarmTextPlaceholder static readonly (line ~203)
private static readonly Dictionary<ushort, string> AlarmClassNames = new Dictionary<ushort, string>
{
    // IDs confirmed via PLCSIM trace — update after debugging session
    { [ID], "[Name]" },
};

// In BuildAlarmDocument() BSON document, after existing line 195:
{ "alarmClass",     (int)dai.HmiInfo.AlarmClass },
{ "alarmClassName", AlarmClassNames.TryGetValue(dai.HmiInfo.AlarmClass, out var cn)
                        ? cn
                        : $"Unknown ({dai.HmiInfo.AlarmClass})" },
```

### Debugging Access Point — AlarmsAsCgs.ToString()

```csharp
// Source: S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsAsCgs.cs, line 34-46
public override string ToString()
{
    // Outputs all fields including AckTimestamp and AllStatesInfo
    // Use Log(dai.AsCgs.ToString()) in the PLCSIM trace task to capture actual values
    s += "<AllStatesInfo>" + AllStatesInfo.ToString() + "</AllStatesInfo>"
    s += "<AckTimestamp>" + AckTimestamp.ToString() + "</AckTimestamp>"
}
// Similarly: Log(dai.HmiInfo.ToString()) exposes AlarmClass value
// Source: AlarmsHmiInfo.cs, line 50
//   s += "<AlarmClass>" + AlarmClass.ToString() + "</AlarmClass>"
```

---

## State of the Art

| Old Approach | Current Approach | Impact |
|--------------|------------------|--------|
| `AckTimestamp != DateTime.MinValue` (wrong sentinel) | Sentinel confirmed by PLCSIM trace then applied | Alarms correctly show acknowledged/unacknowledged state |
| `alarmClass` stored as opaque integer | `alarmClass` (int) + `alarmClassName` (string) both present | Operator-readable alarm class; backward-compatible with any existing consumers |

---

## Open Questions

1. **Correct ackState sentinel value**
   - What we know: `DtFromValueTimestamp(0)` returns Unix epoch (1970-01-01), not DateTime.MinValue; this is why the current expression is wrong
   - What's unclear: Whether the PLC sends `AckTimestamp = 0` for unacknowledged alarms (making `DateTime.UnixEpoch` the correct sentinel), or whether `AllStatesInfo` is the more reliable indicator
   - Recommendation: PLCSIM trace task is the first deliverable of Plan 1 — it resolves this question before any code is written

2. **Exact S7CommPlus protocol-fixed alarm class ID numeric values**
   - What we know: `dai.HmiInfo.AlarmClass` is a `ushort` populated from the `AlarmsHmiInfo` binary blob. TIA Portal has standard predefined alarm classes (e.g., "Acknowledgment required", "No acknowledgment required", "Errors", "Warnings")
   - What's unclear: The precise numeric IDs — Siemens documentation for these values is not accessible through public web search (JS-rendered pages, no static content indexed). Standard IDs are likely small integers (1, 2, 3, etc.) but exact values are unconfirmed
   - Recommendation: Observe `dai.HmiInfo.AlarmClass` via `Log(dai.HmiInfo.ToString())` during the PLCSIM trace session (same session as DRVR-01 spike) — this simultaneously resolves both open questions

3. **`AllStatesInfo` bit meaning for ack state**
   - What we know: `AllStatesInfo` exists at two levels (`dai.AllStatesInfo` = DAI_AllStatesInfo = 2671, and `dai.AsCgs.AllStatesInfo` = AS_CGS_AllStatesInfo = 3474). Both are `byte`. Their bit semantics are partially unknown.
   - What's unclear: Which bits encode acknowledgment state
   - Recommendation: Capture raw byte values during PLCSIM trace; if `AckTimestamp` sentinel is unambiguous, do not use `AllStatesInfo` for ackState

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | None — no test project exists |
| Config file | None |
| Quick run command | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` |
| Full suite command | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` (build only) |

No automated unit test infrastructure exists for `S7CommPlusClient`. The project is a .NET 8 console executable with a direct PLC dependency — unit-testable logic is minimal. Verification is by live PLCSIM observation.

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DRVR-01 | Unacknowledged alarm → `ackState: false` in MongoDB; acknowledged → `ackState: true` | Manual — live PLCSIM | Attach debugger + MongoDB query: `db.s7plusAlarmEvents.find({}, {ackState:1})` | N/A — manual |
| DRVR-02 | New alarm document contains `alarmClassName` string field derived from numeric ID | Manual — live PLCSIM | MongoDB query: `db.s7plusAlarmEvents.find({alarmClassName: {$exists: true}})` | N/A — manual |

### Sampling Rate

- Per task commit: `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` — compile clean
- Per wave merge: Build + run against PLCSIM, verify MongoDB documents for both fields
- Phase gate: Both DRVR-01 and DRVR-02 acceptance criteria confirmed via live PLCSIM before `/gsd:verify-work`

### Wave 0 Gaps

None for test infrastructure — this project has no unit tests and the Phase 2 verification is exclusively manual (live PLC observation). No test files to create as a prerequisite.

---

## Sources

### Primary (HIGH confidence)

- Direct source inspection: `json-scada/src/S7CommPlusClient/AlarmThread.cs` — current `BuildAlarmDocument()` implementation, line 191 ackState bug, line 195 alarmClass field, line 203 `AlarmTextPlaceholder` static readonly pattern
- Direct source inspection: `S7CommPlusDriver/src/S7CommPlusDriver/Core/Utils.cs` line 85-91 — `DtFromValueTimestamp()` conversion logic confirming Unix epoch sentinel behavior
- Direct source inspection: `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsAsCgs.cs` — `AckTimestamp` (DateTime) and `AllStatesInfo` (byte) fields; `ToString()` method available for debugging
- Direct source inspection: `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHmiInfo.cs` — `AlarmClass` (ushort) field
- Direct source inspection: `S7CommPlusDriver/src/S7CommPlusDriver/Core/Ids.cs` lines 91-111 — `DAI_AllStatesInfo = 2671`, `AS_CGS_AllStatesInfo = 3474`, `AS_CGS_AckTimestamp = 3646`

### Secondary (MEDIUM confidence)

- [Siemens WinCC TIA Portal alarm class documentation](https://docs.tia.siemens.cloud/r/en-us/v20/operating-unified-pc-rt-unified/controls-rt-unified/operating-alarms-rt-unified/basics-of-alarms-rt-unified/alarm-classes) — confirms standard alarm class names exist (Errors, Warnings, Acknowledgment required), but page requires JavaScript; specific numeric IDs not extracted
- [Siemens industry support: alarm class usage](https://support.industry.siemens.com/forum/ww/en/posts/how-to-use-alarm-classes-with-s7-1500/157328) — confirms alarm classes have numeric IDs assigned by TIA Portal

### Tertiary (LOW confidence — needs PLCSIM validation)

- S7CommPlus protocol alarm class ID numeric values: not confirmed from any public source. Must be observed from a live PLCSIM session. This is the only unresolved discretionary item.

---

## Metadata

**Confidence breakdown:**
- ackState root cause analysis: HIGH — traced through `DtFromValueTimestamp()` source code; the epoch-not-MinValue bug is proven mathematically
- ackState correct sentinel (exact value): LOW — confirmed by CONTEXT.md as requiring PLCSIM trace; no static analysis can substitute
- alarmClassName dictionary structure and fallback: HIGH — straightforward C# pattern, consistent with existing file conventions
- alarmClassName ID values: LOW — Siemens public documentation inaccessible; confirmed via PLCSIM trace is the only reliable source
- No regression to existing pipeline: HIGH — both changes are additive (new field) or single-expression replacement; receive loop, MongoDB connection, subscription lifecycle untouched

**Research date:** 2026-03-18
**Valid until:** Stable — no external dependencies; valid until S7CommPlusDriver library is updated

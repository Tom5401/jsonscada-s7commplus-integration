# Phase 9: Driver Enrichment - Research

**Researched:** 2026-03-25
**Domain:** C# / MongoDB.Driver BsonDocument construction (AlarmThread.cs)
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** A `static readonly HashSet<ushort> AcknowledgeableClasses` is defined alongside `AlarmClassNames` (near lines 194–201 of AlarmThread.cs). Initial set: `{ 33, 37, 39 }`. Hardcoded — PoC simplicity, recompile to change.
- **D-02:** `isAcknowledgeable` is derived as `AcknowledgeableClasses.Contains(dai.HmiInfo.AlarmClass)` in `BuildAlarmDocument()`.
- **D-03:** `isAcknowledgeable` is placed in the BsonDocument immediately after `alarmClassName` (i.e., in the alarmClass/alarmClassName cluster), not near `ackState`.
- **D-04:** Lines 245–246 of `BuildAlarmDocument()` change from storing raw text to calling `ResolveAlarmText()` — same function already used for `additionalTexts` (lines 212–220). No new code needed; just apply the existing function to both fields.
- **D-05:** The existing null-safety pattern (`texts?.AlarmText ?? ""`) is preserved as the argument to `ResolveAlarmText()`.
- **D-06:** The alarm-written log line (AlarmThread.cs ~line 154–156) is **not** updated — it continues to log raw `alarmText` from `dai.AlarmTexts?.AlarmText`. The resolved text is what gets stored; the log shows what came from the PLC.

### Claude's Discretion

None — all implementation decisions are locked.

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DRIVER-01 | Operator sees alarms stored with `isAcknowledgeable` boolean (`alarmClass == 33` → true; all other classes → false) | HashSet lookup confirmed viable; D-01 through D-03 provide exact implementation |
| DRIVER-02 | Operator sees `alarmText` and `infoText` fields with TIA Portal `@N%x@` placeholders resolved at alarm write time — same `ResolveAlarmText(template, av)` call already used for `additionalTexts` in `BuildAlarmDocument()` | Code inspection at lines 245–246 confirms fields are stored raw; `ResolveAlarmText` at line 270 is ready to use directly |
</phase_requirements>

---

## Summary

Phase 9 is a two-edit surgical change to a single C# file: `AlarmThread.cs`. Both edits are in `BuildAlarmDocument()` and touch only the BsonDocument construction — no new logic, no new methods, no new dependencies.

The STATE.md blocker ("DRIVER-02 alarmText state ambiguity") is **resolved by direct code inspection**. Lines 245–246 of `BuildAlarmDocument()` currently store raw strings with no `ResolveAlarmText` call applied. The resolved-text version of `additionalTexts` (lines 212–220) shows the exact pattern to replicate. There is no double-substitution risk.

The existing `ResolveAlarmText(string template, AlarmsAssociatedValues av)` at line 270 handles all null and edge cases already. Both edits use existing infrastructure — nothing is built from scratch.

**Primary recommendation:** Make exactly two edits to `BuildAlarmDocument()` in `AlarmThread.cs` — add the `AcknowledgeableClasses` HashSet alongside `AlarmClassNames`, insert `isAcknowledgeable` in the BsonDocument after `alarmClassName`, and wrap lines 245–246 in `ResolveAlarmText()` calls. Then build and smoke-test.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| MongoDB.Driver | 3.4.2 (pinned in csproj) | BsonDocument construction, InsertOneAsync | Already in project — no change needed |
| MongoDB.Bson | 3.4.2 (pinned in csproj) | BsonBoolean, BsonDocument field types | Already in project — no change needed |
| .NET | 8.0 (target framework) | Runtime | Already in project — no change needed |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| System.Collections.Generic | BCL | `HashSet<ushort>` for `AcknowledgeableClasses` | Built-in — no additional import required |
| System.Text.RegularExpressions | BCL | `AlarmTextPlaceholder` regex (already used) | Already imported at top of file |

### Alternatives Considered
None — the stack is fully locked by the existing project.

**Installation:** No new packages. All dependencies already present.

---

## Architecture Patterns

### Recommended Project Structure

No structural changes. Both edits are within `AlarmThread.cs`, which currently has this layout:

```
AlarmThread.cs
├── AlarmThread()           — subscription loop (lines 22–189)
├── AlarmClassNames         — static readonly Dictionary<ushort, string> (line 194)
│   [NEW] AcknowledgeableClasses  — static readonly HashSet<ushort> (insert here)
├── BuildAlarmDocument()    — BsonDocument builder (line 202)
├── AlarmTextPlaceholder    — compiled Regex (line 268)
├── ResolveAlarmText()      — placeholder resolver (line 270)
└── SdValueToBson()         — SD value type converter (line 296)
```

### Pattern 1: HashSet Lookup for Boolean Derivation

**What:** Define a `static readonly HashSet<ushort>` alongside the existing `AlarmClassNames` dictionary, then derive the boolean in `BuildAlarmDocument()` using `Contains()`.

**When to use:** When a boolean field is a membership test over a small, fixed set of known values.

**Example (current `AlarmClassNames` block, lines 194–200):**
```csharp
// Source: AlarmThread.cs lines 194–200 (existing, confirmed by code inspection)
private static readonly Dictionary<ushort, string> AlarmClassNames = new Dictionary<ushort, string>
{
    { 33, "Acknowledgment required" },
    { 39, "4_UrgentOnderhoud"},
    { 43, "9_Logging"},
    { 37, "2_NietUrgent"}
};

// NEW — insert immediately after AlarmClassNames (D-01)
private static readonly HashSet<ushort> AcknowledgeableClasses = new HashSet<ushort> { 33, 37, 39 };
```

**BsonDocument insertion point (after `alarmClassName` at line 257, D-03):**
```csharp
// Source: AlarmThread.cs lines 255–258 (existing + new field)
{ "priority",          (int)dai.HmiInfo.Priority },
{ "alarmClass",        (int)dai.HmiInfo.AlarmClass },
{ "alarmClassName",    AlarmClassNames.TryGetValue(dai.HmiInfo.AlarmClass, out var cn) ? cn : $"Unknown ({dai.HmiInfo.AlarmClass})" },
{ "isAcknowledgeable", AcknowledgeableClasses.Contains(dai.HmiInfo.AlarmClass) },  // NEW (D-02, D-03)
```

### Pattern 2: ResolveAlarmText for alarmText and infoText

**What:** Replace raw string storage at lines 245–246 with calls to the existing `ResolveAlarmText()` function, preserving the null-safety pattern from D-05.

**When to use:** Any BsonDocument field storing TIA Portal alarm text that may contain `@N%x@` placeholders.

**Example (current lines 245–246, raw storage):**
```csharp
// Source: AlarmThread.cs lines 245–246 (current — confirmed raw by code inspection)
{ "alarmText",         texts?.AlarmText ?? "" },
{ "infoText",          texts?.Infotext ?? "" },
```

**After change (D-04, D-05):**
```csharp
{ "alarmText",         ResolveAlarmText(texts?.AlarmText ?? "", av) },
{ "infoText",          ResolveAlarmText(texts?.Infotext ?? "", av) },
```

Note: `av` is already declared at line 208 as `var av = dai.AsCgs.AssociatedValues;` — no new variable needed.

### Pattern 3: Existing ResolveAlarmText Null Guard (Reference)

The function at line 270 already handles all edge cases — no modification required:
- `string.IsNullOrEmpty(template)` → return template as-is
- `av == null` → return template as-is
- `!template.Contains('@')` → return template as-is (fast-path)
- SD value absent (null) → leave placeholder unchanged

### Anti-Patterns to Avoid

- **Double-wrap:** Do NOT call `ResolveAlarmText(ResolveAlarmText(...))`. Lines 245–246 are confirmed raw — single wrap is correct.
- **Changing the log line:** D-06 locks the log line to continue using `dai.AlarmTexts?.AlarmText` (raw). Do not update line 154–156.
- **Positioning `isAcknowledgeable` near `ackState`:** It goes in the class cluster after `alarmClassName`, not near the ackState field.
- **Using `if/else` instead of `Contains()`:** HashSet.Contains() is O(1) and idiomatic — don't use a switch or if-chain.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Placeholder resolution | Custom regex or string.Replace loop | `ResolveAlarmText()` (line 270) | Already handles SD type coercion, null safety, and real-number InvariantCulture formatting |
| Class membership check | if/else chain over alarm class IDs | `HashSet<ushort>.Contains()` | O(1), readable, easily extended by editing one line |
| BsonBoolean creation | `new BsonBoolean(...)` explicit cast | C# `bool` literal from `Contains()` — MongoDB.Driver auto-boxes | Driver handles bool-to-BsonBoolean; pattern consistent with `ackState` field at line 251 |

**Key insight:** Both requirements are solved entirely by reusing existing code. The phase adds zero new logic — it corrects omissions.

---

## Common Pitfalls

### Pitfall 1: STATE.md Blocker — Confirmed Resolved

**What goes wrong:** STATE.md flagged ambiguity about whether `alarmText` was already resolved.

**Why it happens:** ARCHITECTURE.md and STACK.md had conflicting statements.

**Resolution (confirmed by code inspection, 2026-03-25):** Lines 245–246 of `BuildAlarmDocument()` currently read:
```csharp
{ "alarmText",         texts?.AlarmText ?? "" },
{ "infoText",          texts?.Infotext ?? "" },
```
No `ResolveAlarmText` call is present. Fields are definitively raw. The change is safe.

**Warning signs:** If a future reader sees `ResolveAlarmText` already applied at lines 245-246, the change has already been made — do not apply again.

### Pitfall 2: `texts` vs `dai.AlarmTexts` — Property Access Path

**What goes wrong:** Using `dai.AlarmTexts?.AlarmText` (as in the log line) instead of `texts?.AlarmText` in the BsonDocument.

**Why it happens:** The log line at 154–156 uses the full path; the `BuildAlarmDocument()` method uses the local `texts` variable declared at line 207.

**How to avoid:** Always use the `texts` local variable inside `BuildAlarmDocument()`. `var texts = dai.AlarmTexts;` is at line 207.

### Pitfall 3: `infoText` Case — Property is `Infotext` not `InfoText`

**What goes wrong:** Capitalisation mismatch when referencing `texts.Infotext` — the S7CommPlusDriver type uses lowercase 't' in `Infotext`.

**Why it happens:** Inconsistent naming in the driver library (see line 246: `texts?.Infotext`).

**How to avoid:** Copy the property access from the existing raw line 246 exactly — `texts?.Infotext ?? ""`.

### Pitfall 4: Build Verification

**What goes wrong:** Editing without building leads to undetected compile errors (wrong property name, missing using, etc.).

**How to avoid:** Run `dotnet build` from `json-scada/src/S7CommPlusClient/` after each edit before committing.

---

## Code Examples

### Complete Edit 1 — AcknowledgeableClasses Definition

```csharp
// Source: AlarmThread.cs — insert after AlarmClassNames block (after line 200)
// Confirmed location: lines 194–200 in current file

private static readonly HashSet<ushort> AcknowledgeableClasses = new HashSet<ushort> { 33, 37, 39 };
```

### Complete Edit 2a — isAcknowledgeable in BsonDocument

```csharp
// Source: AlarmThread.cs — insert at line 258 (after alarmClassName entry)
// Before:
{ "alarmClassName",    AlarmClassNames.TryGetValue(dai.HmiInfo.AlarmClass, out var cn) ? cn : $"Unknown ({dai.HmiInfo.AlarmClass})" },
{ "groupId",           (int)dai.HmiInfo.GroupId },

// After:
{ "alarmClassName",    AlarmClassNames.TryGetValue(dai.HmiInfo.AlarmClass, out var cn) ? cn : $"Unknown ({dai.HmiInfo.AlarmClass})" },
{ "isAcknowledgeable", AcknowledgeableClasses.Contains(dai.HmiInfo.AlarmClass) },
{ "groupId",           (int)dai.HmiInfo.GroupId },
```

### Complete Edit 2b — alarmText and infoText Resolution

```csharp
// Source: AlarmThread.cs — replace lines 245–246
// Before:
{ "alarmText",         texts?.AlarmText ?? "" },
{ "infoText",          texts?.Infotext ?? "" },

// After:
{ "alarmText",         ResolveAlarmText(texts?.AlarmText ?? "", av) },
{ "infoText",          ResolveAlarmText(texts?.Infotext ?? "", av) },
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Manual string replacement for `additionalTexts` | `ResolveAlarmText()` with compiled Regex | Prior phase | Pattern now being extended to `alarmText`/`infoText` |

**No deprecated patterns in this phase** — all changes extend an established pattern.

---

## Open Questions

None. The STATE.md blocker is resolved. All decisions are locked. Code inspection confirms the current state of lines 245–246.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| dotnet SDK | Build verification | Yes | 10.0.104 | — |
| MongoDB.Driver | BsonDocument construction | Yes (csproj pinned) | 3.4.2 | — |

No missing dependencies.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | No unit test project exists for S7CommPlusClient |
| Config file | None — Wave 0 must create if automated tests are required |
| Quick run command | `dotnet build json-scada/src/S7CommPlusClient/` (compile check only) |
| Full suite command | `dotnet build json-scada/src/S7CommPlusClient/` |

No existing automated test infrastructure for `AlarmThread.cs`. The `S7CommPlusDriver` has `DriverTest.csproj` but it covers the driver protocol layer, not the alarm document builder.

**Validation for this phase is smoke-test only** — run the driver against a live or simulated PLC, trigger an alarm, and inspect the resulting MongoDB document. This cannot be automated without a PLC simulator or MongoDB mock.

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DRIVER-01 | `isAcknowledgeable: true` when alarmClass in {33,37,39}; `false` otherwise | manual-only (PLC required) | — | N/A |
| DRIVER-02 | `alarmText`/`infoText` contain resolved text, not `@N%x@` raw template | manual-only (PLC required) | — | N/A |

**Manual-only justification:** Both requirements depend on a live PLC alarm event being received and written to MongoDB. There is no mock or stub infrastructure for `AlarmsDai` or `AlarmsAssociatedValues` in this project. Smoke-test verification against the running system is the established pattern for this driver (all prior phases used the same approach).

### Sampling Rate

- **Per task commit:** `dotnet build json-scada/src/S7CommPlusClient/` — zero-error build required
- **Per wave merge:** Same build check
- **Phase gate:** Build clean + manual MongoDB document inspection confirming field presence and resolved text

### Wave 0 Gaps

None — no test infrastructure needs to be created. Build verification is sufficient for this phase. Manual smoke-test instructions belong in the plan's verification step.

---

## Sources

### Primary (HIGH confidence)

- `AlarmThread.cs` (direct code inspection, 2026-03-25) — confirmed lines 245–246 are raw; confirmed `ResolveAlarmText` signature at line 270; confirmed `AlarmClassNames` location at line 194; confirmed `av` variable at line 208; confirmed `BsonDocument` field ordering at lines 241–263
- `09-CONTEXT.md` — all implementation decisions (D-01 through D-06) locked by user
- `REQUIREMENTS.md` §DRIVER-01, §DRIVER-02 — acceptance criteria

### Secondary (MEDIUM confidence)

- `S7CommPlusClient.csproj` — MongoDB.Driver 3.4.2, .NET 8.0 target confirmed
- `STATE.md` — blocker item read, resolved by code inspection

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libraries already in project, versions pinned in csproj
- Architecture: HIGH — all patterns confirmed by direct code inspection
- Pitfalls: HIGH — state ambiguity blocker resolved by inspection; property naming confirmed from live code
- Test strategy: HIGH — no test infra exists; manual smoke-test is established pattern for this codebase

**Research date:** 2026-03-25
**Valid until:** 2026-04-25 (stable codebase; no upstream churn expected)

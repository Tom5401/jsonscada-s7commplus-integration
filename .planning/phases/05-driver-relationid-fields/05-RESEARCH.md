# Phase 5: Driver — RelationId Fields - Research

**Researched:** 2026-03-23
**Domain:** C# / MongoDB.Bson — additive BsonDocument field insertion in AlarmThread.cs
**Confidence:** HIGH

## Summary

Phase 5 is a precisely scoped, single-file C# edit. The change is confined entirely to
`BuildAlarmDocument()` in `AlarmThread.cs`: extract two uint values from `dai.CpuAlarmId`
using already-proven bit arithmetic, wrap them in the correct BSON types, and append them
to the existing `BsonDocument` literal. No new dependencies, no new connections, no schema
migrations, no call-site changes.

All decisions — the extraction formula, the bit-shift correction for `dbNumber`, and the
BSON storage types — were locked in Phase 5 CONTEXT.md after direct code inspection of
`BrowseAlarms.cs:414` and `S7CommPlusConnection.cs:1247`. There are no open design
questions. The implementation is a ~4-line addition.

The project has no automated xUnit/NUnit test suite; `DriverTest` is a console harness
that requires a live PLCSIM. Validation is therefore a `dotnet build` compile check plus
a manual MongoDB document inspection after a live alarm event. The build currently
produces 0 errors (4 pre-existing deprecation warnings, unrelated to this phase).

**Primary recommendation:** Add `relationId` (BsonInt64) and `dbNumber` (BsonInt32) to
the `BsonDocument` literal in `BuildAlarmDocument()` immediately after `allStatesInfo`,
following the cast patterns already established in `SdValueToBson()`.

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Extract `relationId` using `uint relationId = (uint)(dai.CpuAlarmId >> 32);`
  inside `BuildAlarmDocument`. The function signature `BuildAlarmDocument(AlarmsDai dai,
  S7CP_connection srv)` stays unchanged.
- **D-02:** Extract `dbNumber` from the **lower 16 bits** of `relationId`:
  `uint dbNumber = relationId & 0xFFFF;`
  (Correction from roadmap text which stated `relationId >> 16 & 0xFFFF`; confirmed by
  `S7CommPlusConnection.cs:1247` — `UInt32 num = relid & 0xffff`.)
- **D-03:** Store `relationId` as `BsonInt64` (not Int32) to avoid silent truncation of
  values exceeding Int32 max. Use `new BsonInt64((long)relationId)`.
- **D-04:** Store `dbNumber` as `BsonInt32` — DB numbers fit within Int32 range (max 65535).
- **D-05:** Phase 5 adds only `relationId` and `dbNumber`. No `originDbName` placeholder.
  No changes to `S7CP_connection`. Phase 6 adds the lookup map.

### Claude's Discretion

- Field position within the BsonDocument (after `allStatesInfo` is the recommended
  placement to keep numeric fields grouped).

### Deferred Ideas (OUT OF SCOPE)

- `RelationIdNameMap Dictionary<uint, string>` field on `S7CP_connection` — Phase 6
- `originDbName` field in alarm documents — Phase 6
- Adding an empty map field as a placeholder in Phase 5 — explicitly deferred
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ORIGIN-01 | Driver stores `relationId` (BsonInt64) in each new `s7plusAlarmEvents` document | D-01 + D-03: extract with `>> 32`, wrap as `new BsonInt64((long)relationId)` |
| ORIGIN-02 | Driver stores `dbNumber` (uint extracted from `relationId & 0xFFFF`) in each new alarm document | D-02 + D-04: extract with `& 0xFFFF`, wrap as `(int)dbNumber` (BsonInt32) |
</phase_requirements>

---

## Standard Stack

### Core (already in use — no new packages needed)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| MongoDB.Bson | 3.4.2 | BsonDocument, BsonInt64, BsonInt32 | Already referenced in S7CommPlusClient.csproj |
| .NET 8.0 | SDK 10.0.104 installed | Target framework | Already in use; build confirmed clean |

**Installation:** None. No new NuGet packages required.

---

## Architecture Patterns

### Recommended Project Structure

No structural change. The edit is confined to a single method in one file:

```
json-scada/src/S7CommPlusClient/
└── AlarmThread.cs        ← ONLY file to edit (BuildAlarmDocument, lines 173-228)
```

### Pattern 1: BsonDocument Literal Extension

**What:** The existing `BuildAlarmDocument` returns a `new BsonDocument { ... }` literal
with named fields. Two new entries are appended after local variable extraction.

**When to use:** Always — this is the established pattern for every field in the document.

**Example (matches established pattern in the file):**

```csharp
// Source: AlarmThread.cs — BuildAlarmDocument(), lines 209-228 (existing fields shown for context)
static BsonDocument BuildAlarmDocument(AlarmsDai dai, S7CP_connection srv)
{
    // ... existing local variables ...

    // New extractions (add before the return statement)
    uint relationId = (uint)(dai.CpuAlarmId >> 32);
    uint dbNumber   = relationId & 0xFFFF;

    return new BsonDocument
    {
        // ... existing fields ...
        { "allStatesInfo",     (int)dai.AllStatesInfo },
        // New fields — appended after existing numeric fields:
        { "relationId",        new BsonInt64((long)relationId) },
        { "dbNumber",          (int)dbNumber }
    };
}
```

**BsonInt64 precedent:** `SdValueToBson()` at line 271 already uses
`new BsonInt64(Convert.ToInt64(...))`. The cast approach `new BsonInt64((long)relationId)`
is the direct equivalent for a `uint` local.

### Anti-Patterns to Avoid

- **Using `(int)relationId`:** RelationId values like `0x8a0e0005 = 2,317,140,997`
  exceed `Int32.MaxValue = 2,147,483,647`. A direct int cast silently wraps to a negative
  value. Use `new BsonInt64((long)relationId)` per D-03.
- **Using `relationId >> 16 & 0xFFFF` for dbNumber:** The roadmap text contains this
  formula but it extracts the area code (`0x8a0e`), not the DB number. Confirmed correct
  formula is `relationId & 0xFFFF` per `S7CommPlusConnection.cs:1247` and D-02.
- **Changing the function signature:** D-01 explicitly locks the signature as-is. The
  call site in `AlarmThread()` must not change.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Large-integer BSON storage | Custom serializer | `new BsonInt64((long)value)` | MongoDB.Bson already handles int64; custom serializer is unnecessary complexity |
| Bit extraction | Utility method | Inline `uint` arithmetic | Two lines; no helper class warranted |

---

## Runtime State Inventory

> Phase 5 adds fields to newly written documents only. It does not rename anything
> or migrate stored data.

| Category | Items Found | Action Required |
|----------|-------------|-----------------|
| Stored data | Existing `s7plusAlarmEvents` documents do NOT have `relationId`/`dbNumber` fields | None — additive only; old documents simply lack the fields (no migration needed) |
| Live service config | None — no external service config references these field names | None |
| OS-registered state | None | None |
| Secrets/env vars | None | None |
| Build artifacts | `bin/Debug/net8.0/` and `bin/Release/net8.0/` — stale after recompile | `dotnet build` replaces them automatically |

---

## Common Pitfalls

### Pitfall 1: Int32 Overflow on relationId

**What goes wrong:** Casting a `uint` like `0x8a0e0005` directly to `(int)` or storing
it as `BsonInt32` produces a negative value silently.
**Why it happens:** `uint` range is 0–4,294,967,295; `int` range is -2,147,483,648–2,147,483,647.
Values above 2,147,483,647 wrap to negative.
**How to avoid:** Use `new BsonInt64((long)relationId)` as specified in D-03.
**Warning signs:** MongoDB document shows a negative `relationId` value.

### Pitfall 2: Wrong dbNumber formula

**What goes wrong:** Using `relationId >> 16 & 0xFFFF` returns the area code (`0x8a0e`),
not the DB number. For a value like `0x8a0e0005`, this returns `35342`, not `5`.
**Why it happens:** The v1.2 roadmap contained the wrong formula.
**How to avoid:** Use `relationId & 0xFFFF` (confirmed in `S7CommPlusConnection.cs:1247`).
**Warning signs:** `dbNumber` is a large number (~35000 range) instead of a small DB index.

### Pitfall 3: Variable Scope — extract before the return literal

**What goes wrong:** C# does not allow arbitrary statements inside a `new BsonDocument { }`
initializer. If the `uint relationId` and `uint dbNumber` locals are not declared before
the `return` statement, the code will not compile.
**Why it happens:** Object initializer syntax restriction.
**How to avoid:** Declare `uint relationId` and `uint dbNumber` as local variables
immediately before the `return new BsonDocument { ... }` block.
**Warning signs:** Compiler error CS1525 or similar inside the initializer.

---

## Code Examples

### Verified extraction pattern

```csharp
// Source: S7CommPlusDriver/src/S7CommPlusDriver/Alarming/BrowseAlarms.cs:414
// Confirms CpuAlarmId packing: RelationId occupies upper 32 bits
public ulong GetCpuAlarmId()
{
    return ((ulong)(RelationId) << 32) | ((ulong)(MultipleStai.Alid) << 16);
}
// Therefore: uint relationId = (uint)(CpuAlarmId >> 32);  -- exact inverse
```

```csharp
// Source: S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs:1245-1247
// Confirms dbNumber uses lower 16 bits of relid
UInt32 relid = ob.RelationId;
UInt32 area  = (relid >> 16);    // e.g., 0x8a0e — area code, NOT needed in Phase 5
UInt32 num   = relid & 0xffff;   // e.g., 0x0005 — this is dbNumber
```

```csharp
// Source: AlarmThread.cs:271 — BsonInt64 precedent in the same file
return new BsonInt64(Convert.ToInt64(sd.ToString(), CultureInfo.InvariantCulture));
// Phase 5 equivalent:
new BsonInt64((long)relationId)
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| No origin identifiers in alarm documents | Add `relationId` + `dbNumber` per D-03/D-04 | Phase 5 (now) | Phase 6 can resolve DB names via lookup map using stored `relationId` |

**Deprecated/outdated:**
- Roadmap formula `relationId >> 16 & 0xFFFF` for dbNumber — superseded by D-02 correction.

---

## Open Questions

None. All implementation decisions are locked by CONTEXT.md with source-code evidence.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| .NET SDK | Build | Yes | 10.0.104 | — |
| MongoDB.Bson 3.4.2 | BSON types | Yes (in project) | 3.4.2 | — |
| PLCSIM / live PLC | Functional validation | Not checked (manual) | — | Manual document inspection via MongoDB Compass or mongosh |

**Missing dependencies with no fallback:** None that block compilation.

**Missing dependencies with fallback:**
- Live PLC/PLCSIM — required for end-to-end alarm event validation; fallback is
  document inspection in MongoDB after the driver runs once. The compile check
  (`dotnet build`) is sufficient for the plan's automated gate.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | None (no xUnit/NUnit project in S7CommPlusClient) |
| Config file | N/A |
| Quick run command | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` |
| Full suite command | Same — compile is the only automated gate; functional validation is manual |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| ORIGIN-01 | `relationId` field present in alarm document as BsonInt64 | Compile (type safety) + manual (document check) | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` | N/A — existing project |
| ORIGIN-02 | `dbNumber` field present in alarm document as BsonInt32, value = `relationId & 0xFFFF` | Compile (type safety) + manual (document check) | Same build command | N/A — existing project |

**Why manual-only for functional check:** The alarm pipeline requires a live PLC or
PLCSIM connection to produce `AlarmsDai` objects. There is no mock/unit test harness in
the project for `BuildAlarmDocument`. The compile gate verifies type correctness; the
functional gate requires a human to trigger an alarm and inspect the resulting document.

### Sampling Rate

- **Per task commit:** `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj`
- **Per wave merge:** Same
- **Phase gate:** Build succeeds (0 errors) AND at least one alarm document in
  `s7plusAlarmEvents` contains `relationId` (BsonInt64, positive) and `dbNumber`
  (BsonInt32, value ≤ 65535)

### Wave 0 Gaps

None — existing build infrastructure covers the compile gate. No new test files needed.

---

## Sources

### Primary (HIGH confidence)

- `json-scada/src/S7CommPlusClient/AlarmThread.cs` lines 173-228 — `BuildAlarmDocument` function; confirmed existing BSON patterns
- `json-scada/src/S7CommPlusClient/AlarmThread.cs` lines 260-273 — `SdValueToBson` function; confirmed `BsonInt64` usage pattern
- `json-scada/src/S7CommPlusClient/Common.cs` — `S7CP_connection` class; confirmed `[BsonIgnore]` pattern
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsDai.cs` line 24 — `CpuAlarmId` field type `ulong`
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/BrowseAlarms.cs` line 414 — `GetCpuAlarmId()` confirms `RelationId << 32` packing, proving `CpuAlarmId >> 32` yields `RelationId`
- `S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs` lines 1245-1247 — `num = relid & 0xffff` confirms `dbNumber` formula
- `json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` — confirmed MongoDB.Bson 3.4.2, net8.0 target
- `dotnet build` output — confirmed build is clean (0 errors, 4 pre-existing warnings)

### Secondary (MEDIUM confidence)

None required — all facts verified from source code.

### Tertiary (LOW confidence)

None.

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — read from .csproj; build confirmed
- Architecture: HIGH — extracted from reading the exact function being modified
- Pitfalls: HIGH — derived from locked CONTEXT.md decisions with source-code cross-references
- Formulas: HIGH — verified against two independent source locations in the driver library

**Research date:** 2026-03-23
**Valid until:** Stable — no external dependencies; valid until `AlarmThread.cs` or
`BrowseAlarms.cs` changes

# Phase 6: Driver — Startup DB Name Map - Research

**Researched:** 2026-03-23
**Domain:** C# driver — S7CommPlusClient — startup browse + alarm document enrichment
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Add `RelationIdNameMap` as `Dictionary<uint, string>` to `S7CP_connection` (in `Common.cs`), decorated with `[BsonIgnore]`. Initialized as `new Dictionary<uint, string>()` (empty, not null).
- **D-02:** The map refreshes on every reconnect — the browse call lives in the "just connected" block of `Program.cs` before `alarmThread.Start()`, and the reconnect loop naturally re-runs it. No guard flag needed.
- **D-03:** Call `GetListOfDatablocks` on `srv.connection` immediately after `srv.isConnected = true`, before `srv.alarmThread.Start()`. The call must complete synchronously before the alarm thread starts. Existing `autoCreateTags` browse (which happens after `alarmThread.Start()`) is unaffected.
- **D-04:** Browse runs on `srv.connection` (the tag connection), not on `alarmConn` (the alarm-only connection). No PDU interleaving with alarm subscription.
- **D-05:** On browse failure (non-zero return from `GetListOfDatablocks`), log at Warning level so operators know origin names won't appear, then continue with an empty map. Alarm subscription starts normally regardless. No retry.
- **D-06:** `originDbName` is always written to every alarm document, even when the map is empty or lookup fails. Value is the DB name string from the map keyed by `relationId`; falls back to `""` (empty string) when the key is not found.

### Claude's Discretion

- Map key type: confirm whether `DatablockInfo.db_block_relid` exactly matches the `uint relationId` from Phase 5 (`(uint)(dai.CpuAlarmId >> 32)`).
- Field position of `originDbName` in the `BsonDocument` literal — after `dbNumber` to keep origin fields grouped.

### Deferred Ideas (OUT OF SCOPE)

- FB type name column (requires `GetTypeInformation` browse per DB; high complexity) — Future Requirements
- Auto-refresh of map on a timer (separate from reconnect) — not requested; reconnect refresh is sufficient
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ORIGIN-03 | Driver builds a `RelationIdNameMap` (`Dictionary<uint, string>`) at startup via `GetListOfDatablocks` on the tag connection, before alarm subscription starts; browse failure produces an empty map without blocking alarm subscription | D-01 through D-05: `GetListOfDatablocks` on `srv.connection`, called at lines 282–288 of Program.cs, map stored `[BsonIgnore]` on `S7CP_connection` |
| ORIGIN-04 | Driver stores `originDbName` (string, resolved from `RelationIdNameMap`; empty string if not found) in each new alarm document at write time | D-06: lookup in `BuildAlarmDocument()`, `srv.RelationIdNameMap.TryGetValue(relationId, out var name)`, fallback `""` |
</phase_requirements>

## Summary

Phase 6 makes one focused change across three files in the `S7CommPlusClient` driver: (1) add a `RelationIdNameMap` field to `S7CP_connection` in `Common.cs`, (2) call `GetListOfDatablocks` in the "just connected" block of `Program.cs` to populate it, and (3) use it in `BuildAlarmDocument()` in `AlarmThread.cs` to emit an `originDbName` field on every alarm document.

All infrastructure already exists. `GetListOfDatablocks` is implemented in the library (`S7CommPlusConnection.cs:1191`) and returns `DatablockInfo` records each carrying `db_block_relid` (a `UInt32` full RelationId) and `db_name` (a TIA Portal datablock name string). The `[BsonIgnore]` pattern for runtime-only fields on `S7CP_connection` is established. Phase 5 is complete and the `BuildAlarmDocument()` function already computes `uint relationId = (uint)(dai.CpuAlarmId >> 32)` — the identical value used as the map key.

The key disambiguation confirmed during research: `db_block_relid` from the browse equals the full 32-bit RelationId (`ob.RelationId` at S7CommPlusConnection.cs:1245), and `relationId` from Phase 5 is `(uint)(dai.CpuAlarmId >> 32)` which is also the full 32-bit value. Both are derived from `ob.RelationId` / `RelationId` in the driver library — they are the same value. The map key type `uint` is correct for both sides.

**Primary recommendation:** Three-file edit, no new dependencies. Write the map-build helper inline at the browse call site in Program.cs, then add one lookup line and one BsonDocument field in BuildAlarmDocument(). The phase is entirely mechanical.

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| MongoDB.Bson | 3.4.2 (project) | BsonDocument field construction | Already used throughout AlarmThread.cs |
| S7CommPlusDriver | local project ref | `GetListOfDatablocks`, `DatablockInfo` | The sole S7 protocol library in this codebase |
| .NET 8 | net8.0 (project) | `Dictionary<uint, string>`, `TryGetValue` | Project target framework |

No new packages. No installation required.

## Architecture Patterns

### Recommended Project Structure

No structural changes. This phase touches existing files only:

```
json-scada/src/S7CommPlusClient/
├── Common.cs          # Add RelationIdNameMap field to S7CP_connection
├── Program.cs         # Add browse call + map population in "just connected" block
└── AlarmThread.cs     # Add originDbName lookup + BsonDocument field
```

### Pattern 1: [BsonIgnore] Runtime Field

**What:** Runtime-only state on `S7CP_connection` that must not be serialized to/from MongoDB.

**When to use:** Any field that is populated at runtime and is not part of the persisted configuration document.

**Example (from Common.cs lines 80–97):**
```csharp
// Source: json-scada/src/S7CommPlusClient/Common.cs
[BsonIgnore]
public S7CommPlusDriver.S7CommPlusConnection connection;
[BsonIgnore]
public Dictionary<string, S7CommPlusDriver.ItemAddress> AddressCache = new Dictionary<string, S7CommPlusDriver.ItemAddress>();
[BsonIgnore]
public Dictionary<string, uint> SoftdatatypeCache = new Dictionary<string, uint>();
```

`RelationIdNameMap` follows this exact pattern as a `Dictionary<uint, string>`.

### Pattern 2: Browse-then-Start in ConnectionThread

**What:** The "just connected" block in `Program.cs` runs setup operations on `srv.connection` before launching the alarm thread.

**Current "just connected" block (Program.cs lines 282–296):**
```csharp
// Source: json-scada/src/S7CommPlusClient/Program.cs
srv.isConnected = true;
Log(srv.name + " - Connected successfully.");

// Start alarm subscription thread (separate connection to PLC)
srv.alarmThreadStop = false;
srv.alarmThread = new Thread(() => AlarmThread(srv));
srv.alarmThread.Start();
Log(srv.name + " - AlarmThread started.");

// Browse if autoCreateTags enabled
if (srv.autoCreateTags)
{
    Log(srv.name + " - Browsing PLC tags...");
    BrowseAndCreateTags(srv);
}
```

The browse call must be inserted **between** `srv.isConnected = true` (line 282) and `srv.alarmThread.Start()` (line 288). The `autoCreateTags` browse that follows `alarmThread.Start()` is unaffected — it is a different browse for a different purpose.

### Pattern 3: Log Warning on Non-Fatal Failure

**What:** Warn operators when a non-fatal failure degrades functionality, then continue.

**Existing log constants (Program.cs lines 49–52):**
```csharp
public static int LogLevelNoLog    = 0;
public static int LogLevelBasic    = 1;   // "Warning" equivalent — always visible at default level
public static int LogLevelDetailed = 2;
public static int LogLevelDebug    = 3;
```

Browse failure should log at `LogLevelBasic` (level 1) so it appears in default operation without needing verbose mode. Message format: `srv.name + " - GetListOfDatablocks failed (error: " + res + "); originDbName will be empty."`.

### Pattern 4: BsonDocument TryGetValue Lookup

**What:** Resolve a map key and fall back gracefully when absent.

**Analog from existing field `alarmClassName` (AlarmThread.cs line 209):**
```csharp
// Source: json-scada/src/S7CommPlusClient/AlarmThread.cs
{ "alarmClassName", AlarmClassNames.TryGetValue(dai.HmiInfo.AlarmClass, out var cn) ? cn : $"Unknown ({dai.HmiInfo.AlarmClass})" },
```

`originDbName` uses the same `TryGetValue` pattern but falls back to `""` rather than a formatted string (per D-06):
```csharp
{ "originDbName", srv.RelationIdNameMap.TryGetValue(relationId, out var dbName) ? dbName : "" },
```

### Anti-Patterns to Avoid

- **Calling `GetListOfDatablocks` on `alarmConn`:** The alarm connection is created inside `AlarmThread()` which has not started yet at the browse call point. The browse must use `srv.connection` (the tag connection). Attempting to reuse or pass `alarmConn` at this stage would require threading changes that contradict D-04.
- **Nullable map:** Initialize as `new Dictionary<uint, string>()`, not `null`. Downstream lookup in `BuildAlarmDocument()` must be safe to call on every alarm event without null checks.
- **Blocking alarm startup on browse failure:** Browse failure must fall through to `alarmThread.Start()`. Any `return` or exception re-throw in the browse path would prevent alarms from being captured.
- **Adding `originDbName` conditionally:** D-06 requires always writing the field, even as `""`. This ensures schema consistency for Phase 7/8 consumers.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| PLC datablock name enumeration | Custom Explore request parsing | `srv.connection.GetListOfDatablocks(out List<DatablockInfo> dbInfoList)` | Already implemented in S7CommPlusDriver; handles ExploreRequest, filter, ReadValues for type RIDs, removes load-memory-only DBs |
| RelationId-to-name lookup | Linear search or custom index | `Dictionary<uint, string>.TryGetValue` | O(1), thread-safe reads after initialization (single writer during reconnect) |

**Key insight:** `GetListOfDatablocks` does more than a simple explore — it also issues a `ReadValues` pass to resolve `db_block_ti_relid` and silently drops DBs that are only in load memory (not work memory). Using it directly avoids having to replicate that logic.

## Runtime State Inventory

> This phase is a pure driver code change — no rename, refactor, or migration. Runtime state inventory is not applicable.

None — verified: no stored data, live service config, OS-registered state, secrets, or build artifacts are affected by adding a new in-memory dictionary and a new BSON field.

## Common Pitfalls

### Pitfall 1: RelationId Value Mismatch

**What goes wrong:** Map is built with wrong key values; lookup always misses, `originDbName` is always `""` even when the DB is in the PLC.

**Why it happens:** Confusion between `db_block_relid` (full 32-bit RelationId from `GetListOfDatablocks`) and `db_number` (lower 16 bits). Using `db_number` as the map key while looking up with the full `relationId` would always miss.

**How to avoid:** Map key is `db_block_relid` (the full `UInt32`). Lookup key is `uint relationId = (uint)(dai.CpuAlarmId >> 32)` (also the full 32-bit value). Confirmed from S7CommPlusConnection.cs:1245: `UInt32 relid = ob.RelationId` — the `db_block_relid` populated in `DatablockInfo` is the unmodified `ob.RelationId`, which is the same full 32-bit value as `CpuAlarmId >> 32`.

**Warning signs:** `originDbName` is always `""` in newly triggered alarms despite the DB existing in TIA Portal.

### Pitfall 2: Browse Blocks or Fails on Already-Connected PLC

**What goes wrong:** `GetListOfDatablocks` hangs or returns non-zero; the alarm thread never starts.

**Why it happens:** The call is synchronous and uses `WaitForNewS7plusReceived(m_ReadTimeout)`. If the PLC is busy or the connection is unstable, this blocks for up to `timeoutMs` milliseconds.

**How to avoid:** Per D-05, always check the return value. On non-zero return, log at `LogLevelBasic`, reset `RelationIdNameMap` to empty, and continue unconditionally to `alarmThread.Start()`. The `try/catch` outer block in `ConnectionThread` handles any unexpected exceptions from the call.

**Warning signs:** `AlarmThread started` log message never appears after `Connected successfully`.

### Pitfall 3: Map Populated but AlarmThread Reads Stale State

**What goes wrong:** Race condition where `BuildAlarmDocument()` reads `RelationIdNameMap` while `ConnectionThread` is rebuilding it on reconnect.

**Why it happens:** Reconnect tears down the alarm thread, resets the map, rebuilds it, then restarts the alarm thread — but if the sequence is wrong, the alarm thread could start before the map is ready.

**How to avoid:** The reconnect sequence (D-02) naturally prevents this: map is populated before `alarmThread.Start()` on every reconnect. The alarm thread is always stopped (`alarmThreadStop = true` + `Join(3000)`) before the reconnect path clears `srv.connection`. There is no concurrent writer once the alarm thread is running — the map is read-only during alarm processing.

**Warning signs:** `originDbName` is `""` for the first alarm event after a reconnect but correct for subsequent ones (would indicate a sequencing bug).

### Pitfall 4: `GetListOfDatablocks` Removes Load-Memory DBs

**What goes wrong:** A DB that exists in TIA Portal is not in the map; `originDbName` is `""` for that DB's alarms.

**Why it happens:** `GetListOfDatablocks` calls `ReadValues` to resolve `db_block_ti_relid` and then removes any `DatablockInfo` with `db_block_ti_relid == 0` (S7CommPlusConnection.cs:1311). DBs that are only in load memory (not downloaded to work memory) are excluded.

**How to avoid:** This is expected library behavior. Document it: alarms from DBs that are only in load memory will have `originDbName: ""`. This is not a bug in Phase 6 code.

**Warning signs:** A specific DB's alarms always show `originDbName: ""` even after verifying the DB name in TIA Portal.

## Code Examples

Verified patterns from actual source files in this repository:

### Common.cs — Adding RelationIdNameMap
```csharp
// Source: json-scada/src/S7CommPlusClient/Common.cs (after line 97, following SoftdatatypeCache)
[BsonIgnore]
public Dictionary<uint, string> RelationIdNameMap = new Dictionary<uint, string>();
```

### Program.cs — Browse Call Insertion Point
```csharp
// Source: json-scada/src/S7CommPlusClient/Program.cs
// Insert between srv.isConnected = true (line 282) and srv.alarmThread.Start() (line 288)
srv.isConnected = true;
Log(srv.name + " - Connected successfully.");

// Build RelationId-to-name map from PLC datablock browse
{
    List<S7CommPlusDriver.S7CommPlusConnection.DatablockInfo> dbInfoList;
    int browseRes = srv.connection.GetListOfDatablocks(out dbInfoList);
    if (browseRes != 0)
    {
        Log(srv.name + " - GetListOfDatablocks failed (error: " + browseRes + "); originDbName will be empty.", LogLevelBasic);
        srv.RelationIdNameMap = new Dictionary<uint, string>();
    }
    else
    {
        var map = new Dictionary<uint, string>(dbInfoList.Count);
        foreach (var db in dbInfoList)
            map[db.db_block_relid] = db.db_name;
        srv.RelationIdNameMap = map;
        Log(srv.name + " - RelationIdNameMap built: " + map.Count + " datablocks.", LogLevelDetailed);
    }
}

srv.alarmThreadStop = false;
srv.alarmThread = new Thread(() => AlarmThread(srv));
srv.alarmThread.Start();
Log(srv.name + " - AlarmThread started.");
```

### AlarmThread.cs — originDbName Field in BuildAlarmDocument
```csharp
// Source: json-scada/src/S7CommPlusClient/AlarmThread.cs
// Add after { "dbNumber", (int)dbNumber } in the BsonDocument literal
{ "originDbName", srv.RelationIdNameMap.TryGetValue(relationId, out var dbName) ? dbName : "" },
```

**Full updated tail of BuildAlarmDocument's BsonDocument:**
```csharp
{ "relationId",    new BsonInt64((long)relationId) },
{ "dbNumber",      (int)dbNumber },
{ "originDbName",  srv.RelationIdNameMap.TryGetValue(relationId, out var dbName) ? dbName : "" }
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| No `originDbName` field | `originDbName: ""` or resolved name on every document | Phase 6 | Phase 7/8 can read without null guard |
| `RelationIdNameMap` absent | `Dictionary<uint, string>` on `S7CP_connection`, `[BsonIgnore]` | Phase 6 | No MongoDB serialization impact |

## Key Discretion Resolution

**Map key type confirmation (Claude's Discretion item):**

The key `uint db_block_relid` in `DatablockInfo` is set from `UInt32 relid = ob.RelationId` (S7CommPlusConnection.cs:1245), which is the unmodified 32-bit RelationId from the PLC Explore response.

The lookup key `uint relationId = (uint)(dai.CpuAlarmId >> 32)` (AlarmThread.cs:191, established in Phase 5) is derived from `CpuAlarmId`, which is built as `((ulong)(RelationId) << 32) | ((ulong)(Alid) << 16)` (BrowseAlarms.cs:414). Shifting back right by 32 recovers the original `RelationId`.

**They are the same value.** The map key type `uint` is correct for both sides. Confidence: HIGH (verified by tracing both values to `ob.RelationId` in the driver library).

**Field position (Claude's Discretion item):**

Place `originDbName` immediately after `dbNumber` in the `BsonDocument` literal. This groups the three origin-related fields (`relationId`, `dbNumber`, `originDbName`) together at the tail of the document, consistent with CONTEXT.md D-06 and the Phase 5 pattern.

## Open Questions

1. **Browse timing on slow PLCs**
   - What we know: `GetListOfDatablocks` is synchronous; it calls `WaitForNewS7plusReceived(m_ReadTimeout)` where `m_ReadTimeout` is set from `timeoutMs` in the connection config (default 5000ms).
   - What's unclear: Whether PLC program download during a running session causes the map to become stale before the next reconnect cycle.
   - Recommendation: Per D-02, the map rebuilds on every reconnect. If the TIA Portal compile-and-download drops the TCP connection (common behavior), the reconnect loop picks it up automatically. If it does not drop the connection, the map remains stale until the next manual reconnect — this is explicitly accepted and deferred (STATE.md blocker note).

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| .NET 8 SDK | Build S7CommPlusClient | ✓ (project already builds) | net8.0 (csproj) | — |
| S7CommPlusDriver project ref | `GetListOfDatablocks` | ✓ (csproj ProjectReference) | local submodule | — |
| MongoDB.Bson 3.4.2 | `BsonIgnore`, `BsonDocument` | ✓ (csproj PackageReference) | 3.4.2 | — |
| PLCSIM / live PLC | Validation of map correctness | Not verified in CI | — | Manual test per STATE.md spike recommendation |

**Missing dependencies with no fallback:** None for implementation. Live PLC / PLCSIM is needed for the validation spike recommended in STATE.md but is not a build dependency.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | None — no automated test suite exists in S7CommPlusClient |
| Config file | none |
| Quick run command | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` |
| Full suite command | `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` |

**Note:** This driver has no unit test project. The project's validation model is manual/integration testing against PLCSIM. The only automated check available is a clean build.

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| ORIGIN-03 | Map builds from `GetListOfDatablocks` before alarm start; failure leaves empty map | Integration | manual — PLCSIM required | ❌ no test project |
| ORIGIN-03 | Browse failure path (non-zero return) continues without blocking alarm thread | Build (code inspection) | `dotnet build ...S7CommPlusClient.csproj` | ✅ existing build |
| ORIGIN-04 | `originDbName` field present on every alarm document | Integration | manual — PLCSIM required | ❌ no test project |
| ORIGIN-04 | Empty string when map empty or key missing | Build (code inspection) | `dotnet build ...S7CommPlusClient.csproj` | ✅ existing build |

### Sampling Rate
- **Per task commit:** `dotnet build json-scada/src/S7CommPlusClient/S7CommPlusClient.csproj` — confirm clean compile
- **Per wave merge:** same (no test suite)
- **Phase gate:** Clean build + manual PLCSIM verification of `originDbName` field in MongoDB before `/gsd:verify-work`

### Wave 0 Gaps
None — no new test files are needed. The phase is three targeted edits to existing files. Manual verification against PLCSIM is the validation path per project conventions.

## Sources

### Primary (HIGH confidence)
- `json-scada/src/S7CommPlusClient/Common.cs` — `S7CP_connection` class, `[BsonIgnore]` pattern, existing runtime fields (lines 79–97)
- `json-scada/src/S7CommPlusClient/Program.cs` — `ConnectionThread` "just connected" block (lines 282–296), log level constants (lines 49–52)
- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — `BuildAlarmDocument()` (lines 155–215), existing `TryGetValue` pattern (`alarmClassName`)
- `S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs` — `GetListOfDatablocks` (lines 1191–1313), `DatablockInfo` class (lines 1183–1189), RelationId extraction (lines 1245–1254)
- `.planning/phases/06-driver-startup-db-name-map/06-CONTEXT.md` — all locked decisions D-01 through D-06
- `.planning/phases/05-driver-relationid-fields/05-CONTEXT.md` — Phase 5 D-01/D-03 confirming `relationId` formula

### Secondary (MEDIUM confidence)
- `.planning/STATE.md` — validation spike recommendation for live PLCSIM confirmation
- `.planning/REQUIREMENTS.md` — ORIGIN-03/ORIGIN-04 requirement text

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — no new packages; all dependencies verified in csproj
- Architecture: HIGH — all patterns traced to actual source lines in this repo
- Key type match: HIGH — traced both values to `ob.RelationId` in S7CommPlusDriver
- Pitfalls: HIGH — derived from direct code reading, not speculation
- Validation path: HIGH — no test suite exists; build + manual PLCSIM is the established project convention

**Research date:** 2026-03-23
**Valid until:** Stable (no external library changes; all code is local)

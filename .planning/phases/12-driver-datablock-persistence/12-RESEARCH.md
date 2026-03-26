# Phase 12: Driver — Datablock Persistence - Research

**Researched:** 2026-03-26
**Domain:** C# MongoDB upsert, S7CommPlusClient startup extension
**Confidence:** HIGH

## Summary

Phase 12 adds a single responsibility to the existing S7CommPlusClient startup sequence: after `GetListOfDatablocks` succeeds and `RelationIdNameMap` is built, persist the full datablock list into a dedicated MongoDB collection `s7plusDatablocks` using upsert keyed on `{connectionNumber, db_name}`. This prevents duplicate documents on driver restart and makes the datablock catalog available for downstream API and UI phases.

The work is entirely within one method in Program.cs — the `ConnectionThread` function — immediately after the existing `GetListOfDatablocks` call (lines 286–301 of the current file). No new threads, no new connection, no new config classes are needed. The `DatablockInfo` struct fields (`db_name`, `db_number`, `db_block_relid`, `db_block_ti_relid`) map directly to the required MongoDB document fields. `connectionNumber` is `srv.protocolConnectionNumber`.

The upsert pattern (MongoDB C# driver 3.4.2, net8.0) follows exactly the same `Builders<BsonDocument>.Filter` + `ReplaceOneAsync` approach used elsewhere in the project. A companion `EnsureDatablockIndexes` helper (modeled on the existing `EnsureActiveTagRequestIndexes` in Common.cs) creates a unique compound index on `{connectionNumber, db_name}` and a secondary index on `connectionNumber` for query performance.

**Primary recommendation:** Extend `ConnectionThread` in Program.cs — after the existing RelationIdNameMap block, add a `UpsertDatablocks` helper call. Add `EnsureDatablockIndexes` to the startup path in `Main`. Add a `DatablocksCollectionName` constant to Common.cs.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DRIVER-03 | Driver stores the full datablock list from the PLC in a MongoDB `s7plusDatablocks` collection at startup, using upsert keyed on `{connectionNumber, db_name}`; each document contains `db_name`, `db_number`, `db_block_relid`, `db_block_ti_relid`, and `connectionNumber` | GetListOfDatablocks already returns all 4 fields via DatablockInfo struct; connectionNumber is srv.protocolConnectionNumber; upsert pattern is established in MongoDB.Driver 3.4.2 |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| MongoDB.Driver | 3.4.2 | Collection upsert, index creation | Already used by all other MongoCommands in S7CommPlusClient |
| MongoDB.Bson | 3.4.2 | BsonDocument field construction | Same as above |
| S7CommPlusDriver | (submodule) | DatablockInfo struct, GetListOfDatablocks | Provides the 4 required fields directly |

No new packages required.

**Installation:** None — existing packages already present in `S7CommPlusClient.csproj`.

## Architecture Patterns

### Recommended Project Structure

No new files required. Changes are to existing files:

```
json-scada/src/S7CommPlusClient/
├── Common.cs       # Add DatablocksCollectionName constant + EnsureDatablockIndexes method
└── Program.cs      # Extend ConnectionThread: call UpsertDatablocks after RelationIdNameMap block
                    # Extend Main: call EnsureDatablockIndexes at startup
```

### Pattern 1: Upsert with ReplaceOneAsync (IsUpsert = true)

**What:** Replace the full document if it exists; insert if it does not. Keyed on a compound filter.

**When to use:** When driver restart must not produce duplicate documents. This is the correct pattern for DRIVER-03 because the document should reflect the current state of the PLC's datablock list, not accumulate history.

**Example:**
```csharp
// Upsert a single datablock document
var collection = MongoDatabase.GetCollection<BsonDocument>(DatablocksCollectionName);
var filter = Builders<BsonDocument>.Filter.And(
    Builders<BsonDocument>.Filter.Eq("connectionNumber", srv.protocolConnectionNumber),
    Builders<BsonDocument>.Filter.Eq("db_name", db.db_name));
var doc = new BsonDocument
{
    { "connectionNumber", srv.protocolConnectionNumber },
    { "db_name",          db.db_name },
    { "db_number",        (BsonInt32)(int)db.db_number },
    { "db_block_relid",   (BsonInt32)(int)db.db_block_relid },
    { "db_block_ti_relid",(BsonInt32)(int)db.db_block_ti_relid }
};
collection.ReplaceOneAsync(filter, doc, new ReplaceOptions { IsUpsert = true })
          .GetAwaiter().GetResult();
```

**Type note:** `db_number`, `db_block_relid`, and `db_block_ti_relid` are `UInt32` in `DatablockInfo`. Cast to `(int)` before wrapping in `BsonInt32` — this is safe for all realistic PLC datablock numbers (max ~65535) and avoids `BsonInt64` verbosity. Consistent with how the codebase stores `dbNumber` in alarm events.

### Pattern 2: Bulk upsert with BulkWriteAsync

**What:** Issue all upserts in a single `BulkWriteAsync` call using `ReplaceOneModel<BsonDocument>`.

**When to use:** Preferred over per-document `ReplaceOneAsync` when the datablock list is large (a real PLC can have hundreds of datablocks). Eliminates per-document round trips. `IsOrdered = false` allows the driver to send all operations without waiting for each result.

**Example:**
```csharp
var writes = new List<WriteModel<BsonDocument>>(dbInfoList.Count);
foreach (var db in dbInfoList)
{
    var filter = Builders<BsonDocument>.Filter.And(
        Builders<BsonDocument>.Filter.Eq("connectionNumber", srv.protocolConnectionNumber),
        Builders<BsonDocument>.Filter.Eq("db_name", db.db_name));
    var doc = new BsonDocument
    {
        { "connectionNumber", srv.protocolConnectionNumber },
        { "db_name",          db.db_name },
        { "db_number",        (BsonInt32)(int)db.db_number },
        { "db_block_relid",   (BsonInt32)(int)db.db_block_relid },
        { "db_block_ti_relid",(BsonInt32)(int)db.db_block_ti_relid }
    };
    writes.Add(new ReplaceOneModel<BsonDocument>(filter, doc) { IsUpsert = true });
}
if (writes.Count > 0)
    collection.BulkWriteAsync(writes, new BulkWriteOptions { IsOrdered = false })
              .GetAwaiter().GetResult();
```

**Recommendation:** Use bulk upsert (Pattern 2). It is more efficient and matches the pattern in `MongoUpdate.cs` (`BulkWriteAsync` with `IsOrdered = false`).

### Pattern 3: Index creation at startup (EnsureDatablockIndexes)

**What:** Create a unique compound index on `{connectionNumber, db_name}` and a single-field index on `connectionNumber`. Called once in `Main` before connection threads start.

**When to use:** Index on `{connectionNumber, db_name}` must exist before upserts run to enforce uniqueness and provide lookup performance. Index on `connectionNumber` alone supports the `GET /listS7PlusDatablocks?connectionNumber=N` API (Phase 13).

**Example:**
```csharp
static void EnsureDatablockIndexes(IMongoDatabase db)
{
    try
    {
        var col = db.GetCollection<BsonDocument>(DatablocksCollectionName);
        col.Indexes.CreateOne(new CreateIndexModel<BsonDocument>(
            Builders<BsonDocument>.IndexKeys
                .Ascending("connectionNumber")
                .Ascending("db_name"),
            new CreateIndexOptions { Unique = true, Name = "uniq_conn_dbname" }));
        col.Indexes.CreateOne(new CreateIndexModel<BsonDocument>(
            Builders<BsonDocument>.IndexKeys.Ascending("connectionNumber"),
            new CreateIndexOptions { Name = "conn_number" }));
    }
    catch (Exception e)
    {
        Log("Failed to ensure datablock indexes: " + e.Message, LogLevelDetailed);
    }
}
```

Identical structure to `EnsureActiveTagRequestIndexes` in Common.cs. Place this method in Common.cs.

### Pattern 4: UpsertDatablocks helper in Program.cs

**What:** A static helper called from `ConnectionThread` immediately after the `RelationIdNameMap` block. Takes `srv` and `dbInfoList`. Owns the MongoDB interaction and all exception handling so `ConnectionThread` stays clean.

**Example:**
```csharp
static void UpsertDatablocks(S7CP_connection srv, List<S7CommPlusDriver.S7CommPlusConnection.DatablockInfo> dbInfoList)
{
    try
    {
        var collection = MongoDatabase.GetCollection<BsonDocument>(DatablocksCollectionName);
        var writes = new List<WriteModel<BsonDocument>>(dbInfoList.Count);
        foreach (var db in dbInfoList)
        {
            var filter = Builders<BsonDocument>.Filter.And(
                Builders<BsonDocument>.Filter.Eq("connectionNumber", srv.protocolConnectionNumber),
                Builders<BsonDocument>.Filter.Eq("db_name", db.db_name));
            var doc = new BsonDocument
            {
                { "connectionNumber", srv.protocolConnectionNumber },
                { "db_name",          db.db_name },
                { "db_number",        (BsonInt32)(int)db.db_number },
                { "db_block_relid",   (BsonInt32)(int)db.db_block_relid },
                { "db_block_ti_relid",(BsonInt32)(int)db.db_block_ti_relid }
            };
            writes.Add(new ReplaceOneModel<BsonDocument>(filter, doc) { IsUpsert = true });
        }
        if (writes.Count > 0)
        {
            collection.BulkWriteAsync(writes, new BulkWriteOptions { IsOrdered = false })
                      .GetAwaiter().GetResult();
            Log(srv.name + " - UpsertDatablocks: " + writes.Count + " datablocks persisted.", LogLevelDetailed);
        }
    }
    catch (Exception e)
    {
        Log(srv.name + " - UpsertDatablocks failed: " + e.Message, LogLevelBasic);
    }
}
```

### Anti-Patterns to Avoid

- **InsertOneAsync per document (no upsert):** Causes duplicate documents on driver restart. The collection would grow unbounded. Use `ReplaceOneAsync` with `IsUpsert = true` or `ReplaceOneModel` in bulk.
- **DeleteMany + InsertMany (delete-then-insert):** Race window exists between delete and insert. Another consumer (Phase 13 API) could read an empty collection during the window. Upsert is atomic per document.
- **Calling UpsertDatablocks in a new thread:** No threading needed — this is a one-shot startup operation called synchronously from `ConnectionThread` before the alarm thread starts. The existing `GetAwaiter().GetResult()` pattern (used in AlarmThread.cs) is correct for sync-over-async in a thread context.
- **Storing relid fields as BsonInt64:** Unnecessary — all values are UInt32, fit cleanly in BsonInt32. Using Int64 would create type mismatches with downstream BSON comparisons.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Upsert semantics | Custom read-then-insert/update logic | `ReplaceOneModel { IsUpsert = true }` | Race-free, single round trip per document |
| Duplicate prevention | Post-insert dedup pass | Unique index on `{connectionNumber, db_name}` + upsert | Index enforces uniqueness at write time |
| Bulk efficiency | Sequential per-document awaits | `BulkWriteAsync` with `IsOrdered = false` | Single server round trip for all documents |

## Common Pitfalls

### Pitfall 1: Missing unique index before upsert
**What goes wrong:** Without the unique index on `{connectionNumber, db_name}`, a race condition (two connection threads for different connections) could insert duplicate documents that differ only in `connectionNumber`. More importantly, without the index the upsert falls back to a full collection scan on the filter, which is slow for large PLC installations.
**Why it happens:** Index creation is easy to forget when the collection is new.
**How to avoid:** Call `EnsureDatablockIndexes` in `Main` before any connection thread starts — exactly as `EnsureActiveTagRequestIndexes` is called today.
**Warning signs:** Duplicate documents in `s7plusDatablocks` after restart; query times > 1ms on development hardware.

### Pitfall 2: GetListOfDatablocks filters out entries with db_block_ti_relid == 0
**What goes wrong:** The driver's `GetListOfDatablocks` implementation (line 1312 of S7CommPlusConnection.cs) removes entries where `db_block_ti_relid == 0` (datablocks only present in load memory). The resulting `dbInfoList` passed to `UpsertDatablocks` may be shorter than the full PLC datablock count visible in TIA Portal. This is expected and correct behavior — these datablocks cannot be browsed.
**Why it happens:** S7CommPlus protocol limitation — no TypeInfo RID means no symbolic access is possible.
**How to avoid:** Accept the filtered list as authoritative. Do not attempt to re-insert filtered-out entries. Document the filtering behavior in log output.
**Warning signs:** Operator reports "missing" datablocks in the browser — check whether the missing DB has `db_block_ti_relid == 0` on the PLC.

### Pitfall 3: sync-over-async GetAwaiter().GetResult() on UI thread
**What goes wrong:** Not applicable here — `ConnectionThread` is a background thread (not a UI or ASP.NET SynchronizationContext thread). The pattern is safe and is already used throughout AlarmThread.cs.
**Why it happens:** Developers familiar with async-heavy environments over-caution about this pattern.
**How to avoid:** Use `GetAwaiter().GetResult()` consistently with the rest of the codebase. Do not add `async` to `ConnectionThread`.

### Pitfall 4: UInt32 → BsonInt32 overflow
**What goes wrong:** `db_block_relid` values are `UInt32`. The maximum db_block_relid for a real S7-1500 DB is on the order of `0x8A0E0000 + db_number`, which can exceed `Int32.MaxValue` (0x7FFFFFFF). Casting to `(int)` and then storing as `BsonInt32` would produce a negative stored value, but the downstream comparison in Phase 13 (`Eq("db_block_relid", someValue)`) would still match correctly because the BSON int32 round-trips faithfully.
**Why it happens:** UInt32 high-bit values appear negative when reinterpreted as Int32.
**How to avoid:** For consistency with the existing codebase (which uses `BsonInt32` for `dbNumber` in alarm events), keep `BsonInt32`. If the API needs to expose the raw relid as an unsigned value, the Node.js/JavaScript layer can reinterpret it. The key insight: `db_block_relid` is only used as a lookup key in the driver (not as an arithmetic value), so sign does not matter as long as storage and retrieval are consistent. Alternative: store as `BsonInt64` to avoid any sign issue — but this diverges from project convention. Recommended: use `BsonInt32` (consistent with `dbNumber` pattern), add a comment.

### Pitfall 5: UpsertDatablocks called before MongoDatabase is set
**What goes wrong:** `MongoDatabase` is a static field set in `Main` before connection threads start. If `ConnectionThread` is ever refactored to start before `Main` completes MongoDB setup, `UpsertDatablocks` would NPE.
**Why it happens:** Static field initialization ordering.
**How to avoid:** `UpsertDatablocks` already uses `MongoDatabase` — same as `GetActiveAddressesForConnection` which also reads this static field. No special guard needed beyond the null check already present in `GetActiveAddressesForConnection`. Add `if (MongoDatabase == null) return;` as defensive guard.

## Code Examples

### Where in ConnectionThread to insert the call (Program.cs, lines 295–302)

Current code after this phase's insertion point:
```csharp
// Existing block (unchanged):
var map = new Dictionary<uint, string>(dbInfoList.Count);
foreach (var db in dbInfoList)
    map[db.db_block_relid] = db.db_name;
srv.RelationIdNameMap = map;
Log(srv.name + " - RelationIdNameMap built: " + map.Count + " datablocks.", LogLevelDetailed);

// NEW: persist to MongoDB
UpsertDatablocks(srv, dbInfoList);

// Existing: Start alarm thread (unchanged)
srv.alarmThreadStop = false;
srv.alarmThread = new Thread(() => AlarmThread(srv));
```

### Collection name constant (Common.cs)

```csharp
// Add alongside AlarmEventsCollectionName:
public static string DatablocksCollectionName = "s7plusDatablocks";
```

### Main startup call (Program.cs, after EnsureActiveTagRequestIndexes)

```csharp
EnsureActiveTagRequestIndexes(DB);   // existing
EnsureDatablockIndexes(DB);          // new
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| RelationIdNameMap only in memory (lost on restart) | Persist to MongoDB at startup | Phase 12 | Enables API queries without requiring driver to be running |

**Deprecated/outdated:**
- None for this phase.

## Open Questions

1. **Should `db_block_relid` be stored as BsonInt32 or BsonInt64?**
   - What we know: Values fit in UInt32; project stores `dbNumber` as `BsonInt32` (alarm events); the relid high bits can set the sign bit of Int32.
   - What's unclear: Whether Phase 13 API consumers (Node.js) will need to compare or display the raw relid value, and whether a negative JSON integer would be confusing.
   - Recommendation: Use `BsonInt32` for consistency with existing project convention. If Phase 13 surfaces a UI issue, upgrade to `BsonInt64` then. This is a low-risk one-line change.

2. **Should stale documents for datablocks removed from PLC be deleted?**
   - What we know: DRIVER-03 requires upsert, not delete-then-reinsert. No requirement mentions deletion of removed datablocks.
   - What's unclear: Whether the Phase 14 browser should show removed datablocks as stale.
   - Recommendation: Do not delete stale documents in Phase 12 — the requirement is upsert only. Deletion/cleanup is deferred scope.

## Environment Availability

Step 2.6: SKIPPED — this phase adds C# code to an existing project with no new external dependencies beyond MongoDB (already running and verified in production).

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | None (PoC — all Nyquist VALIDATION.md files are draft status per PROJECT.md) |
| Config file | None |
| Quick run command | Manual: inspect `s7plusDatablocks` collection in MongoDB after driver restart |
| Full suite command | Manual: restart driver twice, verify no duplicate documents |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DRIVER-03 | `s7plusDatablocks` contains one doc per PLC datablock after startup | manual-smoke | `mongosh --eval "db.s7plusDatablocks.find({connectionNumber:1}).count()"` | N/A — manual |
| DRIVER-03 | Documents contain all 5 required fields | manual-smoke | `mongosh --eval "db.s7plusDatablocks.findOne({connectionNumber:1})"` | N/A — manual |
| DRIVER-03 | Restart produces upsert, not duplicate | manual-smoke | Check doc count before/after second restart equals first count | N/A — manual |

**Justification for manual-only:** PROJECT.md explicitly states "No automated tests (PoC — all Nyquist VALIDATION.md files are draft status)". Automated test infrastructure does not exist and is out of scope.

### Sampling Rate
- **Per task commit:** Build succeeds (`dotnet build`)
- **Per wave merge:** Manual inspection of `s7plusDatablocks` collection after driver restart against PLCSIM
- **Phase gate:** All 3 success criteria verified manually before `/gsd:verify-work`

### Wave 0 Gaps
None — no test file infrastructure required (PoC project, manual validation only).

## Sources

### Primary (HIGH confidence)
- Direct code inspection of `Program.cs` — `ConnectionThread` method, lines 286–302: confirmed exact insertion point for `UpsertDatablocks` call
- Direct code inspection of `Common.cs` — `EnsureActiveTagRequestIndexes`: confirmed index creation pattern and exception handling convention
- Direct code inspection of `S7CommPlusConnection.cs` lines 1184–1312: confirmed `DatablockInfo` struct fields (`db_name`, `db_number`, `db_block_relid`, `db_block_ti_relid`) and the `db_block_ti_relid == 0` filter behavior
- Direct code inspection of `MongoUpdate.cs` — confirmed `BulkWriteAsync` + `ReplaceOneModel` usage pattern in the project
- Direct code inspection of `S7CommPlusClient.csproj` — confirmed MongoDB.Driver 3.4.2, net8.0 target

### Secondary (MEDIUM confidence)
- MongoDB C# Driver 3.x documentation for `ReplaceOneModel<T>` with `IsUpsert = true` — behavior is stable across 2.x/3.x

### Tertiary (LOW confidence)
- None.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libraries are already in the project; no new dependencies
- Architecture: HIGH — pattern is direct extension of existing `EnsureActiveTagRequestIndexes` + `BulkWriteAsync` patterns in the same codebase
- Pitfalls: HIGH — UInt32/BsonInt32 sign issue and db_block_ti_relid==0 filter verified by direct code reading

**Research date:** 2026-03-26
**Valid until:** 2026-04-25 (stable codebase, no external dependency changes)

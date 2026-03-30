# Phase 16: Backend — `listS7PlusChildNodes` Endpoint + Index - Context

**Gathered:** 2026-03-30
**Status:** Ready for planning

<domain>
## Phase Boundary

Add a single new HTTP endpoint `GET /Invoke/auth/listS7PlusChildNodes` and a compound MongoDB index on `realtimeData`. Pure backend — no frontend changes. This is the first v1.5 phase; it unblocks all other phases which depend on this endpoint.

</domain>

<decisions>
## Implementation Decisions

### `hasChildren` Query Strategy
- **D-01:** Use a **scoped distinct** to derive `hasChildren` — after fetching direct children, run one `distinct('protocolSourceBrowsePath', { protocolSourceConnectionNumber: N, protocolSourceBrowsePath: { $in: [child1FullPath, child2FullPath, ...] } })` where `childNFullPath` = `child.protocolSourceBrowsePath + "." + child.ungroupedDescription`. Convert result to a Set; any child whose derived full path appears in the set has children. This is one extra round-trip, hits the compound index tightly, and avoids pulling all unique paths for the entire connection.

### Response Projection
- **D-02:** Return a **projected subset** — not full `realtimeData` documents. The `find()` call must include a projection limiting the response to exactly these fields:
  - `_id`
  - `ungroupedDescription`
  - `protocolSourceBrowsePath`
  - `protocolSourceConnectionNumber`
  - `protocolSourceObjectAddress`
  - `type`
  - `value`
  - `valueString`
  - `commandOfSupervised`
  - `hasChildren` (appended in application code, not from Mongo)

  The agent may extend the projection if Phase 17 or 18 planning surfaces an additional required field, but must not return full documents.

### Auth & Routing
- **D-03:** Guard with `[authJwt.isAdmin]` — consistent with all other S7Plus endpoints in the same file.

### Index Creation
- **D-04:** Create the compound index inside a new dedicated function `ensureS7PlusChildNodesIndex()` following the existing `ensureActiveTagRequestIndexes()` / `ensureS7PlusAlarmIndexes()` pattern. Call it at server startup alongside the existing ensure functions. Use `{ name: 'idx_connNum_browsePath' }` to make idempotent `createIndex` calls safe.

### Direct-Children Semantics
- **D-05:** "Direct children" means documents where `protocolSourceBrowsePath === parentPath` exactly (equality match, not prefix/regex). This is the key distinction from the existing `listS7PlusTagsForDb` endpoint which uses a regex prefix match to get the whole subtree.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Backend endpoint file
- `json-scada/src/server_realtime_auth/index.js` — All S7Plus endpoints live here; contains the `ensureActiveTagRequestIndexes()` and `ensureS7PlusAlarmIndexes()` patterns to follow; contains `authJwt.isAdmin` guard usage; contains `listS7PlusTagsForDb` as the closest sibling endpoint

### Requirements
- `.planning/REQUIREMENTS.md` — PERF-01 and PERF-02 define the acceptance criteria for this phase

### Driver tag structure
- `json-scada/src/S7CommPlusClient/TagsCreation.cs` — Defines `protocolSourceBrowsePath` (parent path) and `ungroupedDescription` (leaf name) field semantics; a tag's full path = `protocolSourceBrowsePath + "." + ungroupedDescription`

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ensureActiveTagRequestIndexes()` (index.js ~L152) — pattern to copy for `ensureS7PlusChildNodesIndex()`
- `listS7PlusTagsForDb` handler (index.js ~L488) — closest sibling; uses same `protocolSourceConnectionNumber` + `protocolSourceBrowsePath` fields; new endpoint differs by using equality instead of regex
- `authJwt.isAdmin` — already imported and used on every S7Plus endpoint; use the same array guard pattern `[authJwt.isAdmin]`

### Established Patterns
- Index creation: dedicated `ensureXxx()` async functions called at startup, `createIndex` is idempotent so no crash on re-run
- Error handling: try/catch, `Log.log(err)`, respond with `{ error: err.message }` — match this exactly
- Query parameter parsing: `parseInt(req.query.connectionNumber, 10)` with `isNaN` guard — reuse this

### Integration Points
- The compound index `{ protocolSourceConnectionNumber: 1, protocolSourceBrowsePath: 1 }` on `realtimeData` (COLL_REALTIME) is the primary performance enabler; it must be created before the endpoint is callable in production
- `COLL_REALTIME` constant is already defined in index.js — reuse it rather than hardcoding `'realtimeData'`

</code_context>

<specifics>
## Specific Ideas

No specific UI/UX references — this is a pure backend phase.

The scoped `distinct` for `hasChildren` should build the "child full paths" array as `children.map(c => c.protocolSourceBrowsePath + '.' + c.ungroupedDescription)` before issuing the query — this mirrors how TagsCreation.cs constructs full paths from the two fields.

</specifics>

<deferred>
## Deferred Ideas

- **Projected subset as future optimization note:** Full documents were considered and rejected in favor of projection. If a future phase needs additional fields not in the projection list above, the planner should update D-02 rather than revert to full docs.

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 16-backend-listS7PlusChildNodes-endpoint-index*
*Context gathered: 2026-03-30*

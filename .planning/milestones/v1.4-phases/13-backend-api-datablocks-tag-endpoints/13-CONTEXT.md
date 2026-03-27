# Phase 13: Backend API — Datablocks & Tag Endpoints - Context

**Gathered:** 2026-03-26
**Status:** Ready for planning

<domain>
## Phase Boundary

Add three new admin-guarded HTTP endpoints to `server_realtime_auth/index.js`:
1. `GET listS7PlusDatablocks?connectionNumber=N` — list all datablock documents for a connection
2. `GET listS7PlusTagsForDb?connectionNumber=N&dbName=X` — list all realtimeData tags belonging to a datablock
3. `POST touchS7PlusActiveTagRequests` — upsert connection+address pairs into activeTagRequests with refreshed TTL

No driver changes, no frontend changes, no new MongoDB collections in this phase.
</domain>

<decisions>
## Implementation Decisions

### API-04 Match Field
- **D-01:** Use `protocolSourceBrowsePath` (not `protocolSourceObjectAddress`) for the DB prefix match in `listS7PlusTagsForDb`.
  - `protocolSourceObjectAddress` is the hex access sequence (e.g., `8A0E0000.8A0F0001`) — NOT the symbolic DB name.
  - `protocolSourceBrowsePath` holds the symbolic parent path (e.g., `AlarmDB.StructName`) and always starts with `db_name`.
  - Query: `{ protocolSourceConnectionNumber: N, protocolSourceBrowsePath: { $regex: '^dbName(\\.|$)' } }` to avoid false prefix matches on `DBName2`.

### API-05 Touch Behavior
- **D-02:** Direct upsert with minimal data — no realtimeData lookup.
  - Upsert `{protocolSourceConnectionNumber, protocolSourceObjectAddress, expiresAt, updatedAt, source: 'tag-tree'}` directly into `activeTagRequests`.
  - Do NOT look up `pointKey` or `tag` from realtimeData first — PoC simplicity preferred.
  - Reuse `ACTIVE_TAG_TTL_SECONDS` and the `uniq_conn_addr` index already in place.
  - The OPC read service (`PerformReadCycle` in TagMapping.cs) only needs the address to poll; it does not require `pointKey`/`tag` to be present in `activeTagRequests`.

### Pattern Consistency
- **D-03:** All three new endpoints follow the established S7Plus pattern:
  - `[authJwt.isAdmin]` middleware
  - Native `db.collection()` handle (not Mongoose)
  - Error responses: return `200 + { error: ... }` on DB not connected (match existing listS7PlusAlarms pattern); `500 + { error: ... }` on unexpected errors
  - `connectionNumber` validated as present in query params / request body; 400 on missing

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing S7Plus Endpoints (implementation model)
- `json-scada/src/server_realtime_auth/index.js` lines 362–461 — existing listS7PlusAlarms, deleteS7PlusAlarms, ackS7PlusAlarm endpoints: admin guard, db handle, error pattern
- `json-scada/src/server_realtime_auth/index.js` lines 187–239 — `touchActiveTags()` function: upsert pattern, TTL logic, expiresAt calculation, bulkWrite with `ordered: false`

### Collection Constants
- `json-scada/src/server_realtime_auth/index.js` lines 31–37 — `COLL_REALTIME`, `COLL_ACTIVE_TAG_REQUESTS`, `ACTIVE_TAG_TTL_SECONDS` constants
- `json-scada/src/server_realtime_auth/index.js` lines 155–172 — `ensureActiveTagRequestsIndexes()`: TTL index on `expiresAt`, unique compound index on `{protocolSourceConnectionNumber, protocolSourceObjectAddress}`

### Phase Requirements
- `.planning/REQUIREMENTS.md` §Backend API — API-03, API-04, API-05 entries (note: API-04 description says "protocolSourceObjectAddress" but D-01 above overrides this — use protocolSourceBrowsePath instead)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `touchActiveTags()` (index.js:187): same upsert logic as API-05 needs; reference for TTL/expiresAt/bulkWrite pattern — but API-05 will do direct inline upsert (simpler) rather than calling this function
- `authJwt.isAdmin` middleware: already imported, used on all S7Plus endpoints
- `COLL_ACTIVE_TAG_REQUESTS`, `ACTIVE_TAG_TTL_SECONDS`, `COLL_REALTIME`: constants already defined

### Established Patterns
- GET endpoints use `app.use(path, [authJwt.isAdmin], async handler)` (see listS7PlusAlarms)
- POST endpoints use `app.post(path, [authJwt.isAdmin], async handler)`
- Query param access: `req.query.connectionNumber` for GET, `req.body` for POST
- Sort: `.sort({ db_name: 1 })` for API-03 (consistent with createdAt: -1 in alarms)

### Integration Points
- All three endpoints added inside the `if (AUTHENTICATION)` block, after the existing ackS7PlusAlarm handler (line ~461)
- `db` native handle is already in scope at that point

### Key Discrepancy (confirmed from driver source)
- `protocolSourceObjectAddress` in `realtimeData` = hex AccessSequence (e.g., `8A0E0000.1.8A0F0001`)
- `protocolSourceBrowsePath` = symbolic parent path (e.g., `AlarmDB.StructName`) — this is the correct field for DB-name prefix matching
- Source confirmed: `TagMapping.cs` line 39 (`accessSequence = varInfo.AccessSequence`), line 55 (`address = accessSequence`), line 67 (`path = ExtractPathFromName(name)`)

</code_context>

<specifics>
## Specific Ideas

No specific UI or UX references — these are pure backend endpoints.
</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.
</deferred>

---

*Phase: 13-backend-api-datablocks-tag-endpoints*
*Context gathered: 2026-03-26*

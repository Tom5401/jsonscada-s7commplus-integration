# Phase 7: Backend — Delete Endpoint + _id Exposure - Context

**Gathered:** 2026-03-23
**Status:** Ready for planning

<domain>
## Phase Boundary

Pure backend change in `server_realtime_auth/index.js`. Two tasks:
1. Remove `{ _id: 0 }` projection exclusion from `listS7PlusAlarms` so `_id`, `relationId`, `dbNumber`, and `originDbName` are all returned per document (DELETE-01, ORIGIN-05).
2. Add `POST /Invoke/auth/deleteS7PlusAlarms` endpoint — handles single-row delete (`{ ids: [...] }` body) and bulk filter delete (`{ filter: { alarmState, alarmClassName } }` body) with `authJwt.isAdmin` guard (DELETE-02, DELETE-03).

No driver changes. No frontend changes. No new connections.

</domain>

<decisions>
## Implementation Decisions

### Delete Response Format
- **D-01:** On success the delete endpoint returns **HTTP 204 No Content** (no body). Phase 8 checks `response.ok` (true for 200–299) to confirm success and removes rows from the table.
- **D-02:** On failure (DB error, bad request), return **HTTP 500** with JSON body `{ error: "<message>" }`. Auth failures (`authJwt.isAdmin` guard) return the existing 401/403 from the middleware — not 500.

### Endpoint Placement
- **D-03:** Add the delete endpoint **inline in `index.js`** immediately after the `listS7PlusAlarms` block (lines ~350–370). Reason: same as `listS7PlusAlarms` — uses the native `db` handle, not Mongoose. Not routed through `auth.routes.js`.

### Authorization
- **D-04:** `[authJwt.isAdmin]` middleware on the delete endpoint — identical to the list endpoint. No additional permission level.

### `_id` Serialization
- **D-05:** The requirement (DELETE-01) specifies `_id` serialized as a string. Researcher to confirm whether MongoDB Node.js driver ObjectId serializes to a plain hex string or `{ $oid: "..." }` via `JSON.stringify` / `res.send()`. If it serializes to an object, add `.map(doc => ({ ...doc, _id: doc._id.toHexString() }))` before sending. Planner to decide based on driver version in `package.json`.

### Claude's Discretion
- Empty-filter guard on bulk delete: whether to reject `{ filter: {} }` (would wipe the whole collection) — not discussed; planner may add a guard or leave it per the requirements.
- Invalid ObjectId handling: whether `ids` with non-parseable strings should 400 or silently skip — not discussed; planner decides appropriate defensive behavior.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Backend — List Endpoint (to modify)
- `json-scada/src/server_realtime_auth/index.js` — lines 350–370: `listS7PlusAlarms` block (inline endpoint to modify — remove `{ _id: 0 }` from projection; add delete endpoint after this block)
- `json-scada/src/server_realtime_auth/app/middlewares/authJwt.js` — `isAdmin` middleware used on all admin endpoints; same guard applies to delete endpoint

### Backend — Auth Routes (for style reference)
- `json-scada/src/server_realtime_auth/app/routes/auth.routes.js` — pattern for other `[authJwt.isAdmin]` POST endpoints (style reference only; delete does NOT go here)

### Requirements
- `.planning/REQUIREMENTS.md` — DELETE-01, DELETE-02, DELETE-03, ORIGIN-05: exact field names, body shapes, idempotency expectations

### Phase 8 Dependency
- Phase 8 (Frontend) will consume `_id` from the list response to call the delete endpoint. The `_id` field must be a plain string (not an ObjectId object) for safe JSON round-trip.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `const { MongoClient, ObjectId, Double, GridFSBucket } = require('mongodb')` (index.js line 52) — `ObjectId` already imported; use `new ObjectId(id)` to convert string IDs for `deleteMany`
- `authJwt.isAdmin` middleware — already used at line 353 for `listS7PlusAlarms`; same array pattern `[authJwt.isAdmin]` applies
- `db.collection('s7plusAlarmEvents')` — native collection handle already used at line 358; same reference for delete

### Established Patterns
- Inline async endpoint: `app.use(OPCAPI_AP + 'auth/...', [authJwt.isAdmin], async (req, res) => { try { ... } catch (err) { Log.log(err); res.status(200).send({ error: err.message }) } })` — same structure for delete, but error response is `res.status(500)` instead of `res.status(200)` per D-02
- Error logging: `Log.log(err)` before the error response
- POST body access: `req.body.ids` and `req.body.filter` (Express JSON body parser already configured at line 256/301)

### Integration Points
- Delete endpoint slots in at `index.js` line ~370, after the `listS7PlusAlarms` closing paren and before the `require('./app/routes/user.routes')` call
- For `deleteMany` by IDs: `db.collection('s7plusAlarmEvents').deleteMany({ _id: { $in: ids.map(id => new ObjectId(id)) } })`
- For `deleteMany` by filter: `db.collection('s7plusAlarmEvents').deleteMany({ alarmState: filter.alarmState, alarmClassName: filter.alarmClassName })` — omit undefined keys

</code_context>

<specifics>
## Specific Ideas

No specific requirements beyond the decisions above.

</specifics>

<deferred>
## Deferred Ideas

- Empty-filter guard (reject `{ filter: {} }` to prevent accidental full-collection wipe) — not discussed; planner may add
- Invalid ObjectId rejection vs skip — not discussed; planner decides

</deferred>

---

*Phase: 07-backend-delete-endpoint-id-exposure*
*Context gathered: 2026-03-23*

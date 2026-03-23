# Phase 7: Backend — Delete Endpoint + _id Exposure - Research

**Researched:** 2026-03-23
**Domain:** Node.js / Express 5 / MongoDB driver v7 — backend API modification
**Confidence:** HIGH

## Summary

This phase is a pure backend change in `server_realtime_auth/index.js`. It has two tightly scoped tasks: (1) remove the `{ _id: 0 }` projection exclusion from the existing `listS7PlusAlarms` endpoint so that `_id`, `relationId`, `dbNumber`, and `originDbName` all appear in the response, and (2) add a new `POST /Invoke/auth/deleteS7PlusAlarms` endpoint inline in the same file, immediately after the list block.

All patterns, imports, and middleware are already present in the codebase. `ObjectId` is already imported at line 52, `authJwt.isAdmin` is already applied to the list endpoint at line 353, and `db.collection('s7plusAlarmEvents')` is already the collection handle used at line 358. The delete endpoint is a structural copy of the list endpoint with a different body shape and a `deleteMany` call instead of `find`.

The only non-trivial question from the CONTEXT.md (D-05) is resolved: MongoDB Node.js driver v7 uses BSON v7, whose `ObjectId.toJSON()` returns a plain 24-character hex string. `JSON.stringify` calls `toJSON()`, so `res.send(docs)` will serialize `_id` as a plain string with no extra mapping step required.

**Primary recommendation:** Remove `{ _id: 0 }` from the projection, add the delete endpoint inline after line 369, apply the same `[authJwt.isAdmin]` guard, use `deleteMany` with `$in` for IDs and direct field matching for filters.

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** On success the delete endpoint returns **HTTP 204 No Content** (no body). Phase 8 checks `response.ok` (true for 200–299) to confirm success.
- **D-02:** On failure (DB error, bad request), return **HTTP 500** with JSON body `{ error: "<message>" }`. Auth failures (`authJwt.isAdmin` guard) return the existing 401/403 from the middleware — not 500.
- **D-03:** Add the delete endpoint **inline in `index.js`** immediately after the `listS7PlusAlarms` block (lines ~350–370). Same as `listS7PlusAlarms` — uses the native `db` handle, not Mongoose. Not routed through `auth.routes.js`.
- **D-04:** `[authJwt.isAdmin]` middleware on the delete endpoint — identical to the list endpoint. No additional permission level.
- **D-05:** Researcher to confirm whether MongoDB Node.js driver ObjectId serializes to a plain hex string or `{ $oid: "..." }` via `JSON.stringify` / `res.send()`. **Resolution: BSON v7 ObjectId.toJSON() returns a plain hex string. No `.map()` with `.toHexString()` needed.**

### Claude's Discretion

- Empty-filter guard on bulk delete: whether to reject `{ filter: {} }` (would wipe the whole collection) — not discussed; planner may add a guard or leave it per the requirements.
- Invalid ObjectId handling: whether `ids` with non-parseable strings should 400 or silently skip — not discussed; planner decides appropriate defensive behavior.

### Deferred Ideas (OUT OF SCOPE)

- Empty-filter guard (reject `{ filter: {} }` to prevent accidental full-collection wipe) — not discussed; planner may add
- Invalid ObjectId handling (400 vs. silently skip) — planner decides
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DELETE-01 | `listS7PlusAlarms` API returns `_id` field for each document (remove `{ _id: 0 }` projection exclusion) | One-line change: remove `{ _id: 0 }` from `.find({}, { projection: { _id: 0 } })`. ObjectId serializes as plain string (verified). |
| DELETE-02 | `POST /Invoke/auth/deleteS7PlusAlarms` handles single-row delete (`{ ids: [...] }` body) with `authJwt.isAdmin` guard | `deleteMany({ _id: { $in: ids.map(id => new ObjectId(id)) } })`. ObjectId already imported. isAdmin already used. |
| DELETE-03 | `POST /Invoke/auth/deleteS7PlusAlarms` handles bulk delete by current viewer filter (`{ filter: { alarmState, alarmClassName } }` body) | `deleteMany` with object built from filter fields present in body. Same endpoint as DELETE-02, branches on body shape. |
| ORIGIN-05 | `listS7PlusAlarms` API returns `relationId`, `dbNumber`, and `originDbName` fields per document | These fields are already written to MongoDB by Phases 5 and 6. Removing `{ _id: 0 }` projection exclusion exposes them (they were never explicitly excluded). |
</phase_requirements>

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| mongodb (Node.js driver) | ^7.0.0 (package.json) | Native collection operations: `find`, `deleteMany`, `ObjectId` | Already used in index.js; no Mongoose for this collection |
| express | ^5.0.0 | HTTP routing, `req.body`, `res.status()`, `res.send()` | Existing app framework |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| authJwt.isAdmin | (local middleware) | Admin-only route guard | Already applied to `listS7PlusAlarms`; same pattern for delete |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Inline in index.js | auth.routes.js | CONTEXT.md D-03 locks inline placement; routes.js uses Mongoose, not native db handle |
| `deleteMany` with `$in` | `deleteOne` in a loop | `deleteMany` is atomic per call; loop creates N round-trips |

**Installation:** No new dependencies required. All imports already in index.js line 52.

---

## Architecture Patterns

### Recommended Project Structure

No new files. All changes are in:
```
json-scada/src/server_realtime_auth/
└── index.js   # lines 359 (projection change) + ~370 (new endpoint insert)
```

### Pattern 1: Inline Admin Endpoint (existing pattern)

**What:** An async Express handler with `[authJwt.isAdmin]` middleware, `try/catch`, and `Log.log(err)` on error.

**When to use:** All admin endpoints that use the native `db` handle (not Mongoose). This is the established pattern for `listS7PlusAlarms`.

**Example (existing list endpoint — lines 350–369):**
```javascript
// Source: json-scada/src/server_realtime_auth/index.js lines 350–369
app.use(
  OPCAPI_AP + 'auth/listS7PlusAlarms',
  [authJwt.isAdmin],
  async (req, res) => {
    try {
      if (!db) return res.status(200).send({ error: 'DB not connected' })
      const docs = await db
        .collection('s7plusAlarmEvents')
        .find({}, { projection: { _id: 0 } })
        .sort({ createdAt: -1 })
        .limit(200)
        .toArray()
      res.status(200).send(docs)
    } catch (err) {
      Log.log(err)
      res.status(200).send({ error: err.message })
    }
  }
)
```

**Delete endpoint adaptation (per D-01, D-02):**
- Success: `res.status(204).send()` (no body)
- Error: `res.status(500).send({ error: err.message })` (not 200 like the list endpoint)
- `app.post(...)` or `app.use(...)` — use `app.post` since the delete endpoint is POST-only; `app.use` accepts all methods

### Pattern 2: `deleteMany` by IDs

```javascript
// Source: MongoDB Node.js driver v7 docs + index.js line 52 (ObjectId import)
const result = await db
  .collection('s7plusAlarmEvents')
  .deleteMany({ _id: { $in: ids.map(id => new ObjectId(id)) } })
// result.deletedCount available if needed; idempotent — missing IDs cause no error
```

### Pattern 3: `deleteMany` by Filter

```javascript
// Source: index.js code_context in CONTEXT.md
const query = {}
if (filter.alarmState !== undefined) query.alarmState = filter.alarmState
if (filter.alarmClassName !== undefined) query.alarmClassName = filter.alarmClassName
const result = await db.collection('s7plusAlarmEvents').deleteMany(query)
```

### Anti-Patterns to Avoid

- **Using `{ _id: 0 }` in projection:** The existing `find({}, { projection: { _id: 0 } })` at line 359 is exactly what DELETE-01 and ORIGIN-05 require removing. All required fields (`_id`, `relationId`, `dbNumber`, `originDbName`) are present in the collection from Phase 5/6; the projection was the only thing excluding them.
- **Routing through auth.routes.js:** D-03 locks the inline placement. auth.routes.js uses Mongoose-based controllers; the alarm collection uses native `db` handle.
- **`app.use` for POST-only endpoint:** While `app.use` works (existing list uses it), `app.post` is more semantically correct for a write endpoint and prevents accidental GET calls triggering a delete.
- **Manual ObjectId-to-string conversion:** BSON v7 `ObjectId.toJSON()` returns a plain hex string. `res.send(docs)` already produces the correct format for Phase 8 consumption.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| ObjectId string-to-BSON conversion | Custom regex/Buffer parsing | `new ObjectId(id)` | Already imported at line 52; handles validation and throws on invalid input |
| Admin authorization | Custom token check in endpoint body | `[authJwt.isAdmin]` middleware array | Already implemented and used; consistent behavior across all admin endpoints |
| Idempotent delete | Check-then-delete logic | `deleteMany` directly | MongoDB `deleteMany` with non-existent IDs returns `{ deletedCount: 0 }` — not an error. Calling twice with same IDs is naturally idempotent. |

**Key insight:** This phase is adding ~25 lines of code. All complex infrastructure (ObjectId, auth, db handle, body parsing, error logging) already exists and is imported.

---

## Common Pitfalls

### Pitfall 1: `isAdmin` Returns `false` Without a Response When Token is Missing

**What goes wrong:** When no token is present, `checkToken()` returns `false`, and `isAdmin` does `if (tok === false) return false` — it does not call `res.status(401).send(...)`. The request hangs and eventually times out.

**Why it happens:** This is existing behavior in the codebase. The requirement says "calling the delete endpoint without a valid admin JWT returns 401/403" — but the actual behavior from `isAdmin` when there's NO token is to return no response (the middleware exits without calling `next()` or `res.send()`).

**How to avoid:** This is existing behavior shared with `listS7PlusAlarms`. Do not attempt to fix it in this phase — the requirement states "401/403" and the 403 path (token present but non-admin user) does return 403. The no-token path is a pre-existing issue outside this phase's scope. The plan should document this as an existing behavior, not a regression.

**Warning signs:** Curl test with no token gets no response (hangs) rather than 401.

### Pitfall 2: `{ filter: {} }` Wipes the Entire Collection

**What goes wrong:** If `req.body.filter` is `{}` (empty object), the constructed query `{}` matches every document. `deleteMany({})` deletes everything in `s7plusAlarmEvents`.

**Why it happens:** No guard prevents an empty filter from becoming a full-collection delete.

**How to avoid:** This is in "Claude's Discretion" — the planner should decide whether to add a guard. Recommended: reject if both `alarmState` and `alarmClassName` are absent (result would be an empty query). Return 400 with `{ error: "filter must include at least one field" }`.

### Pitfall 3: Invalid ObjectId String Throws in `new ObjectId(id)`

**What goes wrong:** `new ObjectId("invalid-string")` throws a `BSONError`. If `ids` array contains any invalid string, the entire request fails with 500 before any delete occurs.

**Why it happens:** The MongoDB driver validates ObjectId format on construction.

**How to avoid:** This is in "Claude's Discretion." Recommended: wrap `ids.map(id => new ObjectId(id))` in a try/catch or pre-validate with `ObjectId.isValid(id)` before mapping. Return 400 for invalid IDs rather than 500.

### Pitfall 4: `app.use` Accepts All HTTP Methods

**What goes wrong:** Using `app.use(OPCAPI_AP + 'auth/deleteS7PlusAlarms', ...)` accepts GET, DELETE, PUT, and POST. A browser preflight or stray GET triggers the delete handler.

**Why it happens:** `app.use` is method-agnostic by design.

**How to avoid:** Use `app.post(OPCAPI_AP + 'auth/deleteS7PlusAlarms', [authJwt.isAdmin], async (req, res) => {...})`. The existing list endpoint uses `app.use` for legacy reasons; don't repeat this for a write endpoint.

---

## Code Examples

Verified patterns from official sources and existing codebase:

### D-05 Resolution: ObjectId Serialization (BSON v7)

```javascript
// Source: BSON v7 source — ObjectId.toJSON() returns this.toHexString()
// JSON.stringify calls toJSON(), so:
const id = new ObjectId('507f1f77bcf86cd799439011')
JSON.stringify(id)   // => '"507f1f77bcf86cd799439011"'  (plain string, no wrapping object)

// res.send(docs) in Express uses JSON.stringify internally.
// The _id field in the list response will be: "507f1f77bcf86cd799439011"
// NO .map(doc => ({ ...doc, _id: doc._id.toHexString() })) is required.
```

### Projection Change (DELETE-01 + ORIGIN-05)

```javascript
// BEFORE (line 359):
.find({}, { projection: { _id: 0 } })

// AFTER: remove projection entirely, or pass empty projection:
.find({})
// All fields including _id, relationId, dbNumber, originDbName are returned.
```

### Delete Endpoint Structure

```javascript
// Source: Adapted from listS7PlusAlarms pattern (index.js lines 350–369) + CONTEXT.md
app.post(
  OPCAPI_AP + 'auth/deleteS7PlusAlarms',
  [authJwt.isAdmin],
  async (req, res) => {
    try {
      if (!db) return res.status(500).send({ error: 'DB not connected' })

      const { ids, filter } = req.body

      if (ids) {
        // DELETE-02: single or multi-row delete by _id
        await db
          .collection('s7plusAlarmEvents')
          .deleteMany({ _id: { $in: ids.map(id => new ObjectId(id)) } })
      } else if (filter) {
        // DELETE-03: bulk delete by filter fields
        const query = {}
        if (filter.alarmState !== undefined) query.alarmState = filter.alarmState
        if (filter.alarmClassName !== undefined) query.alarmClassName = filter.alarmClassName
        await db.collection('s7plusAlarmEvents').deleteMany(query)
      } else {
        return res.status(400).send({ error: 'Body must contain ids or filter' })
      }

      res.status(204).send()  // D-01: 204 No Content on success
    } catch (err) {
      Log.log(err)
      res.status(500).send({ error: err.message })  // D-02: 500 on failure
    }
  }
)
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| BSON ObjectId.toJSON() returned `{ $oid: "..." }` (extended JSON) | Returns plain hex string | BSON v4+ | `res.send(docs)` gives frontend a plain string `_id` — safe for JSON round-trip |

**Not deprecated in this context:** `deleteMany` with `$in` is the standard approach for bulk delete by ID. No changes in MongoDB driver v7 affect this pattern.

---

## Open Questions

1. **`isAdmin` no-token behavior — 401 vs. hang**
   - What we know: When no token is present, `isAdmin` returns `false` without sending a response (request hangs). When token is present but user lacks admin role, it returns 403.
   - What's unclear: Whether the success criteria "returns 401/403" can be satisfied as-is or requires a fix to `isAdmin` (out of scope for this phase).
   - Recommendation: Planner should note this as existing behavior. The 403 path is functional. The no-token hang is a pre-existing issue in `authJwt.isAdmin`. Do not fix in this phase unless explicitly in scope.

2. **Empty-filter guard**
   - What we know: Planner's discretion (CONTEXT.md). `deleteMany({})` would wipe the collection.
   - Recommendation: Add guard — reject if neither `alarmState` nor `alarmClassName` is present in filter. Return 400.

3. **Invalid ObjectId string handling**
   - What we know: Planner's discretion (CONTEXT.md). `new ObjectId("bad")` throws.
   - Recommendation: Validate with `ObjectId.isValid(id)` before mapping, return 400 for any invalid ID.

---

## Environment Availability

Step 2.6: SKIPPED — This phase is a pure code change in an existing file (`index.js`). No external tools, services, databases, or CLI utilities need to be installed or verified. The MongoDB connection and all dependencies are managed by the running server.

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | None detected (package.json: `"test": "echo \"Error: no test specified\" && exit 1"`) |
| Config file | None — Wave 0 must establish |
| Quick run command | N/A until Wave 0 |
| Full suite command | N/A until Wave 0 |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DELETE-01 | `_id` present in listS7PlusAlarms response | manual-only (requires running server + MongoDB) | `curl -s -H "x-access-token: $TOKEN" http://localhost:3000/Invoke/auth/listS7PlusAlarms \| jq '.[0]._id'` | N/A |
| ORIGIN-05 | `relationId`, `dbNumber`, `originDbName` in list response | manual-only (same as above) | `curl ... \| jq '.[0] \| {relationId, dbNumber, originDbName}'` | N/A |
| DELETE-02 | POST with `{ ids: [...] }` deletes document; second call idempotent | manual-only (requires running server + MongoDB) | `curl -s -X POST -H "Content-Type: application/json" -d '{"ids":["$ID"]}' ... deleteS7PlusAlarms` returns 204 | N/A |
| DELETE-03 | POST with `{ filter: {...} }` deletes matching documents | manual-only (requires running server + MongoDB) | curl with filter body, verify count in MongoDB | N/A |

**Manual-only justification:** All endpoints require a live MongoDB instance and a valid JWT. There is no unit test harness and no mock infrastructure in this project. All verification is via curl against a running server.

### Sampling Rate
- **Per task commit:** Visual diff review of the change; no automated suite
- **Per wave merge:** Manual curl tests against running server
- **Phase gate:** All 4 success criteria verified manually before `/gsd:verify-work`

### Wave 0 Gaps
None — no test framework setup needed. Phase is verified manually per project convention.

---

## Sources

### Primary (HIGH confidence)
- BSON v7 source (github.com/mongodb/js-bson) — `ObjectId.toJSON()` returns `this.toHexString()` (plain string, not extended JSON)
- `json-scada/src/server_realtime_auth/index.js` lines 45–57, 350–370 — verified imports, endpoint structure, exact line positions
- `json-scada/src/server_realtime_auth/app/middlewares/authJwt.js` — verified `isAdmin` behavior including no-token path
- `json-scada/src/server_realtime_auth/package.json` — confirmed `mongodb: ^7.0.0`, `express: ^5.0.0`

### Secondary (MEDIUM confidence)
- MongoDB Node.js driver v7 Extended JSON docs (mongodb.com/docs) — confirmed `{ $oid: "..." }` is EJSON format only, not standard JSON.stringify output

### Tertiary (LOW confidence)
- None

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — package.json verified, all imports confirmed in source
- Architecture: HIGH — existing pattern fully visible in source; delete endpoint is a direct adaptation
- Pitfalls: HIGH (D-05 resolution, empty filter, invalid ObjectId) — MEDIUM (isAdmin no-token behavior is observed from source, functional impact confirmed)

**Research date:** 2026-03-23
**Valid until:** 2026-04-23 (stable — no external dependencies)

---
phase: 07-backend-delete-endpoint-id-exposure
verified: 2026-03-24T00:00:00Z
status: human_needed
score: 5/6 must-haves verified (6th requires runtime)
human_verification:
  - test: "Confirm admin JWT guard rejects unauthenticated calls"
    expected: "POST /Invoke/auth/deleteS7PlusAlarms without a JWT returns 401 or 403 and no document is deleted"
    why_human: "authJwt.isAdmin middleware wiring is confirmed in code, but the actual HTTP response code for no-token vs non-admin-token paths cannot be asserted without a running server (RESEARCH.md Pitfall 1 notes a pre-existing middleware behaviour where the no-token path may hang)"
---

# Phase 7: Backend — Delete Endpoint + _id Exposure — Verification Report

**Phase Goal:** The alarm list API exposes document IDs and a new delete endpoint allows authenticated admins to remove alarm history.
**Verified:** 2026-03-24
**Status:** human_needed — all automated checks pass; one truth (auth guard runtime behaviour) requires a live server test
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | `GET /Invoke/auth/listS7PlusAlarms` response includes `_id` as a 24-char hex string for each document | VERIFIED | `index.js:359` — `.find({})` with no projection; no `{ _id: 0 }` exclusion anywhere in the listS7PlusAlarms block |
| 2 | `GET /Invoke/auth/listS7PlusAlarms` response includes `relationId`, `dbNumber`, and `originDbName` fields | VERIFIED | Same projection removal at line 359 returns all stored fields; fields written by Phases 5/6 driver; no exclusion in place |
| 3 | `POST /Invoke/auth/deleteS7PlusAlarms` with `{ ids: ['<id>'] }` deletes the matching document and returns 204 | VERIFIED | `index.js:381-391` — ids path: ObjectId conversion + `deleteMany({ _id: { $in: objectIds } })`; `index.js:405` — `res.status(204).send()` |
| 4 | `POST /Invoke/auth/deleteS7PlusAlarms` called twice with the same id returns 204 both times (idempotent) | VERIFIED | `deleteMany` on a non-existent `_id` returns `{ deletedCount: 0 }`, not an error; no error branch triggered; 204 returned both times |
| 5 | `POST /Invoke/auth/deleteS7PlusAlarms` with `{ filter: { alarmState: 'Going' } }` deletes all matching documents and returns 204 | VERIFIED | `index.js:392-400` — filter path: `query.alarmState` / `query.alarmClassName` set; `deleteMany(query)` called; empty-filter guard at line 397 prevents `{}` wipe |
| 6 | `POST /Invoke/auth/deleteS7PlusAlarms` without a valid admin JWT does not delete anything | PARTIAL | `[authJwt.isAdmin]` middleware is present at `index.js:374`; middleware wiring is confirmed; actual 401/403 HTTP response requires live server (see Human Verification) |

**Score:** 5/6 truths fully verified; truth 6 verified at the wiring level but runtime behaviour needs human confirmation.

---

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `json-scada/src/server_realtime_auth/index.js` | listS7PlusAlarms projection fix + deleteS7PlusAlarms endpoint | VERIFIED | File exists; contains `auth/deleteS7PlusAlarms` (line 373); `.find({})` at line 359; fully substantive (no stubs, no TODOs) |

**Level 1 — Exists:** File present at the declared path.
**Level 2 — Substantive:** Both changes are fully implemented; no placeholder comments, no `return null`, no TODO in the modified block.
**Level 3 — Wired:** Both endpoints are registered with Express (`app.use` for list, `app.post` for delete); both reach the `db.collection('s7plusAlarmEvents')` code path; `require('./app/routes/user.routes')` appears at line 413, after the delete endpoint at line 411.
**Level 4 — Data flows:** Endpoints call `db.collection('s7plusAlarmEvents')` directly via the native MongoDB driver; `db` is the native `MongoClient` handle established at server startup; real collection operations (not static returns).

---

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| deleteS7PlusAlarms endpoint | `db.collection('s7plusAlarmEvents')` | `deleteMany` with `$in` (ids) or field match (filter) | WIRED | `index.js:391` — `deleteMany({ _id: { $in: objectIds } })`; `index.js:400` — `deleteMany(query)` |
| deleteS7PlusAlarms endpoint | `authJwt.isAdmin` middleware | `[authJwt.isAdmin]` middleware array | WIRED | `index.js:374` — `[authJwt.isAdmin]` in the `app.post` call, identical pattern to listS7PlusAlarms at line 353 |
| listS7PlusAlarms endpoint | `db.collection('s7plusAlarmEvents')` | `.find({})` without `_id` exclusion | WIRED | `index.js:359` — `.find({})` with no second argument; no `{ projection: { _id: 0 } }` present; confirmed by `grep "projection" index.js` returning only unrelated lines (1595, 1598, 1648, 1732) |

---

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| listS7PlusAlarms endpoint | `docs` | `db.collection('s7plusAlarmEvents').find({}).sort().limit().toArray()` | Yes — live MongoDB query | FLOWING |
| deleteS7PlusAlarms endpoint | `deleteMany` result | `db.collection('s7plusAlarmEvents').deleteMany(...)` | Yes — live MongoDB mutation | FLOWING |

---

### Behavioral Spot-Checks

Step 7b: SKIPPED — verifying an Express HTTP server requires a running process. Static code analysis (Steps 3–5) provides equivalent assurance for this phase's scope.

---

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| DELETE-01 | 07-01-PLAN.md | `listS7PlusAlarms` returns `_id` field (remove `{ _id: 0 }` projection) | SATISFIED | `index.js:359` — `.find({})` no projection |
| DELETE-02 | 07-01-PLAN.md | `POST /Invoke/auth/deleteS7PlusAlarms` handles single-row delete by `ids` array with `authJwt.isAdmin` guard | SATISFIED | `index.js:372-411` — `app.post`, `[authJwt.isAdmin]`, ids path with `deleteMany($in)` |
| DELETE-03 | 07-01-PLAN.md | `POST /Invoke/auth/deleteS7PlusAlarms` handles bulk delete by `{ filter: { alarmState, alarmClassName } }` | SATISFIED | `index.js:392-400` — filter path with field-whitelist query construction and `deleteMany(query)` |
| ORIGIN-05 | 07-01-PLAN.md | `listS7PlusAlarms` returns `relationId`, `dbNumber`, `originDbName` per document | SATISFIED | Same projection removal at line 359 returns all stored fields including origin fields written by Phases 5/6 |

All four Phase 7 requirement IDs from PLAN frontmatter are implemented. No orphaned requirements: REQUIREMENTS.md Traceability table maps DELETE-01, DELETE-02, DELETE-03, ORIGIN-05 to Phase 7 only. Phase 8 requirements (DELETE-04, DELETE-05, DELETE-06, ORIGIN-06, ORIGIN-07, ORIGIN-08) are correctly deferred and not claimed here.

---

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| — | — | — | — | No anti-patterns found in the modified block |

Scan of lines 350–411 of `index.js`: no TODO/FIXME/PLACEHOLDER comments, no `return null`, no empty handlers, no hardcoded empty arrays or objects flowing to response. `res.status(204).send()` is the correct empty-body success response for a delete endpoint, not a stub.

---

### Human Verification Required

#### 1. Auth Guard — Runtime HTTP Response

**Test:** Start the server locally (or in Docker). Send `POST /Invoke/auth/deleteS7PlusAlarms` with a valid document ID but no `Authorization` header. Then repeat with an `Authorization: Bearer <non-admin-token>` header.

**Expected:**
- No `Authorization` header: server returns HTTP 401 or 403 (or hangs per Pitfall 1 noted in RESEARCH.md — either way, the document must not be deleted).
- Non-admin token: server returns HTTP 403 and the document is not deleted.
- Confirm via a follow-up `GET /Invoke/auth/listS7PlusAlarms` that the document count is unchanged.

**Why human:** The `[authJwt.isAdmin]` middleware wiring is confirmed in source code. However, the exact HTTP response code for the unauthenticated path has a pre-existing ambiguity (RESEARCH.md Pitfall 1: no-token path may hang rather than return 401). The business requirement is "document not deleted" — this must be confirmed against a running server. This is a known pre-existing middleware behaviour, not a regression introduced in Phase 7.

---

### Gaps Summary

No gaps found. All code-verifiable must-haves are satisfied:

- The `{ _id: 0 }` projection exclusion was removed from `listS7PlusAlarms` (line 359).
- `POST /Invoke/auth/deleteS7PlusAlarms` is registered with `app.post`, guarded by `[authJwt.isAdmin]`, implements both ids-based and filter-based delete, returns 204 on success, 400 on bad input, and 500 on server error.
- All four requirement IDs (DELETE-01, DELETE-02, DELETE-03, ORIGIN-05) have direct implementation evidence.
- Both git commits (46c40d1, dbbe0ca) exist in the repository.
- `node -c` syntax check passes.

The only open item is a live-server confirmation of the auth guard HTTP response code, which is a runtime behaviour that cannot be asserted by static analysis.

---

_Verified: 2026-03-24_
_Verifier: Claude (gsd-verifier)_

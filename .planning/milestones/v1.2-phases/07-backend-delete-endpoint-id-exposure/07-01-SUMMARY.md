---
phase: 07-backend-delete-endpoint-id-exposure
plan: 01
subsystem: backend-api
tags: [mongodb, express, api, delete-endpoint, id-exposure]
dependency_graph:
  requires: [phase-05-driver-relationid-fields, phase-06-driver-startup-db-name-map]
  provides: [deleteS7PlusAlarms-endpoint, _id-in-list-response, origin-fields-in-list-response]
  affects: [phase-08-frontend-delete-buttons-origin-columns]
tech_stack:
  added: []
  patterns: [express-inline-endpoint, mongodb-deleteMany, authJwt-isAdmin-guard, objectid-conversion]
key_files:
  created: []
  modified:
    - json-scada/src/server_realtime_auth/index.js
decisions:
  - "HTTP 204 No Content on delete success — matches D-01 from context"
  - "Empty filter guard rejects { filter: {} } with 400 — prevents accidental full-collection wipe"
  - "Invalid ObjectId try/catch returns 400 — defensive input validation"
  - "app.post (not app.use) for delete endpoint — POST-only prevents accidental GET triggering delete"
  - "deleteMany is naturally idempotent — deleting already-deleted IDs returns deletedCount:0, not error"
metrics:
  duration: "~15 min"
  completed: "2026-03-24"
  tasks: 2
  files: 1
requirements_satisfied: [DELETE-01, DELETE-02, DELETE-03, ORIGIN-05]
---

# Phase 7 Plan 01: Backend Delete Endpoint + _id Exposure Summary

**One-liner:** Removed `{ _id: 0 }` projection exclusion from `listS7PlusAlarms` and added `POST /Invoke/auth/deleteS7PlusAlarms` with admin guard, ids-based and filter-based delete, returning 204 on success.

## What Was Built

Two changes to `json-scada/src/server_realtime_auth/index.js`:

1. **listS7PlusAlarms projection fix** — Removed `{ projection: { _id: 0 } }` from the `.find()` call. All document fields are now returned: `_id` (as 24-char hex string via BSON v7 ObjectId.toJSON()), `relationId`, `dbNumber`, `originDbName`, and all existing alarm fields.

2. **deleteS7PlusAlarms endpoint** — New `app.post(OPCAPI_AP + 'auth/deleteS7PlusAlarms', [authJwt.isAdmin], ...)` endpoint with two delete modes:
   - **IDs mode:** `{ ids: ['<hex-id>', ...] }` body — converts strings to ObjectIds, runs `deleteMany({ _id: { $in: objectIds } })`
   - **Filter mode:** `{ filter: { alarmState, alarmClassName } }` body — builds safe query from named fields, runs `deleteMany(query)`
   - Success: HTTP 204 No Content
   - Bad input (missing body, invalid ObjectId, empty filter): HTTP 400 with `{ error }` body
   - Server error: HTTP 500 with `{ error }` body (after `Log.log(err)`)

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Remove _id projection exclusion from listS7PlusAlarms | 46c40d1 | json-scada/src/server_realtime_auth/index.js |
| 2 | Add deleteS7PlusAlarms endpoint | dbbe0ca | json-scada/src/server_realtime_auth/index.js |

## Verification Results

- `node -c index.js` — PASS (syntax valid)
- `grep listS7PlusAlarms\|deleteS7PlusAlarms` — both endpoints present at lines 352 and 411
- `grep "projection.*_id.*0"` — no match (exclusion removed)
- `grep "app\.post.*deleteS7PlusAlarms"` — present (not app.use)
- `grep "\[authJwt\.isAdmin\]"` on deleteS7PlusAlarms block — present
- `grep "res\.status(204)"` — present
- `grep "deleteMany.*_id.*\$in"` — present
- `grep "Object\.keys(query)\.length === 0"` — present (empty filter guard)
- `require('./app/routes/user.routes')` appears after deleteS7PlusAlarms — confirmed

## Success Criteria Met

1. `listS7PlusAlarms` returns all fields (no `{ _id: 0 }` projection) — DELETE-01, ORIGIN-05
2. `deleteS7PlusAlarms` exists with `app.post` and `[authJwt.isAdmin]` guard — DELETE-02, DELETE-03
3. Delete by IDs: `{ ids: [...] }` body triggers `deleteMany` with `$in` — DELETE-02
4. Delete by filter: `{ filter: { ... } }` body triggers `deleteMany` with field match — DELETE-03
5. Empty filter rejected with 400 — safety guard
6. Invalid ObjectId rejected with 400 — input validation
7. Success returns 204 No Content — D-01
8. Error returns 500 with `{ error }` body — D-02
9. File passes `node -c` syntax check

## Deviations from Plan

None — plan executed exactly as written.

The worktree had a submodule initialization issue (cloned from remote instead of using local objects) which required checking out the worktree's json-scada module separately before making changes. This is an infrastructure concern, not a plan deviation.

## Known Stubs

None — both endpoints are fully wired. The `_id` field is returned from MongoDB as-is (no transformation needed; BSON v7 ObjectId.toJSON() returns a plain hex string).

## Self-Check: PASSED

- `.planning/phases/07-backend-delete-endpoint-id-exposure/07-01-SUMMARY.md` — FOUND (this file)
- Worktree commit `46c40d1` — FOUND (task 1: projection fix)
- Worktree commit `dbbe0ca` — FOUND (task 2: delete endpoint)
- json-scada submodule commit `074ae673` — FOUND (combined changes)
- `node -c` syntax check — PASSED

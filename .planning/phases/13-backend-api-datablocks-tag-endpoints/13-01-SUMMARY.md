---
phase: 13-backend-api-datablocks-tag-endpoints
plan: 01
subsystem: backend-api
tags: [api, mongodb, s7plus, nodejs, express]
dependency_graph:
  requires: [Phase 12 — s7plusDatablocks collection populated by driver]
  provides: [listS7PlusDatablocks endpoint, listS7PlusTagsForDb endpoint, touchS7PlusActiveTagRequests endpoint]
  affects: [Phase 14 DatablockBrowser frontend, Phase 15 TagTreeBrowser frontend]
tech_stack:
  added: []
  patterns: [app.use for GET endpoints, app.post for POST endpoint, authJwt.isAdmin guard, parseInt/isNaN validation, regex escape for db name prefix match, bulkWrite upsert with ordered:false]
key_files:
  created: []
  modified:
    - json-scada/src/server_realtime_auth/index.js
decisions:
  - "protocolSourceBrowsePath (not protocolSourceObjectAddress) used in listS7PlusTagsForDb per D-01 — browse path holds symbolic DB name prefix, object address is hex access sequence"
  - "touchS7PlusActiveTagRequests does direct upsert without realtimeData lookup per D-02 — PoC simplicity; OPC read service only needs address to poll"
  - "source: 'tag-tree' set on all activeTagRequests upserts to distinguish from existing 'opc-read' source"
metrics:
  duration: "~5 minutes"
  completed: "2026-03-26"
  tasks_completed: 2
  files_modified: 1
---

# Phase 13 Plan 01: Backend API — Datablocks & Tag Endpoints Summary

Three new admin-guarded HTTP endpoints added to server_realtime_auth/index.js: listS7PlusDatablocks (queries s7plusDatablocks collection), listS7PlusTagsForDb (queries realtimeData by protocolSourceBrowsePath regex per D-01), and touchS7PlusActiveTagRequests (bulkWrite upsert into activeTagRequests with source: 'tag-tree' per D-02).

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add GET listS7PlusDatablocks and listS7PlusTagsForDb endpoints | bb933b52 | json-scada/src/server_realtime_auth/index.js |
| 2 | Add POST touchS7PlusActiveTagRequests endpoint | 86e2a3d9 | json-scada/src/server_realtime_auth/index.js |

## What Was Built

### listS7PlusDatablocks (GET)
- Route: `GET /Invoke/auth/listS7PlusDatablocks?connectionNumber=N`
- Guard: `[authJwt.isAdmin]`
- Queries `s7plusDatablocks` collection, filtered by `connectionNumber`, sorted by `db_name: 1`
- Returns 400 if `connectionNumber` missing/invalid; 200+error if DB not connected; 500 on unexpected error

### listS7PlusTagsForDb (GET)
- Route: `GET /Invoke/auth/listS7PlusTagsForDb?connectionNumber=N&dbName=X`
- Guard: `[authJwt.isAdmin]`
- Queries `realtimeData` (via `COLL_REALTIME`) with `protocolSourceBrowsePath` regex `^dbName(\.|$)` — avoids false prefix matches (e.g., `DB1` not matching `DB10`)
- `dbName` is regex-escaped to prevent injection
- Returns 400 if either param missing/invalid

### touchS7PlusActiveTagRequests (POST)
- Route: `POST /Invoke/auth/touchS7PlusActiveTagRequests`
- Guard: `[authJwt.isAdmin]`
- Body: array of `{connectionNumber, protocolSourceObjectAddress}` pairs
- Upserts each valid pair into `activeTagRequests` with refreshed `expiresAt` (using `ACTIVE_TAG_TTL_SECONDS`), `updatedAt`, and `source: 'tag-tree'`
- Skips invalid items silently; returns 400 if all items invalid
- Single `bulkWrite(..., { ordered: false })` round-trip
- No `pointKey` or `tag` fields set (per D-02)

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all three endpoints are fully wired to their respective MongoDB collections. No placeholder data.

## Self-Check: PASSED

Files modified:
- FOUND: /c/Users/tnielen/Documents/Levvel_PoC/dev/json-scada/src/server_realtime_auth/index.js

Commits verified:
- FOUND: bb933b52 (feat(13-01): add GET listS7PlusDatablocks and listS7PlusTagsForDb endpoints)
- FOUND: 86e2a3d9 (feat(13-01): add POST touchS7PlusActiveTagRequests endpoint)

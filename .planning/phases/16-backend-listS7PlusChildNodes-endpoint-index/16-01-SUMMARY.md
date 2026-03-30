---
phase: 16-backend-listS7PlusChildNodes-endpoint-index
plan: "01"
subsystem: api
tags: [mongodb, express, s7plus, index, endpoint]

requires: []
provides:
  - "GET /Invoke/auth/listS7PlusChildNodes endpoint — returns direct-child realtimeData documents by equality filter on protocolSourceBrowsePath"
  - "ensureS7PlusChildNodesIndex() — compound index {protocolSourceConnectionNumber, protocolSourceBrowsePath} on realtimeData, created idempotently at server startup"
  - "hasChildren boolean per document — derived from scoped distinct query on childFullPaths"
affects: [17, 18, 19, frontend-tag-tree, s7plus]

tech-stack:
  added: []
  patterns:
    - "ensure-index pattern: dedicated async ensureXxx() function called at MongoDB connect, createIndex is idempotent so safe on restart"
    - "scoped distinct for hasChildren: build childFullPaths array, run distinct in one extra round-trip, convert to Set for O(1) lookup"
    - "equality filter for direct children: protocolSourceBrowsePath: path (no regex) as the critical distinction from listS7PlusTagsForDb"

key-files:
  created: []
  modified:
    - json-scada/src/server_realtime_auth/index.js

key-decisions:
  - "D-01: scoped distinct for hasChildren — one extra round-trip but index-tight, avoids pulling all unique paths for full connection"
  - "D-02: projected subset (9 fields) — not full realtimeData documents"
  - "D-03: [authJwt.isAdmin] guard — consistent with all other S7Plus endpoints"
  - "D-04: dedicated ensureS7PlusChildNodesIndex() function, called at startup after ensureS7PlusAlarmIndexes()"
  - "D-05: equality match on protocolSourceBrowsePath (not regex) — direct children only, not subtree"

patterns-established:
  - "ensureS7PlusChildNodesIndex pattern: mirrors ensureActiveTagRequestIndexes()/ensureS7PlusAlarmIndexes() — use for future index additions"
  - "listS7PlusChildNodes endpoint: template for lazy-loaded tree navigation (equality filter + projection + scoped distinct)"

requirements-completed:
  - PERF-01
  - PERF-02

duration: 20min
completed: 2026-03-30
---

# Phase 16: Backend — listS7PlusChildNodes Endpoint + Index Summary

**Added compound MongoDB index on realtimeData and a new GET endpoint that returns exactly one level of the tag tree per expand with `hasChildren` boolean — unblocking all v1.5 phases (17–19).**

## Performance

- **Duration:** ~20 min
- **Started:** 2026-03-30T09:00:00Z
- **Completed:** 2026-03-30T11:06:06Z
- **Tasks:** 2
- **Files modified:** 1 (json-scada submodule)

## Accomplishments
- Added `ensureS7PlusChildNodesIndex()` — creates compound index `{protocolSourceConnectionNumber: 1, protocolSourceBrowsePath: 1}` (`idx_connNum_browsePath`) on `realtimeData` at server startup (PERF-02)
- Added `GET /Invoke/auth/listS7PlusChildNodes` endpoint guarded by `[authJwt.isAdmin]`, using equality filter for direct children (D-05), 9-field projection (D-02), and scoped `distinct` for `hasChildren` boolean (D-01) (PERF-01)
- Startup call added after `await ensureS7PlusAlarmIndexes()` per D-04 — index is idempotent, safe across restarts

## Task Commits

1. **Task 1+2: ensureS7PlusChildNodesIndex + listS7PlusChildNodes endpoint** — `bcf5fa11` (feat) *(in json-scada submodule)*
2. **Parent repo submodule update** — `c8db59d` (feat)

## Files Created/Modified
- `json-scada/src/server_realtime_auth/index.js` — Added `ensureS7PlusChildNodesIndex()` function (~L186), `listS7PlusChildNodes` app.use block (~L530), and startup `await` call (~L2865)

## Decisions Made
All decisions followed locked CONTEXT.md decisions D-01 through D-05. No new choices required.

## Deviations from Plan

### Auto-fixed Issues

**1. UTF-8 BOM introduced during file write**
- **Found during:** Post-write diff review
- **Issue:** PowerShell `[System.Text.Encoding]::UTF8` adds BOM; original file had none
- **Fix:** Stripped BOM bytes before amending commit using `[System.IO.File]::ReadAllBytes` / `WriteAllBytes`
- **Files modified:** `json-scada/src/server_realtime_auth/index.js`
- **Verification:** `git show HEAD -- src/server_realtime_auth/index.js` confirms clean first line
- **Committed in:** `bcf5fa11` (amended)

---

**Total deviations:** 1 auto-fixed (encoding)
**Impact on plan:** Cosmetic encoding fix only. No scope creep.

## Issues Encountered
- `json-scada` is a git submodule — gsd-tools commit requires staging inside the submodule first, then updating the parent repo reference separately. Pattern: commit inside submodule → return to parent → `gsd-tools commit --files json-scada`

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- **Phase 17 (frontend TagTreeBrowser):** Can now call `GET /Invoke/auth/listS7PlusChildNodes?connectionNumber=N&path=P` to fetch one level of the tree per expand. Response is an array of `{ _id, ungroupedDescription, protocolSourceBrowsePath, protocolSourceConnectionNumber, protocolSourceObjectAddress, type, value, valueString, commandOfSupervised, hasChildren }`.
- **Index performance:** Compound index on `{protocolSourceConnectionNumber, protocolSourceBrowsePath}` covers both the direct-children `find()` query and the `distinct()` query for `hasChildren`.

---
*Phase: 16-backend-listS7PlusChildNodes-endpoint-index*
*Completed: 2026-03-30*

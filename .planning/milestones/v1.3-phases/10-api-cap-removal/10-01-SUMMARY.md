---
phase: 10-api-cap-removal
plan: 01
subsystem: api
tags: [mongodb, nodejs, alarm-viewer, index]

# Dependency graph
requires:
  - phase: 07-backend-delete-endpoint-id-exposure
    provides: listS7PlusAlarms endpoint and s7plusAlarmEvents collection
provides:
  - listS7PlusAlarms endpoint returns all alarm documents (no ceiling)
  - MongoDB { createdAt: -1 } index on s7plusAlarmEvents created at server startup
affects: [alarm-viewer, s7plusAlarmEvents, server_realtime_auth]

# Tech tracking
tech-stack:
  added: []
  patterns: [ensureXxxIndexes() pattern for collection index management at startup]

key-files:
  created: []
  modified:
    - json-scada/src/server_realtime_auth/index.js

key-decisions:
  - "No replacement limit — full collection returned, index protects query performance"
  - "Descending createdAt index (not ascending) matches sort direction for optimal scan"
  - "Index name idx_createdAt_desc for clarity in getIndexes() output"

patterns-established:
  - "ensureXxxIndexes() pattern: guard + createIndex in try/catch, called from MongoDB connect callback"

requirements-completed: [API-01, API-02]

# Metrics
duration: 2min
completed: 2026-03-25
---

# Phase 10 Plan 01: API Cap Removal Summary

**Removed 200-alarm hard ceiling from listS7PlusAlarms and added MongoDB { createdAt: -1 } index on s7plusAlarmEvents for query protection.**

## What Was Built

The `listS7PlusAlarms` endpoint in `json-scada/src/server_realtime_auth/index.js` previously applied `.limit(200)` to the query, silently capping the alarm viewer at 200 rows regardless of collection size. This plan removes that limit and adds a `{ createdAt: -1 }` index to ensure unbounded queries remain performant.

Two changes, one file:

1. **Removed `.limit(200)`** from the `find({}).sort({ createdAt: -1 })` chain in the `listS7PlusAlarms` handler. The endpoint now returns all documents sorted by `createdAt` descending.

2. **Added `ensureS7PlusAlarmIndexes()`** — a new async function following the existing `ensureActiveTagRequestIndexes()` pattern. Creates `{ createdAt: -1 }` with name `idx_createdAt_desc` on `s7plusAlarmEvents`. Called with `await` inside the MongoDB connect callback immediately after `ensureActiveTagRequestIndexes()`.

## Tasks Completed

| # | Task | Commit | Files |
|---|------|--------|-------|
| 1 | Add ensureS7PlusAlarmIndexes and wire to startup | f07120d2 (submodule) | json-scada/src/server_realtime_auth/index.js |
| 2 | Remove .limit(200) from listS7PlusAlarms query | f07120d2 (submodule) | json-scada/src/server_realtime_auth/index.js |

Both tasks shipped atomically in commit `cc27b84` (parent repo) / `f07120d2` (json-scada submodule).

## Verification

- `grep -c "ensureS7PlusAlarmIndexes" index.js` returns 2 (definition + call)
- `grep "limit(200)" index.js` returns no matches
- `grep "createIndex.*createdAt.*-1" index.js` matches line 181
- Connect callback calls `await ensureS7PlusAlarmIndexes()` on the line after `await ensureActiveTagRequestIndexes()`

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None.

## Self-Check: PASSED

- File exists: `/c/Users/tnielen/Documents/Levvel_PoC/dev/.claude/worktrees/agent-abc31389/json-scada/src/server_realtime_auth/index.js` — FOUND
- Submodule commit f07120d2 — FOUND (HEAD of json-scada)
- Parent repo commit cc27b84 — FOUND (HEAD of worktree-agent-abc31389)

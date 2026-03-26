---
gsd_state_version: 1.0
milestone: v1.4
milestone_name: — Tag Tree Browser
status: Ready to plan
stopped_at: 14-01-PLAN.md complete — all tasks done
last_updated: "2026-03-26T12:06:35.722Z"
last_activity: 2026-03-26
progress:
  total_phases: 4
  completed_phases: 3
  total_plans: 3
  completed_plans: 3
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-26)

**Core value:** Alarms from S7-1200/S7-1500 PLCs appear in json-scada via native protocol subscription — not polling — with full metadata (text, timestamp, ack state, associated values)
**Current focus:** Phase 14 — datablockbrowser

## Current Position

Phase: 15
Plan: Not started

## Performance Metrics

**Velocity:**

- v1.0: 2 plans, 2 days
- v1.1: 6 plans, 2 days (2026-03-18 → 2026-03-19)
- v1.2: 4 plans, 2 days (2026-03-23 → 2026-03-24)
- v1.3: 4 plans, 1 day (2026-03-25)

**By Phase:**

| Phase | Plans | Duration |
|-------|-------|----------|
| 1. End-to-End Alarm Pipeline | 2/2 | 2 days |
| 2. Driver Fixes | 2/2 | ~1 hour |
| 3. Read-Only Alarm Viewer | 2/2 | ~8 min |
| 4. Ack Write-Back | 2/2 | ~55 min |
| 5. Driver — RelationId Fields | 1/1 | ~30 min |
| 6. Driver — Startup DB Name Map | 1/1 | ~2 min |
| 7. Backend — Delete Endpoint + _id Exposure | 1/1 | ~15 min |
| 8. Frontend — Delete Buttons + Origin Columns | 1/1 | ~1h |
| 9. Driver Enrichment | 1/1 | ~10 min |
| 10. API Cap Removal | 1/1 | ~5 min |
| 11. Vue UI Enhancements | 2/2 | ~10 min |
| Phase 12 P01 | 10 | 2 tasks | 2 files |
| Phase 13-backend-api-datablocks-tag-endpoints P01 | 5m | 2 tasks | 1 files |
| Phase 14 P01 | 2min | 2 tasks | 4 files |

## Accumulated Context

### Decisions

See PROJECT.md Key Decisions table for full log.

**v1.4 architecture decisions (from research):**

- Tags fetched via realtimeData query (protocolSourceObjectAddress prefix match) — no dedicated PLC browse connection needed for v1.4 scope
- Tree built entirely client-side by parsing protocolSourceObjectAddress strings
- activeTagRequests upsert on expand enables TTL-based polling without a dedicated browse connection
- New-tab navigation via window.open + router.resolve().href (createWebHashHistory — no URL construction needed)
- JWT must be in localStorage (not sessionStorage) for cross-tab auth to work
- [Phase 12]: Upsert keyed on {connectionNumber, db_name} prevents duplicates on restart; UpsertDatablocks only called on browse success (stale data preserved on failure)
- [Phase 13-backend-api-datablocks-tag-endpoints]: protocolSourceBrowsePath used in listS7PlusTagsForDb (not protocolSourceObjectAddress per D-01); touchS7PlusActiveTagRequests direct upsert without realtimeData lookup per D-02; source: 'tag-tree' set on activeTagRequests upserts
- [Phase 14]: selectedConnection initialized to null (no pre-selection on load per D-01)
- [Phase 14]: window.open with /#/s7plus-tag-tree hash URL for new-tab navigation (createWebHashHistory requires # prefix)

### Pending Todos

None.

### Blockers/Concerns

- [Phase 15] JWT token storage location must be confirmed against AdminUI auth store before wiring window.open — if token is in sessionStorage, new tab opens unauthenticated (research Pitfall 4)
- [Phase 15] protocolSourceObjectAddress format in realtimeData must be confirmed against a real document before building tree-path parsing logic

## Session Continuity

Last activity: 2026-03-26
Last session: 2026-03-26T12:02:22.456Z
Stopped at: 14-01-PLAN.md complete — all tasks done
Resume file: None
Next action: `/gsd:plan-phase 12`

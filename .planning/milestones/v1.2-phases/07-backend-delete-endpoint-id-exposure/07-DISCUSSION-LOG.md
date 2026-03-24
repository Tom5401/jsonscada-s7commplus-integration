# Phase 7: Backend — Delete Endpoint + _id Exposure - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-23
**Phase:** 07-backend-delete-endpoint-id-exposure
**Areas discussed:** Delete response body

---

## Delete response body

| Option | Description | Selected |
|--------|-------------|----------|
| { ok: true, deletedCount: N } | Phase 8 can show count in toast for bulk ops | |
| { ok: true } | Simple success flag, no count | |
| 204 No Content | REST-idiomatic, no body | ✓ |

**User's choice:** 204 No Content
**Notes:** Phase 8 checks `response.ok` and removes rows optimistically. No body to parse.

---

## Delete error response

| Option | Description | Selected |
|--------|-------------|----------|
| 500 + JSON { error: msg } | Distinct from success; Phase 8 can show message | ✓ |
| 200 + JSON { error: msg } | Matches existing listS7PlusAlarms error pattern | |

**User's choice:** 500 + JSON { error: msg }
**Notes:** Auth failures (401/403) still handled by `authJwt.isAdmin` middleware — not 500.

---

## Claude's Discretion

- Empty-filter guard (reject `{ filter: {} }`) — deferred to planner
- Invalid ObjectId handling (400 vs skip) — deferred to planner
- Endpoint placement (inline vs auth.routes) — decided by Claude: inline in index.js, same pattern as listS7PlusAlarms

## Deferred Ideas

- Empty-filter guard preventing accidental full-collection wipe
- Invalid ObjectId validation in `ids` array

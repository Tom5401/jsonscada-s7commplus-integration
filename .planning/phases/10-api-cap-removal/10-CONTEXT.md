# Phase 10: API Cap Removal - Context

**Gathered:** 2026-03-25
**Status:** Ready for planning

<domain>
## Phase Boundary

Remove the 200-alarm hard ceiling from the `listS7PlusAlarms` endpoint and protect MongoDB query performance with a `{ createdAt: -1 }` index on `s7plusAlarmEvents` — both changes in the same atomic commit. Backend-only. One file: `json-scada/src/server_realtime_auth/index.js`.

</domain>

<decisions>
## Implementation Decisions

### Query Ceiling

- **D-01:** Remove `.limit(200)` entirely — no ceiling at all. The `{ createdAt: -1 }` index ensures query performance even as the collection grows. PoC simplicity wins over a magic-number ceiling.

### Index Creation

- **D-02:** Create only `{ createdAt: -1 }` on `s7plusAlarmEvents` — exactly as API-02 specifies. No additional indexes (e.g., `connectionName`) in this phase. Phase 11's source filter will work fine at PoC scale without an index.
- **D-03:** Follow the established `ensureActiveTagRequestIndexes()` pattern — create a parallel `ensureS7PlusAlarmIndexes()` function and call it alongside the existing index function at MongoDB connect time (line 2663). `createIndex()` is idempotent — safe on every restart.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Implementation File
- `json-scada/src/server_realtime_auth/index.js` — Contains the `listS7PlusAlarms` endpoint with `.limit(200)` at line 361; `ensureActiveTagRequestIndexes()` pattern at line 153; MongoDB connect + index call at line 2663.

### Requirements
- `.planning/REQUIREMENTS.md` §API-01, §API-02 — Acceptance criteria for this phase.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ensureActiveTagRequestIndexes()` (index.js line 153): Established pattern for idempotent index creation at startup. New `ensureS7PlusAlarmIndexes()` should mirror this structure (try/catch, `Log.log` on error).

### Established Patterns
- `createIndex()` (not `ensureIndex`) is the MongoDB driver call used in this codebase.
- Error paths in `listS7PlusAlarms` return HTTP 200 (pre-existing json-scada pattern — do not change).
- MongoDB connect callback at line 2659–2664: index functions are called right after `db = clientMongo.db(...)`.

### Integration Points
- `listS7PlusAlarms` endpoint (line 352–369): Remove `.limit(200)` from the `.find({}).sort({createdAt:-1}).limit(200).toArray()` chain.
- MongoDB connect block (line 2659): Add `await ensureS7PlusAlarmIndexes()` call alongside `ensureActiveTagRequestIndexes()`.

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches beyond decisions above.

</specifics>

<deferred>
## Deferred Ideas

- `connectionName` index on `s7plusAlarmEvents` — deferred to Phase 11 or later; PoC scale makes it unnecessary now.

</deferred>

---

*Phase: 10-api-cap-removal*
*Context gathered: 2026-03-25*

# Phase 10: API Cap Removal - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-25
**Phase:** 10-api-cap-removal
**Areas discussed:** Query ceiling, Index scope

---

## Query Ceiling

| Option | Description | Selected |
|--------|-------------|----------|
| Remove entirely | No limit at all — simplest code, index protects performance | ✓ |
| Raise to 10,000 | Hard ceiling raised to 10,000 — safety net if collection grows | |

**User's choice:** Remove entirely
**Notes:** PoC constraint favors simplicity. Index provides performance protection.

---

## Index Scope

| Option | Description | Selected |
|--------|-------------|----------|
| createdAt only | Exactly what API-02 specifies — `{ createdAt: -1 }` | ✓ |
| createdAt + connectionName | Add connectionName now for Phase 11 source filter | |

**User's choice:** createdAt only
**Notes:** Phase 11's connectionName filter will work at PoC scale without an index. Deferred.

---

## Deferred Ideas

- `connectionName` index — reviewed, deferred (not needed at PoC scale; Phase 11 can add if needed)

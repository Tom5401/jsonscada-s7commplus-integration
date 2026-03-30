# Phase 16: Backend — `listS7PlusChildNodes` Endpoint + Index - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Session date:** 2026-03-30
**Discussed areas:** hasChildren query scope, Response projection

---

## Area 1: `hasChildren` Query Scope

**Question:** How should the `hasChildren` boolean be derived for each direct child?

**Options presented:**
| Option | Description |
|---|---|
| A — Full-connection distinct | `distinct('protocolSourceBrowsePath', {connNum: N})` once, filter locally |
| B — Scoped distinct on child paths | `distinct(...)` scoped to `$in: [childFullPath...]` — one query, index-backed |
| C — N individual exists checks | One `countDocuments` per child — N+1 round trips |

**Selected:** B — Scoped distinct on child paths

**Rationale:** Scales to 100k+ tag databases; one extra round-trip; hits the new compound index tightly; avoids pulling all unique paths for the entire connection.

---

## Area 2: Response Projection

**Question:** Should `listS7PlusChildNodes` return full `realtimeData` documents or a projected subset?

**Options presented:**
| Option | Description |
|---|---|
| Full documents | No projection, simpler code |
| Projected subset (~10 fields) | Only fields needed by Phase 17 tree + Phase 18 write button |

**Initial selection:** Full documents

**Revised to:** Projected subset after reviewing the ~40-field full document against the 9–10 fields actually needed by the tree (Phase 17 display + Phase 18 write gate).

**Final fields in projection:** `_id`, `ungroupedDescription`, `protocolSourceBrowsePath`, `protocolSourceConnectionNumber`, `protocolSourceObjectAddress`, `type`, `value`, `valueString`, `commandOfSupervised`

---

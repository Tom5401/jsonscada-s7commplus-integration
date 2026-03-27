# Phase 13: Backend API — Datablocks & Tag Endpoints - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-26

---

## Area 1: API-04 Match Field

**Question:** Which field in realtimeData should API-04 use to find all tags belonging to a datablock? (`protocolSourceObjectAddress` is hex like `8A0E0000.8A0F0001` — not the symbolic name)

| Option | Description |
|--------|-------------|
| **protocolSourceBrowsePath prefix** ✓ | `{ $regex: '^dbName(\\.|$)' }` — symbolic, avoids false matches on DBName2 |
| display_name prefix | Full symbolic path including leaf tag name |
| Two-step via db_block_relid | Look up hex relid from s7plusDatablocks, then match hex prefix of protocolSourceObjectAddress |

**Selected:** `protocolSourceBrowsePath prefix` (recommended default)

**Context:** REQUIREMENTS.md said "protocolSourceObjectAddress starts with X (quoted DB name prefix)" but codebase analysis confirmed the actual stored value is the hex AccessSequence. `protocolSourceBrowsePath` holds the correct symbolic parent path.

---

## Area 2: API-05 Data Completeness

**Question:** The existing `touchActiveTags()` stores `pointKey` + `tag` alongside `connection+address`. The new HTTP endpoint only receives `{connectionNumber, protocolSourceObjectAddress}` pairs. Should it look up the full tag first?

| Option | Description |
|--------|-------------|
| **Direct upsert, no lookup** ✓ | Just upsert {connectionNumber, protocolSourceObjectAddress, expiresAt} — simpler PoC approach |
| Look up realtimeData first | Full record consistent with existing touchActiveTags() usage, but adds a query per batch |

**Selected:** `Direct upsert, no lookup` (recommended default)

**Context:** PoC simplicity preferred. OPC read service only needs the address field to do its polling — pointKey/tag are not required for the polling loop to function.

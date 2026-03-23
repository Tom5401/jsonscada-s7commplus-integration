# Requirements — v1.2 Alarm Origin & Cleanup

**Milestone:** v1.2
**Status:** Active
**Last updated:** 2026-03-23

## Active Requirements

### ORIGIN — Alarm Origin Enrichment

- [ ] **ORIGIN-01**: Driver stores `relationId` (BsonInt64) in each new `s7plusAlarmEvents` document
- [ ] **ORIGIN-02**: Driver stores `dbNumber` (uint extracted from `relationId >> 16 & 0xFFFF`) in each new alarm document
- [ ] **ORIGIN-03**: Driver builds a `RelationIdNameMap` (`Dictionary<uint, string>`) at startup via `GetListOfDatablocks` on the tag connection, before alarm subscription starts; browse failure produces an empty map without blocking alarm subscription
- [ ] **ORIGIN-04**: Driver stores `originDbName` (string, resolved from `RelationIdNameMap`; empty string if not found) in each new alarm document at write time
- [ ] **ORIGIN-05**: `listS7PlusAlarms` API returns `relationId`, `dbNumber`, and `originDbName` fields per document
- [ ] **ORIGIN-06**: Alarm viewer displays Origin DB Name as a new column
- [ ] **ORIGIN-07**: Alarm viewer displays DB Number as a new column
- [ ] **ORIGIN-08**: Alarm viewer displays RelationId (raw) as a new column

### DELETE — Alarm History Management

- [ ] **DELETE-01**: `listS7PlusAlarms` API returns `_id` field for each document (remove `{ _id: 0 }` projection exclusion)
- [ ] **DELETE-02**: `POST /Invoke/auth/deleteS7PlusAlarms` handles single-row delete (`{ ids: [...] }` body) with `authJwt.isAdmin` guard
- [ ] **DELETE-03**: `POST /Invoke/auth/deleteS7PlusAlarms` handles bulk delete by current viewer filter (`{ filter: { alarmState, alarmClassName } }` body)
- [ ] **DELETE-04**: Per-row Delete button removes the alarm document; row removed optimistically from table on success
- [ ] **DELETE-05**: "Delete Filtered (N)" toolbar button deletes all currently visible filtered rows; disabled when zero rows visible
- [ ] **DELETE-06**: Attempting to delete a Coming + unacked alarm shows a warning ("This alarm is still active on the PLC. Delete anyway?")

## Future Requirements

- Alarm log retention policy (auto-expire after N days)
- FB type name column (requires GetTypeInformation browse per DB; high complexity)
- Symbolic variable path within DB (requires Alid-to-variable-type-info matching; full DB browse)
- Auto-resubscribe after alarm subscription connection failure (known tech debt)

## Out of Scope

- Polling-based alarm detection — native subscription only (carried from v1.0)
- Upstream contribution to json-scada — internal PoC use only
- Multi-PLC load distribution or HA redundancy — single connection PoC
- Multi-language alarm text — LCID hardcoded to 1033 (English)
- Server-side guard preventing active-alarm deletion — client-side confirmation only (avoids PLC query in delete path)

## Traceability

| REQ-ID | Phase | Plan |
|--------|-------|------|
| ORIGIN-01 | — | — |
| ORIGIN-02 | — | — |
| ORIGIN-03 | — | — |
| ORIGIN-04 | — | — |
| ORIGIN-05 | — | — |
| ORIGIN-06 | — | — |
| ORIGIN-07 | — | — |
| ORIGIN-08 | — | — |
| DELETE-01 | — | — |
| DELETE-02 | — | — |
| DELETE-03 | — | — |
| DELETE-04 | — | — |
| DELETE-05 | — | — |
| DELETE-06 | — | — |

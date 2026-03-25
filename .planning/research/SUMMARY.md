# Project Research Summary

**Project:** S7CommPlus Alarm Viewer Enhancements (v1.3)
**Domain:** Industrial SCADA alarm viewer — incremental enhancements to an existing v1.2 driver and Vue UI
**Researched:** 2026-03-25
**Confidence:** HIGH

## Executive Summary

This milestone (v1.3) adds operator-facing improvements to an existing, production-validated alarm monitoring stack built on S7CommPlusDriver (C# / .NET 8), MongoDB, a Node.js REST API, and Vue 3 / Vuetify 3. Research across all four dimensions confirms that **no new packages, libraries, or architectural components are required**. Every protocol field, database field, and UI component capability needed for v1.3 is already present in the codebase. The work is almost entirely wiring existing capabilities together with small, targeted code changes — the largest single change is the "Ack All" button in Vue.

The recommended approach is a layered, dependency-driven build order: fix data quality at the driver level first (`isAcknowledgeable` field, `infoText` placeholder substitution), then remove the API data cap and add the MongoDB index in the same step, then add all Vue UI features last (sortable priority column, combined timestamp column, source PLC filter, Ack All button, page preservation). This order ensures the UI phases can be verified against real enriched data rather than patching around missing fields in stale documents.

The primary risks are all implementation-level rather than design-level. The two most consequential: using `Promise.all` instead of `Promise.allSettled` for bulk ack (one failed ack silently cancels remaining acks), and removing `.limit(200)` from the API without adding a `{ createdAt: -1 }` MongoDB index (causes full collection scans every 5 seconds). Both are easily avoided when the pitfalls checklist is applied. A secondary risk is using the wrong field key name in Vue (`alarmPriority` vs the actual stored field `priority`), which produces a silently broken sort column.

---

## Key Findings

### Recommended Stack

The v1.3 stack is identical to v1.2. All research confirmed that **no new npm or NuGet packages are required**. Vuetify 3.10.12 (already installed) provides `v-data-table` with native sortable columns and client-side pagination. MongoDB `mongodb ^7.0.0` (Node.js driver) supports the only API change needed (removing `.limit(200)`). Vue 3 `async/await` + `Promise.allSettled` handles bulk ack with no library additions.

**Core technologies (all already installed):**
- **S7CommPlusDriver (C# submodule):** PLC protocol — unchanged; `priority` and `alarmClass` already stored since v1.2
- **.NET 8 / C#:** Driver host — 3-line change to `BuildAlarmDocument()` in `AlarmThread.cs`
- **MongoDB:** Alarm event log + ack command queue — no schema migration; 1 new field added to new documents only
- **Node.js + mongodb driver:** REST API — 1-line change (remove `.limit(200)`) + idempotent `createIndex`
- **Vue 3:** UI framework — `filteredAlarms` computed, `Promise.allSettled` bulk ack loop, new `ref` for page state
- **Vuetify 3 v-data-table:** Already configured with `sortable: true` and `:items-per-page`; no new components needed

See `.planning/research/STACK.md` for full field-level evidence.

### Expected Features

All 8 v1.3 features are classified as P1 table stakes. No feature is deferred. Research confirmed all are achievable with LOW-to-MEDIUM effort.

**Must have (table stakes — all P1):**
- `isAcknowledgeable` boolean stored in MongoDB — enables correct Ack button behavior; derived from `alarmClass == 33`
- Placeholder substitution extended to `infoText` — two-line C# change; `ResolveAlarmText()` already handles this for `additionalTexts` and `alarmText`
- Remove 200-alarm API cap (raise to 5000) — unblocks existing pagination controls; must be paired with `createIndex`
- Single combined timestamp column (`2026-03-24_12:57:10.758` format) — replaces split date/time columns; requires manual JS formatter
- Sortable priority column — `priority` field already in every document; one header array entry in Vue
- Source PLC filter (connectionName) — `connectionName` already in every document; mirrors existing `alarmClassFilter` pattern
- Ack button gated on `isAcknowledgeable` — `v-if` guard in Vue template; use `!!item.isAcknowledgeable` for pre-v1.3 document safety
- "Ack All (N)" button with confirmation dialog — `Promise.allSettled` loop over `filteredAlarms`; reuses existing `ackS7PlusAlarm` endpoint

**Should have (differentiators — add after validation):**
- Priority chip color coding — only if varied priority values are confirmed in the PoC PLC configuration
- Ack All scope respects active connectionName filter — automatic consequence of basing the count on `filteredAlarms`

**Defer (v2+):**
- Server-side pagination for large alarm archives
- Alarm log retention/expiry policy
- Auto-resubscribe after alarm subscription failure

See `.planning/research/FEATURES.md` for full dependency graph and per-feature implementation notes.

### Architecture Approach

The v1.3 architecture makes no structural changes. The four-layer stack (PLC → C# driver → MongoDB → Node.js API → Vue) is unchanged. All v1.3 changes are additive modifications to existing components, with a single data-flow change: `infoText` gains placeholder resolution at write time, and `isAcknowledgeable` is computed at write time rather than client-side. The bulk ack design deliberately avoids a new backend endpoint — sequential calls from Vue to the existing `ackS7PlusAlarm` endpoint match the inherently serial S7CommPlus ack protocol and preserve the per-command audit trail without additional backend complexity.

**Major components and v1.3 changes:**
1. **`AlarmThread.cs` — `BuildAlarmDocument()`:** Add `isAcknowledgeable` field; ensure `infoText` is wrapped with `ResolveAlarmText()` — 3 lines changed
2. **`server_realtime_auth/index.js` — `listS7PlusAlarms`:** Remove `.limit(200)` and add `createIndex({ createdAt: -1 })` at startup — 2 lines changed
3. **`S7PlusAlarmsViewerPage.vue`:** Add priority column, combined timestamp column, source PLC filter + computed options, `currentPage` ref + `v-model:page` binding, Ack All button — UI-only changes, no new files

See `.planning/research/ARCHITECTURE.md` for full data-flow diagrams (v1.2 vs v1.3) and anti-pattern analysis.

### Critical Pitfalls

Research identified 10 pitfalls. The top 5 by consequence:

1. **`priority` field already exists — use `key: 'priority'` in Vue headers, not `key: 'alarmPriority'`** — a key mismatch produces a silently broken sort column. Verify with `db.s7plusAlarmEvents.findOne()` before writing any Vue code for this column.

2. **Remove `.limit(200)` and add `{ createdAt: -1 }` index in the same phase** — without the index, MongoDB performs a full collection scan on every 5-second poll. `createIndex` is idempotent and takes under 1 second at PoC scale; there is no reason to defer it.

3. **Use `Promise.allSettled`, not `Promise.all`, for bulk ack** — `Promise.all` short-circuits on the first rejection, silently leaving remaining alarms unacked with no operator feedback. `Promise.allSettled` processes all acks regardless of individual failures.

4. **`isAcknowledgeable` must NOT hide the Ack button** — alarm classes outside the known set (33, 37, 39, 43) will have `isAcknowledgeable: false` even if the PLC expects acknowledgement. Drive Ack button visibility from `ackState === false`; use `isAcknowledgeable` only for the Ack All count and as a display hint.

5. **Timestamp format `YYYY-MM-DD_HH:mm:ss.SSS` requires a manual JS formatter** — no built-in `Date` method produces this format. `toISOString()` uses `T` separator and is UTC-normalized; `toLocaleString()` is locale-dependent and drops milliseconds. Use local-time accessors (`getHours()`, not `getUTCHours()`) unless UTC convention is agreed.

See `.planning/research/PITFALLS.md` for all 10 pitfalls with warning signs, recovery steps, and a verification checklist.

---

## Implications for Roadmap

Based on the dependency graph from FEATURES.md and the build-order analysis from ARCHITECTURE.md, 3 phases are recommended.

### Phase 1: Driver Enrichment
**Rationale:** `isAcknowledgeable` must be stored in MongoDB before any Vue feature can depend on it. Bundling `infoText` substitution in the same phase limits driver rebuilds to one. These are the only C# changes in the milestone.
**Delivers:** All new alarm events carry `isAcknowledgeable: true/false` and have resolved `infoText` values in MongoDB. Historical documents are unaffected (no migration required).
**Addresses:** `isAcknowledgeable` stored field, `ResolveAlarmText` extension to `infoText`
**Avoids:** Pitfall 8 — confirm exact current state of `alarmText` vs `infoText` in `BuildAlarmDocument` lines 245–246 before making changes; do not add frontend substitution

### Phase 2: API Cap Removal + MongoDB Index
**Rationale:** The Node.js API change unlocks the full alarm history for the Vue layer. The MongoDB index must accompany the limit removal in the same phase to prevent performance regression. This is a minimal, low-risk phase that unblocks Phase 3 pagination and filter testing.
**Delivers:** `listS7PlusAlarms` returns all documents (up to 5000) sorted by `createdAt` desc; query performance protected by `{ createdAt: -1 }` index.
**Addresses:** Remove 200-alarm API cap
**Avoids:** Pitfall 5 — index creation must not be deferred; it is idempotent and has zero downside

### Phase 3: Vue UI Enhancements
**Rationale:** All Vue changes depend on Phase 1 data (`isAcknowledgeable`) and Phase 2 data volume being in place. Building them together minimises context-switching in the component file. Recommended sub-order within phase: (1) display columns — priority, timestamp; (2) filter — connectionName; (3) pagination page preservation; (4) Ack All button.
**Delivers:** Complete v1.3 UI — sortable priority column, combined timestamp, source PLC filter, page preservation across 5-second refresh, Ack button gated correctly, "Ack All (N)" with confirmation dialog.
**Addresses:** Priority column, timestamp column, source filter, ack button guard, Ack All button, page state preservation
**Avoids:** Pitfall 1 (`key: 'priority'`), Pitfall 3/4 (`Promise.allSettled`; targets only `!ackState && !!isAcknowledgeable`), Pitfall 6 (`v-model:page` binding), Pitfall 7 (manual timestamp formatter with local time), Pitfall 9 (no PrimeVue), Pitfall 10 (filter on `connectionName`, not integer `connectionId`)

### Phase Ordering Rationale

- **Driver before Vue:** `isAcknowledgeable` is a stored field. The Vue Ack All button counts `filteredAlarms.filter(a => !a.ackState && !!a.isAcknowledgeable)`. If the field is absent from documents (pre-Phase 1 deployment), this count is always 0 and the button is silently non-functional.
- **API before Vue:** Removing the 200-doc cap is what makes the pagination controls in `v-data-table` meaningful beyond the first 200 rows. Building Phase 3 against a 200-row ceiling would produce misleading test results.
- **Index with API, not deferred:** Performance protection costs one idempotent line. Splitting it into a separate phase adds unnecessary risk with zero benefit.

### Research Flags

No phase requires `/gsd:research-phase`. All implementation decisions are fully resolved by existing codebase evidence.

Phases with standard, well-documented patterns (skip research-phase):
- **Phase 1 (Driver):** `BuildAlarmDocument` modification is identical to existing field additions; `ResolveAlarmText` signature is confirmed in source.
- **Phase 2 (API):** Single-line removal + one idempotent index creation call.
- **Phase 3 (Vue):** All patterns (`v-data-table` sortable columns, computed filter options, `Promise.allSettled`, `v-model:page`) are established Vue 3 / Vuetify 3 patterns confirmed against the installed version.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All findings verified from source code and installed `package.json`; no assumptions or inferences |
| Features | HIGH | Each feature traced to specific source lines in `AlarmThread.cs`, `S7PlusAlarmsViewerPage.vue`, and `index.js` |
| Architecture | HIGH | Component boundaries and data flow confirmed by reading all affected files; no inferred behaviour |
| Pitfalls | HIGH | Every pitfall grounded in actual v1.2 code; warning signs verified against real field names and function signatures |

**Overall confidence:** HIGH

### Gaps to Address

- **`isAcknowledgeable` on pre-v1.3 documents:** Old documents will not have the field. Vue must use `!!item.isAcknowledgeable` (not `=== true`) throughout — particularly in the Ack All filter and the ack button guard. No migration needed for PoC.
- **Timestamp UTC vs local timezone:** `dai.AsCgs.Timestamp` in the C# driver is a `DateTime`. Whether it is stored as UTC or local affects which JS accessor to use in the Vue formatter. The Phase 3 implementor must verify the displayed time matches TIA Portal/PLCSIM for the same event before finalising `getHours()` vs `getUTCHours()`.
- **`alarmText` substitution state:** PITFALLS.md contains an apparent internal discrepancy about whether `alarmText` is already substituted in v1.2. ARCHITECTURE.md states it is currently stored raw; STACK.md states `ResolveAlarmText` is already applied to `alarmText`. The Phase 1 implementor must confirm the exact state of `BuildAlarmDocument` lines 245–246 before making changes to avoid double-substitution.

---

## Sources

### Primary (HIGH confidence — direct source code)
- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — `BuildAlarmDocument()`, `ResolveAlarmText()`, `AlarmClassNames` dictionary, `PendingAcks` ack dispatch path
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHmiInfo.cs` — `Priority` (byte, offset 8), `AlarmClass` (ushort, offset 12–13) confirmed in HmiInfo blob
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — existing headers, filters, ack/delete logic, pagination props, `connectionId`/`connectionName` rendering
- `json-scada/src/server_realtime_auth/index.js` — `.limit(200)` at line 361, single-ack endpoint pattern, `deleteS7PlusAlarms` filter whitelist
- `json-scada/src/S7CommPlusClient/MongoCommands.cs` — `s7plus-alarm-ack` dispatch, sequential `SendAlarmAck` path confirmed
- `json-scada/src/AdminUI/node_modules/vuetify/package.json` — Vuetify 3.10.12 confirmed as installed version
- `json-scada/src/AdminUI/package.json` — Vue ^3.4.31, Vuetify 3.10 declared dependencies

### Secondary (HIGH confidence — project documents)
- `.planning/PROJECT.md` — v1.3 target features, PoC constraints, `AlarmClass 33 = isAcknowledgeable = true`, known tech debt
- `.planning/milestones/v1.2-MILESTONE-AUDIT.md` — confirms `priority` and `alarmClass` added to `BuildAlarmDocument` in Phase 5 of v1.2

### Reference
- MDN `Promise.allSettled()` — does not short-circuit on rejection; correct pattern for independent bulk operations
- MDN `Date.prototype` — `getHours()` (local) vs `getUTCHours()` (UTC) distinction; no built-in `_`-separated millisecond formatter
- MongoDB docs: `createIndex` — idempotent; safe to call at server startup on an existing collection

---
*Research completed: 2026-03-25*
*Ready for roadmap: yes*

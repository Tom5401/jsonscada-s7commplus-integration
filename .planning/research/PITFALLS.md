# Pitfalls Research

**Domain:** S7CommPlus alarm viewer enhancements — priority, isAcknowledgeable, bulk ack, pagination, placeholder substitution, sortable columns, timestamp formatting (v1.3)
**Researched:** 2026-03-25
**Confidence:** HIGH (codebase analysis — all pitfalls grounded in actual v1.2 code)

---

## Critical Pitfalls

### Pitfall 1: `priority` and `alarmClass` Already Exist in the Schema — Do Not Rename or Migrate

**What goes wrong:**
A developer treats `alarmPriority` and `isAcknowledgeable` as new fields that require MongoDB schema migration or $set backfill of old documents. Time is spent on migration scripts, or the driver is modified to emit different field names than already exist. If the Vue `v-data-table` header key is `alarmPriority` but the MongoDB field is `priority`, the sort column silently produces no ordering.

**Why it happens:**
The v1.3 milestone description lists "alarmPriority number stored in every alarm document" as a target feature, implying it is new. But `BuildAlarmDocument` in `AlarmThread.cs` already writes `{ "priority", (int)dai.HmiInfo.Priority }` and `{ "alarmClass", (int)dai.HmiInfo.AlarmClass }` into every document (lines 255–256 of the current AlarmThread.cs). Both fields have been present since v1.2.

**How to avoid:**
- Read `BuildAlarmDocument` before designing any schema work. Confirm whether `priority` and `alarmClass` are already present.
- For v1.3, the only new driver-side field is `isAcknowledgeable` (boolean, derived from `alarmClass == 33` at write time).
- Do NOT rename `priority` to `alarmPriority` — the existing field is `priority` in both the C# document builder and in every existing MongoDB document.
- The Vue header for the priority sort column must use `key: 'priority'`, not `key: 'alarmPriority'`.

**Warning signs:**
- Vue `v-data-table` sort on priority column produces no visible ordering.
- MongoDB `sort({ alarmPriority: -1 })` returns unsorted results (field does not exist on documents).
- New alarm documents have both `priority` AND `alarmPriority` fields (double-write introduced by mistake).

**Phase to address:** First driver phase — verify existing field names before writing any new code.

---

### Pitfall 2: isAcknowledgeable Must Handle Unknown AlarmClass Values — Do Not Hide Ack Button Based on It

**What goes wrong:**
The driver maps `alarmClass == 33 → isAcknowledgeable: true` and all other classes to `false`. A PLC with project-specific alarm classes (classes 39, 43, 37 already in the `AlarmClassNames` dictionary) sends an alarm with an unmapped class. `isAcknowledgeable` is written as `false`. If the Vue viewer hides the Ack button when `isAcknowledgeable === false`, the operator cannot ack a legitimate alarm.

**Why it happens:**
The rule "class 33 = acknowledgement required" is correct for TIA Portal standard alarm classes, but TIA Portal allows user-defined alarm classes. The existing `AlarmClassNames` dictionary in `AlarmThread.cs` already has project-specific classes (39, 43, 37). If a project-defined class requires acknowledgement but is not class 33, the derivation silently misclassifies it.

**How to avoid:**
- Document the `class 33 = isAcknowledgeable` rule as the agreed derivation in CONTEXT.md. Accept false negatives for unknown classes.
- The Ack button visibility must remain driven by `ackState === false`, not `isAcknowledgeable`. Use `isAcknowledgeable` only for styling or display hints (e.g., show a badge, color the row), not to hide the Ack control.
- Do NOT completely hide the Ack button based solely on `isAcknowledgeable: false`.

**Warning signs:**
- Operator reports "Ack button missing for a Coming alarm that was supposed to be acknowledgeable."
- Alarm classes outside {33, 37, 39, 43} appear in the viewer with no Ack button but with `ackState: false`.
- All alarms from a new PLC configuration show `isAcknowledgeable: false` and are silently unackable.

**Phase to address:** Phase that adds `isAcknowledgeable` to the driver and Vue viewer.

---

### Pitfall 3: Bulk Ack Race — Sending Acks for Alarms Already Acked Between Render and Click

**What goes wrong:**
The operator sets Status=Incoming and clicks "Ack All." The Vue component iterates `filteredAlarms` and sends one `ackS7PlusAlarm` request per unacked alarm. Between the time the filtered list was rendered (up to 5 seconds old) and the time the last ack request hits the server, the PLC or another operator session may have already acked some alarms. Duplicate acks are sent to the PLC.

**Why it happens:**
The 5-second auto-refresh means `filteredAlarms` can be up to 5 seconds stale at the moment of button click. There is no snapshot-and-lock between "render" and "execute."

**How to avoid:**
- Send ack requests only for alarms where `ackState === false` in the current client-side `filteredAlarms` array.
- Accept duplicate acks as benign — the existing `SendAlarmAck` path in `AlarmThread.cs` does not error on already-acked alarms in tested behavior.
- Do NOT add a pre-flight check that re-fetches the list before bulk ack — adds complexity with no real benefit at PoC scale.
- Add all targeted IDs to `pendingAcks` immediately on button click (same pattern as single-alarm ack) so the UI shows spinners and prevents re-clicking.

**Warning signs:**
- "Ack All" triggers multiple ack requests for the same alarm ID (visible in browser network tab).
- AlarmThread logs show `SendAlarmAck` called twice for the same `cpuAlarmId` within one 5-second window.
- PLC Wireshark trace shows duplicate AckJob PDUs for the same alarm.

**Phase to address:** Vue component phase that adds the "Ack All" button.

---

### Pitfall 4: Bulk Ack Partial Failure — Using `Promise.all` Cancels Remaining Acks

**What goes wrong:**
"Ack All" sends 10 ack requests using `Promise.all`. Request #7 fails (network timeout, PLC busy). `Promise.all` rejects immediately and the remaining requests #8, #9, #10 are never awaited for their results. The alarms targeted by #8–#10 remain unacked with no indication to the operator.

**Why it happens:**
`Promise.all` short-circuits on the first rejection. In bulk operations over independent resources (each alarm ack is independent), this behavior is wrong.

**How to avoid:**
- Use `Promise.allSettled` for bulk ack: `await Promise.allSettled(targets.map(item => ackAlarm(item.cpuAlarmId, item.connectionId)))`.
- For any individual ack failure, remove the ID from `pendingAcks` on catch (same as existing single-ack error path) to allow retry.
- If any ack fails, show a brief warning: "Some alarms could not be acknowledged." This is not required for PoC but is straightforward to add.

**Warning signs:**
- After "Ack All," most alarms clear but one or two remain with spinner permanently.
- Browser console shows rejection for a specific cpuAlarmId but subsequent alarm acks in the array were never attempted.
- `pendingAcks.value.size > 0` remains non-zero after all alarms in `filteredAlarms` show `ackState: true`.

**Phase to address:** Vue component phase that adds "Ack All" button — `Promise.allSettled`, not `Promise.all`.

---

### Pitfall 5: Removing `limit(200)` Without Adding a MongoDB Index Causes Full Collection Scan on Every 5-Second Refresh

**What goes wrong:**
The fix for the 200-alarm cap removes `.limit(200)` from `listS7PlusAlarms`. The query becomes `find({}).sort({ createdAt: -1 }).toArray()` with no limit. Without a `{ createdAt: 1 }` index on `s7plusAlarmEvents`, MongoDB scans the entire collection on every 5-second poll. At 10,000 alarm documents, each poll takes several seconds, blocking the Node.js event loop and causing visible UI sluggishness.

**Why it happens:**
The collection was designed for 200-document display — no index was added in v1.1–v1.2 because the limit masked the performance problem. Removing the limit exposes it.

**How to avoid:**
- Add `{ createdAt: -1 }` index to `s7plusAlarmEvents` in the same phase that removes the limit. A one-liner at server startup: `await db.collection('s7plusAlarmEvents').createIndex({ createdAt: -1 })`. `createIndex` is idempotent and non-blocking in MongoDB.
- Consider also adding `{ alarmState: 1, ackState: 1 }` for filtered bulk ack and bulk delete operations.
- For PoC at < 1,000 documents: full scan is tolerable, but index creation costs nothing and should not be deferred.

**Warning signs:**
- `listS7PlusAlarms` response time > 500ms in browser network tab as collection grows.
- MongoDB slow query log shows `COLLSCAN` for `s7plusAlarmEvents` find operations.
- CPU spike on the json-scada host every 5 seconds aligned with the poll interval.

**Phase to address:** Same phase that removes the limit — index creation must be bundled, not deferred.

---

### Pitfall 6: Pagination Page Drift When New Alarms Arrive During Auto-Refresh

**What goes wrong:**
The operator is on page 3 (alarms 101–150). The auto-refresh fires and 5 new alarms arrive at the top of the sort order (`createdAt` descending). The `alarms.ref` array is replaced. Vuetify `v-data-table` recalculates pagination from the new array. If the page number `ref` is not bound with `v-model:page`, it resets to page 1. Even with `v-model:page` bound, the content of page 3 shifts because new items pushed existing items down — but the operator stays on page 3 rather than being thrown back to page 1.

**Why it happens:**
The current `fetchAlarms` pattern replaces `alarms.value = json` entirely on every poll. Without `v-model:page` binding, Vuetify resets pagination on each re-render.

**How to avoid:**
- Preserve the current page number across refreshes: `const page = ref(1)` + `v-model:page="page"` on `v-data-table`. This is a one-line change.
- Accept that content within a page shifts when new alarms arrive — this is expected behavior and acceptable for PoC.
- Do NOT implement "pause refresh while paginated" — keeping auto-refresh unconditional is simpler.

**Warning signs:**
- Operator clicks to page 2, refresh fires, operator is thrown back to page 1.
- Page count changes unexpectedly during active alarm activity.

**Phase to address:** Vue component phase that adds pagination — `v-model:page` binding is mandatory.

---

### Pitfall 7: Timestamp Format `YYYY-MM-DD_HH:mm:ss.SSS` Cannot Use Any Built-in JS Date Method

**What goes wrong:**
A developer replaces the existing `formatDate`/`formatTime` functions with a single `formatTimestamp` that calls `new Date(isoStr).toLocaleString()` or `toISOString()`. Neither produces `2026-03-24_12:57:10.758`. `toLocaleString()` is locale-dependent (e.g., `3/24/2026, 12:57:10 PM`). `toISOString()` produces `2026-03-24T12:57:10.758Z` — wrong separator (`T` not `_`) and UTC-normalized. `toLocaleTimeString()` omits milliseconds entirely.

**Why it happens:**
The format with `_` separator and milliseconds is non-standard. No JavaScript built-in produces it directly. `Intl.DateTimeFormat` does not support `_` as a separator.

**How to avoid:**
- Write a small manual formatter using local time accessors:
  ```javascript
  const pad = (n, w = 2) => String(n).padStart(w, '0')
  const formatTimestamp = (isoStr) => {
    if (!isoStr) return ''
    const d = new Date(isoStr)
    return `${d.getFullYear()}-${pad(d.getMonth()+1)}-${pad(d.getDate())}_` +
           `${pad(d.getHours())}:${pad(d.getMinutes())}:${pad(d.getSeconds())}.${pad(d.getMilliseconds(), 3)}`
  }
  ```
- Use `getHours()` (local time), not `getUTCHours()` — operators are in the same timezone as the SCADA server and expect times to match the PLC HMI display.
- The `timestamp` field in MongoDB is a `BsonDateTime` set from `dai.AsCgs.Timestamp` (C# DateTime). Confirm whether this DateTime is UTC or local in the C# driver before finalizing the JS accessor choice.
- Remove the two separate `date` and `time` header entries and the two slot templates. Replace with one `timestamp` header entry.

**Warning signs:**
- Timestamp column shows `Invalid Date` or `NaN` (isoStr is null or undefined).
- Displayed time is 1–2 hours different from the PLC HMI timestamp (UTC vs local mismatch).
- Milliseconds always show `.000` (`toLocaleTimeString()` omits them).
- Column shows `2026-03-24T12:57:10.758` instead of `2026-03-24_12:57:10.758` (`T` separator not replaced).

**Phase to address:** Vue component phase that adds the combined timestamp column.

---

### Pitfall 8: Placeholder Substitution in `alarmText` Already Runs Server-Side — Extending to `infoText` Is a Driver Change, Not a Frontend Change

**What goes wrong:**
A developer adds client-side JavaScript placeholder substitution (`@N%f@` pattern) in the Vue viewer for `alarmText`, not realizing that `BuildAlarmDocument` in `AlarmThread.cs` already calls `ResolveAlarmText(texts?.AlarmText ?? "", av)` before writing to MongoDB. The `alarmText` field in every MongoDB document already has placeholders resolved. Adding frontend substitution causes double-substitution or, if the frontend pattern is slightly different, corrupts the displayed text.

**Why it happens:**
The v1.3 milestone description says "Placeholder substitution extended to alarm text and info text." This describes extending the existing server-side `ResolveAlarmText` to also cover `infoText` (which currently uses `texts?.Infotext ?? ""` without substitution). A developer who reads the requirement without reading the C# code assumes all substitution is currently frontend-only.

**How to avoid:**
- Read `BuildAlarmDocument` and `ResolveAlarmText` in `AlarmThread.cs` before designing this feature. Currently: `alarmText` is substituted; `infoText` is NOT (raw `texts?.Infotext ?? ""`). `additionalTexts` array: all 9 slots already call `ResolveAlarmText`.
- The v1.3 change is to pass `infoText` through `ResolveAlarmText(texts?.Infotext ?? "", av)` in `BuildAlarmDocument` in C# — one line change.
- No frontend substitution code is needed. Do not add it.

**Warning signs:**
- Alarm text appears with double-substituted values in the viewer.
- `infoText` in the viewer still shows raw `@1%f@` placeholders after the v1.3 deployment (substitution was added to frontend instead of driver).
- `alarmText` in MongoDB shows raw `@N%f@` placeholders (substitution was accidentally removed from driver and moved to frontend only).

**Phase to address:** Driver phase that extends placeholder substitution — change is in `BuildAlarmDocument`, not in Vue.

---

### Pitfall 9: PrimeVue Is Not in the Stack — Sortable Columns Use Vuetify `v-data-table`

**What goes wrong:**
A developer imports PrimeVue `DataTable` to implement sortable columns, creating a dependency on a library not in the project stack. `vuetify 3.10.8` is already installed and the viewer already uses `v-data-table`. Adding `primevue` duplicates the component library, bloats the bundle, and introduces a conflicting design system.

**Why it happens:**
The milestone description may reference "PrimeVue" as a research comparison or as context. The actual viewer uses Vuetify `v-data-table`. The existing headers array already has `sortable: true` on most columns (alarmClassName, alarmText, cpuAlarmId, originDbName, dbNumber).

**How to avoid:**
- Adding sort for `priority` is one addition to the headers array: `{ title: 'Priority', key: 'priority', sortable: true }`. No new library.
- Adding `isAcknowledgeable` as a column is likewise `{ title: 'Ackable', key: 'isAcknowledgeable', sortable: true }`.
- The key must exactly match the MongoDB field name as returned by the API.
- Do NOT add `primevue` to `package.json`.

**Warning signs:**
- `package.json` shows `primevue` as a dependency.
- A second `DataTable` component appears in the Vue file alongside `v-data-table`.
- Sort on priority produces no reordering (key name mismatch with MongoDB field).

**Phase to address:** Vue component phase — adding sortable priority is a one-line change to the headers array.

---

### Pitfall 10: Source PLC Filter — `connectionName` vs `connectionId` Naming Must Be Consistent

**What goes wrong:**
A developer adds a source filter dropdown populated from `connectionName` values but sends `connectionId` (the integer) as the filter key to the backend or uses it in client-side filter logic. Filtering by integer connection number works only when the number is known; filtering by name is more robust. The existing column header already uses `key: 'connectionId'` but renders `item.connectionName || item.connectionId` — this asymmetry carries forward into any new filter logic.

**Why it happens:**
The known tech debt in PROJECT.md explicitly documents: "connectionId (MongoDB field name) vs connectionNumber (API body key) naming inconsistency." The viewer column renders both as a fallback, masking which field is authoritative.

**How to avoid:**
- The source filter dropdown should be populated from unique `connectionName` values (human-readable PLC names).
- Client-side filter logic (preferred approach since all data is in memory after removing the limit) should filter `alarms.value` by `alarm.connectionName === selectedConnection`.
- If `connectionName` is empty (which can happen if `srv.name` was empty at event time), show the integer `connectionId` as fallback in the dropdown — same pattern as the existing column render.
- Do NOT add a new backend filter field without sanitizing it as a string equality check.

**Warning signs:**
- Filter dropdown shows integer numbers (1, 2, 3) instead of human-readable names.
- Selecting a source filter shows zero results even though alarms from that PLC are visible when filter is "All."
- Backend receives `{ connectionId: "PLCSIM_Name" }` (string where int is expected).

**Phase to address:** Vue component phase that adds the source filter — client-side filter on `connectionName`.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| `isAcknowledgeable` computed client-side from `alarmClass` in the viewer | No driver change needed | Stale if the rule changes; history inaccurate | Never — field should be written server-side at event time so history is stable |
| Remove limit without adding index | Simpler code change | Full collection scan every 5s poll | Never — index must accompany limit removal |
| Pagination with full dataset in Vue state | No server-side pagination API changes needed | Memory proportional to collection size | Acceptable at PoC scale (< 10,000 docs); flag for production |
| `Promise.all` for bulk ack | Simpler than `Promise.allSettled` | One failure aborts all remaining acks | Never — use `Promise.allSettled` |
| Keep two separate Date + Time columns | Zero code change | Does not meet v1.3 display requirement | Never — requirement is explicit |
| `toLocaleTimeString()` for timestamp | One line | Locale-dependent, no milliseconds, no `_` separator | Never — manual formatter is required for the specified format |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| MongoDB `s7plusAlarmEvents` field names | Referencing `alarmPriority` in Vue header key | Field is `priority` — verify with `db.s7plusAlarmEvents.findOne()` |
| `isAcknowledgeable` on pre-v1.3 documents | `item.isAcknowledgeable === true` check throws on undefined | Use `!!item.isAcknowledgeable` — treats undefined as false |
| `listS7PlusAlarms` limit removal | Removing `.limit(200)` without index | Add `{ createdAt: -1 }` index in same phase; `createIndex` is idempotent |
| Vuetify `v-data-table` page reset on refresh | Not binding `v-model:page` | `const page = ref(1)` + `v-model:page="page"` preserves operator position |
| Bulk ack with `Promise.all` | One rejected ack aborts all remaining | `Promise.allSettled(targets.map(item => ackAlarm(...)))` |
| `ResolveAlarmText` scope | Adding frontend placeholder substitution for `alarmText` (already resolved server-side) | Only `infoText` needs the fix, and it is a C# change in `BuildAlarmDocument` |
| `timestamp` column sort | ISO timestamp strings in `v-data-table` default to string sort | ISO format sorts correctly lexicographically — no custom sort function needed |
| Source filter field | Using integer `connectionId` field for source filter dropdown | Populate from `connectionName`; filter client-side on `connectionName` |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| No `createdAt` index + no limit | Slow `listS7PlusAlarms`; Node event loop blocked every 5s poll | Add `{ createdAt: -1 }` index with limit removal | Noticeable at ~5,000 documents; significant at ~50,000 |
| Full alarm array in Vue state (no server-side pagination) | Browser memory grows proportionally; table rendering slows | Acceptable for PoC; document as known limitation | ~10,000 docs (~5MB) is fine; ~100,000 would require server-side pagination |
| Bulk ack sending N sequential awaits | 50 alarms with serial `await` takes 50+ seconds | Send requests in parallel with `Promise.allSettled` | Any bulk ack > 10 alarms with sequential await |
| No-limit fetch every 5s with large collection | Network payload grows with collection size | Acceptable at PoC scale; production needs server-side pagination | ~1,000 docs per fetch = ~500KB/5s = acceptable; 100,000 = not acceptable |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| New "Ack All" path bypassing `authJwt.isAdmin` | Any authenticated user triggers bulk PLC ack | "Ack All" reuses existing per-alarm `ackS7PlusAlarm` endpoint already guarded with `[authJwt.isAdmin]` — no new endpoint needed |
| `connectionName` filter passed unsanitized to MongoDB backend query | String could be used for query injection if filter path extended to backend | Keep filter client-side; if moved to backend, whitelist: `if (filter.connectionName) query.connectionName = String(filter.connectionName)` |
| `isAcknowledgeable` flag trusted from client to gate ack permission | Client sends `{ isAcknowledgeable: true }` to bypass ack restriction | `isAcknowledgeable` is a display hint only; ack is always driven by `cpuAlarmId`, never gated on the flag |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| "Ack All" sends ack for alarms with `ackState: true` | Noisy duplicate PLC commands | Filter `filteredAlarms` by `ackState === false` before iterating |
| Timestamp shows no milliseconds | Cannot distinguish two alarms from same alarm ID in same second | Always include `.SSS` milliseconds in the timestamp format |
| Timestamp shows UTC when operator is in local timezone | Alarm appears to have occurred hours different from PLC HMI display | Use `getHours()` (local), not `getUTCHours()`, unless UTC convention is agreed |
| Page resets to 1 on every 5-second refresh | Operator cannot read alarm details on any page except page 1 | Bind `v-model:page` to a persistent `ref` |
| "Ack All" on 200 alarms with serial awaits freezes UI for 10+ seconds | Operator thinks the system is broken | Use parallel `Promise.allSettled`; show spinner on the button while in progress |
| isAcknowledgeable hides Ack button for alarms with unmapped alarm class | Operator cannot ack a legitimate alarm | Drive Ack button visibility from `ackState === false` only; use `isAcknowledgeable` for styling only |
| Priority sort column uses wrong key name | Column header click has no effect | Key must match MongoDB field name exactly: `priority`, not `alarmPriority` |

---

## "Looks Done But Isn't" Checklist

- [ ] **`priority` field name**: Confirm the sortable priority column header uses `key: 'priority'` — run `db.s7plusAlarmEvents.findOne()` and check the field name in the actual document.
- [ ] **`isAcknowledgeable` on old documents**: Confirm the Vue template uses `!!item.isAcknowledgeable` (not `=== true`) to handle pre-v1.3 documents where the field is absent.
- [ ] **`infoText` placeholder substitution**: Trigger an alarm with a `@1%f@` placeholder in info text and verify the stored MongoDB document shows the resolved value, not the raw placeholder.
- [ ] **`alarmText` not double-substituted**: Confirm `alarmText` in MongoDB already contains resolved values — and that no frontend substitution code was added for `alarmText`.
- [ ] **Timestamp milliseconds present**: Verify the combined timestamp column shows `.758` (3-digit ms) for an alarm that arrived with a sub-second PLC timestamp.
- [ ] **Timestamp local vs UTC**: Verify the displayed time matches the time shown in TIA Portal/PLCSIM for the same event.
- [ ] **Page preserved across refresh**: Navigate to page 2, wait for two auto-refresh cycles, confirm still on page 2.
- [ ] **`createdAt` index exists**: Run `db.s7plusAlarmEvents.getIndexes()` after deploying the limit-removal phase — confirm `{ createdAt: -1 }` is present.
- [ ] **Bulk ack uses `Promise.allSettled`**: Inspect the "Ack All" handler source — confirm `Promise.all` is NOT used.
- [ ] **Source filter populates from `connectionName`**: Confirm the dropdown shows human-readable PLC names (e.g., "PLCSIM_Advanced"), not integers.
- [ ] **`isAcknowledgeable` stored at write time**: Trigger a new alarm after driver deployment and verify the MongoDB document has `isAcknowledgeable: true` or `false` (not a missing field).
- [ ] **No PrimeVue dependency added**: Confirm `package.json` does not include `primevue`.

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Priority column broken (wrong field key in header) | LOW | Update `key: 'alarmPriority'` to `key: 'priority'` in Vue headers array; rebuild frontend |
| `isAcknowledgeable` missing on old documents | LOW | Field populates on new events automatically; existing docs show `undefined` treated as `false` — no migration needed for PoC |
| `infoText` still has raw placeholders after v1.3 | LOW | Add `ResolveAlarmText` call to `infoText` in `BuildAlarmDocument`; restart driver; old documents retain raw placeholders (acceptable) |
| Limit removed but no index — performance degradation | LOW | `db.s7plusAlarmEvents.createIndex({ createdAt: -1 })`; takes < 1 second on PoC-scale collection |
| Page resets to 1 on refresh | LOW | Add `v-model:page` binding; no data change needed |
| Timestamp format wrong | LOW | Update `formatTimestamp` helper function in Vue; no data change |
| Bulk ack used `Promise.all` — one failure killed remaining | MEDIUM | Change to `Promise.allSettled`; retest; no data loss (acks are idempotent) |
| Source filter showing integers instead of names | LOW | Change filter population and logic to use `connectionName` field |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| `priority` already exists — use correct field name | First phase (driver or Vue, whichever comes first) | `db.s7plusAlarmEvents.findOne()` shows `priority` field |
| `isAcknowledgeable` unknown class fallback + Ack button driven by `ackState` | Phase that adds `isAcknowledgeable` to driver | Trigger alarm with unmapped class; confirm Ack button is still visible |
| Bulk ack race + duplicate ack | Phase that adds "Ack All" button | Check network tab for duplicate requests for same `cpuAlarmId` |
| Bulk ack partial failure — `Promise.allSettled` | Phase that adds "Ack All" button | Source inspection confirms `Promise.allSettled` used |
| No index after limit removal | Same phase as limit removal | `db.s7plusAlarmEvents.getIndexes()` confirms `createdAt` index |
| Pagination page drift | Phase that adds pagination | Navigate to page 2; wait 3 refreshes; confirm page number preserved |
| Timestamp format — manual formatter required | Phase that replaces date/time columns | Column shows `YYYY-MM-DD_HH:mm:ss.SSS` with local time and ms |
| Placeholder substitution is server-side for `alarmText` | Phase that extends to `infoText` | MongoDB `findOne()` shows resolved text in `alarmText`; raw placeholder in `infoText` before fix, resolved after |
| PrimeVue not in stack — use Vuetify sort | Phase that adds sortable columns | `package.json` has no `primevue`; sort works via header `sortable: true` |
| `connectionName` vs `connectionId` filter | Phase that adds source filter | Dropdown shows human-readable names; filtering returns correct alarms |

---

## Sources

- Codebase analysis: `json-scada/src/S7CommPlusClient/AlarmThread.cs` — `BuildAlarmDocument` already writes `priority` (line 255) and `alarmClass` (line 256); `ResolveAlarmText` applied to `alarmText` and all `additionalTexts`; `infoText` is NOT substituted (uses `texts?.Infotext ?? ""` directly)
- Codebase analysis: `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — `headers` array (existing `sortable: true` columns), `formatDate`/`formatTime` locale-dependent helpers, `pendingAcks` single-ack pattern, `connectionId` column renders `connectionName || connectionId`
- Codebase analysis: `json-scada/src/server_realtime_auth/index.js` — `listS7PlusAlarms` with `.limit(200)` and `.sort({ createdAt: -1 })`, no index; `ackS7PlusAlarm` per-alarm pattern; `deleteS7PlusAlarms` filter whitelist pattern
- Project context: `PROJECT.md` — known tech debt: "connectionId (MongoDB field name) vs connectionNumber (API body key) naming inconsistency"
- Stack: `.planning/codebase/STACK.md` — Vuetify 3.10.8 installed; PrimeVue not in stack
- v1.2 milestone audit: `.planning/milestones/v1.2-MILESTONE-AUDIT.md` — confirms `priority` and `alarmClass` added to `BuildAlarmDocument` in Phase 5
- MDN: `Date.prototype.toLocaleString()` — locale-dependent output, no `_` separator support, milliseconds omitted by default
- MDN: `Promise.allSettled()` — does not short-circuit on rejection; returns array of {status, value/reason} for all promises
- MongoDB docs: `createIndex` — idempotent; safe to call at server startup on existing collection

---
*Pitfalls research for: S7CommPlus alarm viewer enhancements v1.3 (priority, isAcknowledgeable, bulk ack, pagination, placeholder substitution, sortable columns, timestamp, source filter)*
*Researched: 2026-03-25*

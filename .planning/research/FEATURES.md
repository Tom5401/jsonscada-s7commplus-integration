# Feature Research

**Domain:** S7CommPlus alarm viewer enhancements + data enrichment (v1.3)
**Researched:** 2026-03-25
**Confidence:** HIGH — all findings derived directly from live codebase (AlarmThread.cs, S7PlusAlarmsViewerPage.vue, index.js, AlarmsHmiInfo.cs)

---

## Context: What Exists (v1.2 Baseline)

Already shipped. These are the foundation these features build on:

- `BuildAlarmDocument()` in AlarmThread.cs stores 18 fields per document. `priority` and `alarmClass` are **already stored** as `(int)dai.HmiInfo.Priority` and `(int)dai.HmiInfo.AlarmClass`. `isAcknowledgeable` is NOT yet stored.
- `ResolveAlarmText()` applies `@N%x@` placeholder substitution **only to additionalTexts** (AdditionalText1-9). `alarmText` and `infoText` are written raw without substitution (BuildAlarmDocument lines 245-246).
- `AlarmClassNames` dictionary maps class ID 33 to "Acknowledgment required" — the exact source of truth for `isAcknowledgeable` derivation.
- `listS7PlusAlarms` API has a hardcoded `.limit(200)` in server_realtime_auth/index.js. This is the sole bottleneck for pagination.
- `S7PlusAlarmsViewerPage.vue`: v-data-table already uses `items-per-page` with options `[25, 50, 100, 200]` and default 50 — pagination controls exist in the UI today; they hit the 200-cap at the data layer.
- Timestamp is stored as a single MongoDB `BsonDateTime` field. The Vue viewer splits it into two display columns via `formatDate()` and `formatTime()` helper functions.
- `connectionName` field stored in every alarm document (srv.name). Not yet exposed as a filter in the viewer.
- Filters: statusFilter (All/Incoming/Outgoing) and alarmClassFilter. No connectionName filter.

---

## Domain Questions Answered

### 1. How does isAcknowledgeable typically surface in industrial alarm viewers?

In TIA Portal WinCC, Ignition, and Wonderware InTouch, `isAcknowledgeable` is a boolean property of the alarm class configuration. Standard surface patterns:

- **Ack column visibility**: The Ack button/icon is only rendered for alarms where acknowledgement is required. For non-acknowledgeable alarms (logging classes, status events), the Ack column cell is empty or shows a dash — no button, no spinner. This is the dominant TIA Portal WinCC pattern.
- **Not a filter**: Acknowledgeable vs non-acknowledgeable is NOT typically a filter criterion. It controls whether the Ack UI affordance appears. Operators know their alarm class semantics.
- **Stored as a boolean**: Stored per document so the Vue viewer can check `item.isAcknowledgeable` without needing to replicate the class-33 logic in JavaScript.
- **In this codebase**: `alarmClass` and `alarmClassName` are already stored. Adding `isAcknowledgeable = (alarmClass == 33)` in `BuildAlarmDocument()` is one line. The Vue viewer already renders the Ack button conditionally based on `ackState` — adding `&& item.isAcknowledgeable` refines when the button appears.

**Verdict**: Derive in C# at document-write time (`alarmClass == 33`). Store as boolean `isAcknowledgeable` in MongoDB. In Vue: hide Ack button when `!item.isAcknowledgeable`. Confidence: HIGH (consistent with TIA Portal WinCC behavior, derivable from existing `AlarmClassNames` dictionary in codebase).

### 2. What UX patterns work for "Ack All filtered" with confirmation?

Standard industrial patterns (Ignition, TIA Portal WinCC Comfort, Wonderware):

- **Label with count**: Button text includes count of matching unacked alarms, e.g. "Ack All (7)". This prevents surprise about scope — operator sees exactly how many alarms will be acknowledged before clicking.
- **Confirmation dialog**: A single-step confirmation is standard for bulk operations. The confirmation shows the count again. Two-step (type to confirm) is excessive for an operations UI. No confirmation at all is dangerous for bulk ack.
- **Filter-scoped**: The operation acknowledges only what the current viewer filter shows. This matches the existing "Delete Filtered" pattern already in the viewer — same mental model.
- **Disabled when 0**: Button is disabled when `filteredUnackedAlarms.length === 0`. Same pattern as "Delete Filtered" button which uses `filteredAlarms.length === 0`.
- **What counts as "unacked"**: Only alarms where `ackState === false` AND `isAcknowledgeable === true`. Trying to ack an already-acknowledged alarm or a non-acknowledgeable alarm is a no-op that confuses operators.
- **API design**: POST endpoint receives array of `{cpuAlarmId, connectionNumber}` pairs derived from the filtered+unacked set. Sends each ack sequentially or concurrently — the existing `ackS7PlusAlarm` endpoint handles one at a time; bulk ack needs either a new multi-ack endpoint or N sequential calls.

**Verdict**: "Ack All Filtered (N)" button showing count of unacked+acknowledgeable alarms in current filter. Confirmation dialog shows count. Button disabled when count is 0. Sends N individual ack requests (reuses existing endpoint) in parallel via `Promise.all`. Confidence: MEDIUM (pattern from industrial HMI standards, implementation via existing endpoint).

### 3. How does pagination in alarm viewers typically work?

Industrial alarm viewers (Ignition, TIA Portal WinCC, json-scada's own EventsViewerPage pattern):

- **Client-side pagination on preloaded data**: Load all visible alarms into the client, let the table component handle pagination. This is what the current v-data-table already does. The 200-row API cap breaks this model — fix is to remove the cap, not to add server-side pagination.
- **Page size options**: Standard options are 25, 50, 100, 200, All (or "all"). The current viewer already has `[25, 50, 100, 200]`. This does not need to change except removing the 200-cap that makes "200" the practical maximum.
- **Sort persistence across pages**: Vuetify v-data-table maintains sort state within the session automatically. No special handling needed — the existing `sortable: true` column headers will work across pages.
- **Sort on priority**: Numeric sort on the `priority` field (byte, 0-255 in the protocol). Lower number = higher priority in TIA Portal convention. Column header click toggles ascending/descending. Vuetify handles this natively when `sortable: true` is set and the data is numeric.
- **Auto-refresh with pagination**: The 5s refresh call replaces `alarms.value = json` which resets page position to page 1 on every refresh. Standard fix: preserve current page across refreshes using v-data-table's `v-model:page` or by updating the array in-place. For a PoC this edge case is acceptable to leave as-is (page resets on refresh).
- **"All" page option**: Avoid adding "All" as a page size — with the 200-cap removed, the collection could be large. Displaying thousands of rows locks the browser. The existing options `[25, 50, 100, 200]` are sufficient.

**Verdict**: Remove `.limit(200)` from the Node.js API (or replace with a high value like 5000). The Vue v-data-table already handles pagination correctly. Add `priority` as a sortable column. No server-side pagination needed for a PoC. Confidence: HIGH (direct codebase evidence).

### 4. How does the @N%x@ placeholder format work for alarm text and info text fields?

From `ResolveAlarmText()` in AlarmThread.cs and the existing `AlarmTextPlaceholder` regex `@(\d+)%[a-zA-Z]@`:

- **Format**: `@N%f@` where N is a 1-based index into SD (associated value) slots (SD_1..SD_10), and `%f`/`%d`/`%s` is a format specifier that is currently ignored — the actual AssociatedValue.ToString() is used regardless.
- **Current scope**: `ResolveAlarmText()` is already implemented and proven for AdditionalText1-9. It is NOT called for `alarmText` (alarm text) or `infoText` (info text) — those are written raw from `texts?.AlarmText` and `texts?.Infotext` respectively.
- **Why it matters**: TIA Portal allows placing `@1%s@` type placeholders in the main alarm text (the text displayed in the Event text column), not just in additional texts. If a PLC programmer puts `@1%d@` in the alarm text (e.g. "Motor speed: @1%d@ RPM exceeded limit"), the current code stores the raw placeholder string instead of the resolved value.
- **Fix is trivial**: In `BuildAlarmDocument()`, replace `texts?.AlarmText ?? ""` with `ResolveAlarmText(texts?.AlarmText ?? "", av)` and similarly for `infoText`. The `ResolveAlarmText()` method already handles null/empty strings safely (first guard clause).
- **Risk**: None. `ResolveAlarmText()` returns the template unchanged if no `@` is present (fast path). For the majority of alarms with no placeholders in alarmText, behavior is identical to today.

**Verdict**: Extend `ResolveAlarmText()` calls to cover `alarmText` and `infoText` in `BuildAlarmDocument()`. Two-line change. Confidence: HIGH (direct codebase evidence, logic is identical to existing additionalTexts path).

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features the v1.3 milestone requires. Missing any makes the milestone incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| `isAcknowledgeable` boolean in MongoDB | Viewer must know which alarms require ack to show/hide the Ack button correctly; `alarmClass == 33` is available in every document already | LOW | One field added to `BuildAlarmDocument()`: `{ "isAcknowledgeable", dai.HmiInfo.AlarmClass == 33 }`. Class 33 is "Acknowledgment required" — confirmed from AlarmClassNames dictionary. |
| `alarmPriority` field in MongoDB documents | Operators and sortable column need a clean priority value; `priority` already stored as raw byte but under the name `priority` — needs verification it's exported by the API correctly | LOW | `priority` already stored in BuildAlarmDocument line 255 as `(int)dai.HmiInfo.Priority`. Verify it appears in `listS7PlusAlarms` response and add it to viewer headers. |
| Single combined timestamp column in viewer | Two columns (Date + Time) waste horizontal space; operators read timestamps as a unit | LOW | Replace `date` and `time` headers with a single `timestamp` header. Format: `2026-03-24_12:57:10.758` (matches PROJECT.md spec). Custom slot renders `new Date(iso).toISOString().replace('T','_').slice(0,23)`. |
| Sortable priority column in viewer | Operators need to triage high-priority alarms first; priority is numeric (0-255 byte) | LOW | Add `{ title: 'Priority', key: 'priority', sortable: true }` to headers array. Vuetify v-data-table handles numeric sort natively. |
| Source PLC filter in viewer | Multi-PLC deployments need to isolate alarms by connection; `connectionName` already stored in every document | LOW | Add v-select for `connectionName` filter. Options derived dynamically from `alarms.value` (same pattern as `alarmClassOptions` computed property). Filter added to `filteredAlarms` computed. |
| Remove 200-alarm API cap | With pagination controls already in the table, the 200-row limit is the only barrier to seeing more alarms | LOW | In server_realtime_auth/index.js: remove or raise `.limit(200)`. Raise to 5000 as a practical safety cap for PoC; do not remove limit entirely. |
| Extend placeholder substitution to `alarmText` and `infoText` | TIA Portal alarm text fields can contain `@N%x@` placeholders; current code leaves them unresolved in the main alarm text column | LOW | In `BuildAlarmDocument()`: wrap `texts?.AlarmText ?? ""` and `texts?.Infotext ?? ""` with `ResolveAlarmText(..., av)`. Two-line change. `ResolveAlarmText` already handles null/empty safely. |
| `isAcknowledgeable` controls Ack button visibility in viewer | Non-acknowledgeable alarms (logging class, status) should not show an Ack button — currently ALL alarms show an Ack button | LOW | In ackState template slot: add `v-if="item.isAcknowledgeable"` guard. When false, render nothing (or a dash) in that cell. |
| Ack All button (bulk ack filtered+unacked) | Operators with many active alarms need one-click bulk acknowledge; per-row ack is tedious at scale | MEDIUM | Button "Ack All (N)" where N = unacked acknowledgeable alarms in current filter. Confirmation dialog. Sends N parallel ack requests via `Promise.all`. Disables when count = 0. |

### Differentiators (Competitive Advantage)

Features beyond strict requirements that strengthen the PoC demonstration.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Ack button hidden (not just disabled) for non-acknowledgeable alarms | Cleaner table — logging-class alarms have no Ack affordance at all, matching TIA Portal WinCC behavior | LOW | Requires `isAcknowledgeable` table stakes feature. Show empty cell or `—` dash instead of button. |
| Priority column with visual indicator (chip color or badge) | Makes high-priority alarms visually distinct without sorting; e.g. priority 1-3 = red chip, 4-7 = orange, 8+ = grey | LOW | Pure Vue cosmetic. Only adds value if PLC actually uses varied priority values in the PoC environment. |
| Ack All scoped to current filter including connectionName filter | If Source PLC filter is active, bulk ack only affects that PLC's alarms | LOW | Natural consequence of basing "Ack All" on `filteredAlarms` computed property — no extra logic needed if filter is applied before computing unacked count. |

### Anti-Features (Commonly Requested, Often Problematic)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Remove API limit entirely (no cap) | "Show all alarms" seems desirable | MongoDB can return thousands of documents; no limit + large collection locks the browser during render | Replace `.limit(200)` with `.limit(5000)`. The v-data-table pagination then handles display safely. |
| Server-side pagination (API accepts page/offset) | Scales to unlimited history | Doubles backend complexity; adds page-state synchronization with the 5s auto-refresh; defeats the simple `alarms.value = json` refresh model | Client-side pagination with a reasonable API cap (5000). For PoC this is sufficient. |
| "All" option in items-per-page | "I want to see everything" | Browser DOM rendering 1000+ rows is slow; defeats the purpose of pagination | Keep `[25, 50, 100, 200]` options. 200 per page is the practical maximum for operator usability. |
| Ack All without confirmation | Faster workflow | Bulk ack is irreversible and affects the PLC; one accidental click acks every alarm on screen | One confirmation dialog showing count. Not two-step — just one modal. |
| `isAcknowledgeable` as a viewer filter | "Show only alarms I can ack" | Operators work with full alarm list; hiding non-acknowledgeable alarms removes context | Use `isAcknowledgeable` only to control Ack button visibility, not as a filter. |
| Separate `alarmPriority` field name vs existing `priority` field | Rename for clarity | `priority` is already stored and presumably already returned by the API; renaming breaks any existing consumer and requires a migration | Use the existing `priority` field name. Just expose it in the viewer. |

---

## Feature Dependencies

```
[isAcknowledgeable stored in MongoDB]
    └──enables──> [Ack button hidden for non-acknowledgeable alarms in viewer]
    └──enables──> [Ack All button counts only acknowledgeable+unacked alarms]

[priority stored in MongoDB (already exists)]
    └──enables──> [Sortable priority column in viewer]

[Remove 200-alarm API cap]
    └──unblocks──> [Existing pagination controls in v-data-table show rows beyond page 1]

[ResolveAlarmText extended to alarmText + infoText]
    (no viewer dependency — purely a data quality improvement at write time)

[connectionName in MongoDB (already exists)]
    └──enables──> [Source PLC filter in viewer]
                      └──enhances──> [Ack All scoped to filter]

[Single timestamp column]
    └──replaces──> [Separate date + time columns]
    (no other dependencies — pure display change)

[Ack All button]
    └──requires──> [isAcknowledgeable in MongoDB] (to count only ackable alarms)
    └──reuses──> [existing ackS7PlusAlarm endpoint] (N parallel calls)
    └──scoped by──> [filteredAlarms computed property] (existing)
```

### Dependency Notes

- **`priority` field**: Already stored by BuildAlarmDocument as `(int)dai.HmiInfo.Priority`. Verify it is returned by `listS7PlusAlarms` (no projection exclusion in current API — it returns all fields via `find({}).toArray()`). Only work needed: add it to the Vue headers array.
- **`isAcknowledgeable` derivation**: `AlarmClassNames` dict already maps class 33 to "Acknowledgment required". Boolean is derived as `dai.HmiInfo.AlarmClass == 33`. If other alarm classes ever require ack, this logic becomes a set lookup — but for this PoC with the 4 known classes, direct comparison is sufficient.
- **Ack All endpoint vs N calls**: The existing `ackS7PlusAlarm` endpoint handles one alarm. Two options: (a) call it N times in parallel via `Promise.all` in Vue — simple, no backend change; (b) add a bulk ack endpoint — cleaner for large N but adds backend complexity. For a PoC with likely <20 simultaneous alarms, option (a) is preferred.
- **Pagination and auto-refresh**: Removing the API cap while keeping the 5s refresh means the full alarm list is fetched every 5 seconds. For a PoC environment with a bounded alarm history, this is acceptable. If the collection grows large, the 5s refresh becomes a performance concern — outside PoC scope.
- **Timestamp column replacement**: `formatDate()` and `formatTime()` functions remain in the Vue component (harmless unused code, or delete them). The single combined column format `2026-03-24_12:57:10.758` uses underscore separator as specified in PROJECT.md — not a space, not ISO T.

---

## MVP Definition (v1.3)

### Launch With

- [ ] `isAcknowledgeable` stored in `s7plusAlarmEvents` documents — `alarmClass == 33`
- [ ] `ResolveAlarmText()` extended to cover `alarmText` and `infoText` in `BuildAlarmDocument()`
- [ ] Remove `.limit(200)` from `listS7PlusAlarms` (raise to 5000)
- [ ] Single combined timestamp column in viewer (format: `2026-03-24_12:57:10.758`)
- [ ] Sortable `priority` column added to viewer headers
- [ ] Source PLC filter (connectionName-based) added to viewer
- [ ] Ack button hidden when `!item.isAcknowledgeable`
- [ ] "Ack All (N)" button with confirmation dialog — bulk ack via N parallel ackS7PlusAlarm calls

### Add After Validation

- [ ] Priority chip color coding — only if varied PLC priorities are confirmed in PoC environment
- [ ] Ack All scope correctly excludes already-acked alarms — confirm unacked-only count logic

### Future Consideration (v2+)

- [ ] Server-side pagination for large alarm archives
- [ ] Alarm log retention policy (auto-expire after N days)
- [ ] Auto-resubscribe after alarm subscription failure

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Remove 200-alarm API cap | HIGH (fixes broken pagination) | LOW (one line change) | P1 |
| Single timestamp column | MEDIUM (UX clarity) | LOW (pure Vue display) | P1 |
| Sortable priority column | HIGH (triage support) | LOW (add header + isAcknowledgeable dep) | P1 |
| `isAcknowledgeable` stored in MongoDB | HIGH (enables correct Ack button behavior) | LOW (one field in BuildAlarmDocument) | P1 |
| Ack button gated on isAcknowledgeable | HIGH (correctness) | LOW (one v-if in Vue template) | P1 |
| Source PLC filter | MEDIUM (multi-PLC clarity) | LOW (same pattern as alarmClass filter) | P1 |
| Extend @N%x@ to alarmText/infoText | MEDIUM (data quality) | LOW (two-line change in C#) | P1 |
| Ack All button | HIGH (operator efficiency) | MEDIUM (N parallel calls + confirmation dialog) | P1 |
| Priority chip color coding | LOW (cosmetic) | LOW (pure Vue) | P2 |

---

## Implementation Notes by Feature

### isAcknowledgeable
- Location: `BuildAlarmDocument()` in AlarmThread.cs
- Change: Add `{ "isAcknowledgeable", dai.HmiInfo.AlarmClass == 33 }` to the BsonDocument
- Note: `alarmClass` and `alarmClassName` are already stored — this is a derived boolean, not new protocol work
- Old documents in MongoDB will not have this field; Vue should default to `item.isAcknowledgeable !== false` to avoid hiding the Ack button on legacy documents

### alarmPriority / priority
- Already stored as `priority` in MongoDB. No driver change needed.
- Only work: add `{ title: 'Priority', key: 'priority', sortable: true }` to Vue headers array

### Single timestamp column
- Remove `date` and `time` from headers array
- Add `{ title: 'Timestamp', key: 'timestamp', sortable: true }`
- Custom slot: format as `2026-03-24_12:57:10.758` using `new Date(item.timestamp).toISOString().replace('T','_').slice(0,23)`
- Note: MongoDB returns BsonDateTime as ISO string in JSON

### Ack All button
- Compute `filteredUnacked = filteredAlarms.filter(a => !a.ackState && a.isAcknowledgeable !== false)`
- Button disabled when `filteredUnacked.length === 0`
- Confirmation dialog: "Acknowledge {{ filteredUnacked.length }} alarm(s)?"
- On confirm: `Promise.all(filteredUnacked.map(a => ackAlarm(a.cpuAlarmId, a.connectionId)))`
- Reuses existing `ackAlarm()` function — no new endpoint needed

### Placeholder substitution for alarmText/infoText
- Location: `BuildAlarmDocument()` in AlarmThread.cs
- Change lines 245-246 from:
  - `{ "alarmText", texts?.AlarmText ?? "" }`
  - `{ "infoText", texts?.Infotext ?? "" }`
- To:
  - `{ "alarmText", ResolveAlarmText(texts?.AlarmText ?? "", av) }`
  - `{ "infoText", ResolveAlarmText(texts?.Infotext ?? "", av) }`

### Remove 200-alarm cap
- Location: server_realtime_auth/index.js line 361
- Change `.limit(200)` to `.limit(5000)`
- No Vue change needed — pagination already works

### Source PLC filter
- Add `connectionFilter` ref initialized to `'All'`
- Add computed `connectionOptions = ['All', ...new Set(alarms.value.map(a => a.connectionName).filter(Boolean)).sort()]`
- Add condition to `filteredAlarms` computed: `connectionFilter.value === 'All' || alarm.connectionName === connectionFilter.value`
- Add v-select in the filter row alongside existing status and alarm class selects

---

## Sources

- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — `BuildAlarmDocument()` lines 202-264, `ResolveAlarmText()` lines 270-293, `AlarmClassNames` dictionary lines 194-200
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHmiInfo.cs` — `Priority` (byte), `AlarmClass` (ushort) field definitions confirmed
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — existing headers array, filter computed props, ackAlarm function, pagination options
- `json-scada/src/server_realtime_auth/index.js` lines 350-370 — `.limit(200)` hardcoded cap confirmed
- `.planning/PROJECT.md` — v1.3 target features confirmed, constraints (PoC simplicity)

---

*Feature research for: S7CommPlus alarm viewer enhancements and data enrichment (v1.3)*
*Researched: 2026-03-25*

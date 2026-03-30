# Project Research Summary

**Project:** TagTreeBrowser Overhaul — v1.5
**Domain:** Industrial SCADA Tag Browser (Vue 3 + Vuetify 3 + Node.js + MongoDB)
**Researched:** 2026-03-30
**Confidence:** HIGH

## Executive Summary

v1.5 is a focused overhaul of an existing, working Vue 3 component (`TagTreeBrowserPage.vue`) in the json-scada AdminUI. All four targeted features — lazy tree loading, boolean value display, value writing, and non-datablock tag support — are additions to the current codebase, not greenfield work. No new npm packages are required. Every capability needed (Vuetify `load-children`, OPC WriteRequest endpoint, commandsQueue write path, non-datablock tag storage) is either already installed or already running in the codebase. The primary work is replacing the v1.4 pattern of eagerly loading all tags for a DB with an on-demand child-fetch pattern, and surfacing infrastructure that already exists but is not yet wired to the tree UI.

The recommended approach is sequential by dependency: first add the new `listS7PlusChildNodes` backend endpoint (exact `protocolSourceBrowsePath` equality match), then rewrite the tree's initial load and expand behavior around Vuetify's `load-children` prop, then layer in the boolean display fix, write dialog, and non-datablock rows on top of the working lazy foundation. The write path must route through the existing `/Invoke/` OPC WriteRequest handler — not a new endpoint — to preserve the `canSendCommands` authorization check, SOE log entry, and userActions audit trail that the existing handler provides.

The two highest risks are both Vuetify-specific: (1) `load-children` only fires when a node's `children` property is an empty array `[]` — `undefined` renders as a leaf with no expand toggle, making depth-2+ nodes silently unreachable; (2) children must be mutated in place (`push`) not replaced by reference assignment, which silently breaks Vuetify's internal reactivity model. Both are confirmed from direct inspection of Vuetify 3.10 source and are easy to avoid if the node shape contract is established before any tree code is written.

---

## Key Findings

### Recommended Stack

No new dependencies. The installed stack (Vuetify 3.10.12, Vue 3.4.x, vue-router 4.4.x, Vite 7.3.x) provides every capability required. Vuetify 3.10's `v-treeview` `load-children` prop is confirmed working from direct source inspection of `VTreeviewChildren.js`. The known 3.7.1 regression (GitHub #20450) predates the installed version.

**Core technologies:**
- **Vuetify 3.10 v-treeview with `load-children`**: lazy tree expansion — fires once per node when `children.length === 0`; confirmed in installed `VTreeviewChildren.js:60-64`
- **MongoDB exact `protocolSourceBrowsePath` match**: direct-children query — returns only one depth level per expand; enables compound index `{protocolSourceConnectionNumber, protocolSourceBrowsePath}` to be fully exploited
- **Existing `/Invoke/` OPC WriteRequest (ServiceId 676)**: value write path — already handles auth, SOE logging, commandsQueue insert; reuse is mandatory, not optional
- **`commandsQueue` + S7CommPlusClient change stream**: write execution path — driver already watches this collection and calls `WriteValues()`; no driver change needed for the write feature

### Expected Features

**Must have (table stakes — all v1.5):**
- **TRUE/FALSE display for boolean tags** — frontend-only one-line fix; `0`/`1` is observably wrong compared to the tabular view; zero risk
- **Lazy tree loading** — prevents browser freeze on 100k+ tag databases; requires new `listS7PlusChildNodes` backend endpoint returning direct children via exact `protocolSourceBrowsePath` match
- **Value refresh scoped to expanded/visible nodes** — necessary companion to lazy loading; existing `touchExpandedLeafTags()` mechanism already provides the scaffold

**Should have (differentiators — all v1.5):**
- **Write/push values from tree leaf nodes** — operators can test PLC I/O without context-switching to tabular view; requires new `PushValueDialog.vue` and routes through existing OPC write infrastructure
- **Non-datablock tags (MArea, QArea, IArea) in DatablockBrowser** — memory bits and I/O areas are browsed by the driver and stored in `realtimeData` but currently invisible in the UI

**Defer (v1.x post-validation):**
- Write ack feedback (poll `commandsQueue` for `delivered: true`)
- Configurable refresh interval

**Defer (v2+):**
- WebSocket/SSE real-time push
- Tag search/filter within tree
- FB type name column (requires `GetTypeInformation` browse per DB — explicitly out of scope per PROJECT.md)

### Architecture Approach

The architecture is an incremental enhancement to three existing files plus one new component file. The backend stays in the single `index.js` file by convention — the new `listS7PlusChildNodes` endpoint is ~35 lines following the inline pattern of the five existing S7Plus endpoints. No new folders, no new services, no driver changes. The write dialog is extracted as `PushValueDialog.vue` for readability and potential reuse. Non-datablock area rows are virtual rows appended client-side in `DatablockBrowserPage.vue` — no backend change is needed because the driver already stores area tags in `realtimeData` with deterministic `protocolSourceBrowsePath` values (e.g. `"MArea"`, `"QArea"`).

**Major components:**
1. **`listS7PlusChildNodes` endpoint (NEW, ~35 lines in index.js)** — exact `protocolSourceBrowsePath` equality match + secondary `distinct` query for folder/leaf classification; add compound index at startup
2. **`TagTreeBrowserPage.vue` (MODIFIED)** — removes `buildTree()` full-load; adds `load-children` lazy expand + `formatLeafValue()` + write button per leaf
3. **`PushValueDialog.vue` (NEW)** — receives full leaf tag document, renders type-appropriate input (v-select for digital, v-text-field for analog), posts OPC WriteRequest to `/Invoke/`
4. **`DatablockBrowserPage.vue` (MODIFIED)** — appends 5 static virtual area rows (IArea, QArea, MArea, S7Timers, S7Counters) after fetching real DB list

### Critical Pitfalls

1. **`children: undefined` on folder nodes kills lazy loading** — Vuetify only shows an expand toggle and fires `load-children` when `children` is exactly `[]`. Set `children: []` on every non-leaf node; omit `children` only on confirmed leaf tags. Bugs here are invisible at depth-1 and only surface at depth-2+.

2. **Reusing the existing regex-prefix query for the lazy endpoint defeats the purpose** — `listS7PlusTagsForDb` uses `$regex: '^dbName(\\.|$)'` which returns the entire subtree. Each expand would download the full DB. The new endpoint must use exact equality on `protocolSourceBrowsePath` (which is the parent path per `ExtractPathFromName()` in `TagMapping.cs`, not the node's own path).

3. **In-place array mutation is mandatory for `load-children`** — `rawItem.children = fetchedChildren` (reference replacement) breaks Vuetify's reactivity; children do not render. Must use `rawItem.children.push(...fetchedChildren)`.

4. **Value write must go through `/Invoke/` OPC handler, not a custom endpoint** — The existing `case opc.ServiceCode.WriteRequest` at `index.js:754` enforces `canSendCommands`, creates SOE log entries, and updates `userActions`. A custom endpoint bypasses all of this silently.

5. **Non-datablock tags may have `protocolSourceBrowsePath = ""` in some driver versions** — The current driver names area tags `"MArea.X"` etc., yielding `protocolSourceBrowsePath = "MArea"`. But this must be verified with a direct MongoDB query on a live connection before Phase 5 is considered done.

---

## Implications for Roadmap

All four features have clear implementation dependencies. The architecture research provides an explicit 6-step build order; the roadmap should reflect this sequence. Phases 4 and 5 are independent of each other and can be parallelized.

### Phase 1: Backend Foundation — `listS7PlusChildNodes` Endpoint + Index

**Rationale:** Every other v1.5 feature depends on the ability to fetch direct children of a path. This endpoint must exist and be verified against real data before any frontend work begins. The `protocolSourceBrowsePath` parent-path semantics must be confirmed with a direct MongoDB query before writing the query.
**Delivers:** New auth-guarded endpoint returning direct child nodes with folder/leaf classification; compound index `{protocolSourceConnectionNumber, protocolSourceBrowsePath}` added idempotently at startup.
**Addresses:** Lazy tree loading (backend half); non-datablock area browsability (area paths use the same endpoint with no special-casing).
**Avoids:** Pitfall 2 (full-subtree regex query), Pitfall 3 (browse path semantics misunderstanding).

### Phase 2: Lazy Tree Loading — Rewrite `TagTreeBrowserPage.vue`

**Rationale:** Replaces the v1.4 full-load / `buildTree()` pattern. Must be stable before adding value display fixes, write buttons, or area rows — all three depend on the lazy tree being the live rendering model.
**Delivers:** `load-children` integrated into `v-treeview`; initial root-level load replaces full DB load; expand state preserved through value refresh cycles; scoped value refresh targeting only expanded leaves.
**Addresses:** Table-stakes lazy loading and scoped refresh requirements.
**Avoids:** Pitfall 1 (`children: []` vs `undefined`), Anti-Pattern 3 (in-place array mutation), Anti-Pattern 1 (full-load at open).

### Phase 3: Boolean Value Display Fix

**Rationale:** Completely independent of lazy loading once the tree renders correctly. A one-line template change plus `formatLeafValue()` helper. Can be bundled with Phase 2 in the same PR or shipped immediately after as a standalone fix.
**Delivers:** Digital tags display `TRUE`/`FALSE` matching the tabular view; analog and string tags unaffected.
**Addresses:** Table-stakes boolean display requirement.
**Avoids:** Pitfall 4 (raw `doc.value` display for digital tags).

### Phase 4: Value Write Dialog — `PushValueDialog.vue`

**Rationale:** Depends on Phase 2 being stable (write button is in the tree leaf append slot) and on `_id`, `type`, `stateTextTrue/False`, and `commandOfSupervised` being projected in the Phase 1 endpoint response.
**Delivers:** New `PushValueDialog.vue` component; write button on leaf nodes where `commandOfSupervised != 0`; write action routed through existing `/Invoke/` OPC WriteRequest handler.
**Addresses:** Differentiator write/push feature.
**Avoids:** Pitfall 5 (bypassing OPC handler, losing auth/SOE/audit trail); anti-pattern of reusing `dlgcomando.html` legacy popup (incompatible with Vue SPA — `window.opener` is null).

### Phase 5: Non-Datablock Tags — Virtual Area Rows in `DatablockBrowserPage.vue`

**Rationale:** Independent of Phase 4. Depends only on Phase 1 (the lazy endpoint already handles `path=MArea` identically to `path=MyDB`). The DatablockBrowserPage change is client-side virtual rows — no new endpoint, no driver change.
**Delivers:** 5 virtual area rows (IArea, QArea, MArea, S7Timers, S7Counters) in DatablockBrowser; each opens TagTreeBrowserPage with `?db=<AreaName>` identically to a DB row.
**Addresses:** Differentiator non-datablock tag support.
**Avoids:** Pitfall 6 (verify `protocolSourceBrowsePath` values in MongoDB for area tags before shipping).

### Phase Ordering Rationale

- Phase 1 first because it unblocks both the lazy tree (Phase 2) and the area rows (Phase 5).
- Phase 2 before Phase 4 because the write button is embedded in the tree leaf slot.
- Phase 3 can be done alongside Phase 2 (same file, independent change).
- Phases 4 and 5 are independent of each other and can be parallelized once Phase 2 is merged.
- The boolean fix (Phase 3) is the highest value-to-effort ratio change in the milestone — deliver it as early as possible.

### Research Flags

Phases with standard, well-documented patterns (no deeper research needed before planning):
- **Phase 1 (backend endpoint):** Follows identical inline pattern to `listS7PlusTagsForDb`; exact query and index specified in ARCHITECTURE.md. Run a direct MongoDB query to verify `protocolSourceBrowsePath` semantics — this is a 2-minute verification, not a research task.
- **Phase 3 (boolean display):** Single-function change. Pattern confirmed from `tabular.html` and `TagsCreation.cs`. No unknowns.
- **Phase 4 (write dialog):** OPC WriteRequest wire format documented in STACK.md and ARCHITECTURE.md from direct `tabular.html` inspection. No unknowns.

Phases needing a targeted verification step before coding:
- **Phase 2 (lazy tree):** After connecting `load-children` to the endpoint, run a depth-3 expansion smoke test immediately. Also test rapid expand-collapse-expand for duplicate children. The Vuetify behavior is confirmed from source, but these integration edge cases are worth catching before adding the write button in Phase 4.
- **Phase 5 (non-datablock):** Before shipping, run `db.realtimeData.find({ protocolSourceBrowsePath: "MArea" })` on a live connection. If the field is `""` instead of `"MArea"`, the virtual rows will open to an empty tree (silent failure requiring a one-line driver fix).

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All claims verified from installed `node_modules` source and project source files. No new packages. No speculation. |
| Features | HIGH | All four features traced end-to-end from UI requirement to existing backend infrastructure. Dependencies confirmed from codebase. |
| Architecture | HIGH | Component boundaries, data flow, query patterns, and build order all verified from direct source inspection. Exact query code provided in ARCHITECTURE.md. |
| Pitfalls | HIGH | All 6 critical pitfalls grounded in direct codebase inspection and Vuetify issue tracker. Recovery strategies included for each. |

**Overall confidence:** HIGH

### Gaps to Address

- **Non-datablock `protocolSourceBrowsePath` values need live verification:** ARCHITECTURE.md and STACK.md document that area tags use `protocolSourceBrowsePath = "MArea"` etc. based on the driver's naming convention in `S7CommPlusConnection.cs:878-880`. PITFALLS.md raises a counter-scenario where the field could be `""`. Verify with `db.realtimeData.find({ protocolSourceConnectionNumber: X, protocolSourceBrowsePath: "MArea" })` on a live PLC connection before completing Phase 5. If `""` is found, add a one-line driver fix to populate the path.

- **`commandOfSupervised` presence on area tags:** The write dialog disables the write button when `commandOfSupervised == 0`. For area tags (MArea/QArea), it is unknown whether the driver creates paired command tags. If not, write buttons will not appear for area leaf tags. This is not a regression (no write existed before) but should be noted in the Phase 4 task so testers know what to expect.

- **`listS7PlusChildNodes` projection must include write-dialog fields:** The write dialog needs `_id`, `type`, `stateTextTrue`, `stateTextFalse`, `commandOfSupervised`, and `protocolSourceObjectAddress`. These must be confirmed in the Phase 1 endpoint projection spec — the write dialog cannot be built correctly without them.

---

## Sources

### Primary (HIGH confidence — direct codebase inspection)
- `node_modules/vuetify/lib/components/VTreeview/VTreeviewChildren.js:15,60-64` — `load-children` trigger condition (fires when `children.length === 0`)
- `node_modules/vuetify/lib/components/VTreeview/VTreeview.d.ts:449-451` — `loadChildren: (item: unknown) => Promise<void>` type signature
- `json-scada/src/S7CommPlusClient/TagsCreation.cs:206-208, 279` — `stateTextTrue/False = "TRUE"/"FALSE"` for digital; `protocolSourceBrowsePath` assignment
- `json-scada/src/S7CommPlusClient/TagMapping.cs:135` — `ExtractPathFromName()` confirming `protocolSourceBrowsePath` = parent path (strips last dot-segment)
- `json-scada/src/S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs:878-882` — area node names: IArea, QArea, MArea, S7Timers, S7Counters
- `json-scada/src/AdminUI/public/tabular.html:875-960` — OPC WriteRequest wire format (ServiceId 676); digital value conversion
- `json-scada/src/server_realtime_auth/index.js:488-513, 754, 1143-1402` — existing endpoint patterns; WriteRequest handler; `canSendCommands` enforcement
- `json-scada/src/AdminUI/src/components/TagTreeBrowserPage.vue` — v1.4 baseline: `buildTree()`, `patchLeafValues()`, `touchExpandedLeafTags()`
- `json-scada/src/S7CommPlusClient/MongoCommands.cs` — `commandsQueue` document structure; change stream write execution path

### Secondary (MEDIUM confidence — Vuetify issue tracker)
- [Vuetify GitHub #20450](https://github.com/vuetifyjs/vuetify/issues/20450) — `load-children` regression in 3.7.1; resolved; does not affect 3.10
- [Vuetify GitHub #19983](https://github.com/vuetifyjs/vuetify/issues/19983) — empty `children` array shows expand arrow; confirms `[]` vs `undefined` contract
- [Vuetify GitHub #10175](https://github.com/vuetifyjs/vuetify/issues/10175) — `load-children` fires only once per empty array state; re-triggers only if array is cleared

---
*Research completed: 2026-03-30*
*Ready for roadmap: yes*

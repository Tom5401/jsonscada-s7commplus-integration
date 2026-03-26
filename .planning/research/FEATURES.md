# Feature Landscape

**Domain:** TIA Portal-equivalent hierarchical tag browser for S7CommPlus SCADA (v1.4)
**Researched:** 2026-03-26
**Confidence:** HIGH — findings grounded in live codebase (GUIBrowser Form1.cs, S7CommPlusConnection.cs, existing S7PlusAlarmsViewerPage.vue, server_realtime_auth/index.js) plus MEDIUM-confidence domain research from Ignition, OPC UA, and TIA Portal WinCC patterns.

---

## Context: What Exists (v1.3 Baseline)

Already shipped. These are the foundations v1.4 builds on:

- `GetListOfDatablocks()` in S7CommPlusConnection.cs — issues an ExploreRequest, returns `List<DatablockInfo>` with `db_name`, `db_number`, `db_block_relid`, `db_block_ti_relid` per block. Already called in `Program.cs` at driver startup to build `RelationIdNameMap`.
- `getTypeInfoByRelId(uint relTiId)` in S7CommPlusConnection.cs — returns a `PObject` whose `VarnameList.Names` and `VartypeList.Elements` describe all member variables of a datablock type, including Softdatatype, array bounds, and nested `RelationId` for UDT members. This is the lazy-expansion API — it is called per-node, on demand.
- `S7CommPlusGUIBrowser` (WinForms prototype) — demonstrates the full browse interaction: `GetListOfDatablocks` populates the root tree level; `AfterExpand` calls `getTypeInfoByRelId` to lazily expand each node; array members are expanded inline with index notation; nested UDTs add a `"Loading..."` placeholder child to trigger further expansion on click. This is the direct design reference for `TagTreeBrowser`.
- `S7PlusAlarmsViewerPage.vue` — `originDbName` column contains the datablock name. This is the integration point: clicking `originDbName` should open `TagTreeBrowser` for that specific datablock.
- `realtimeData` MongoDB collection — polled tags stored with `protocolSourceObjectAddress` (S7CommPlus AccessSequence address string), `value`, `valueString`, `type`, `tag` (`connName;address` format), `protocolSourceConnectionNumber`. This is how live values will be resolved.
- `activeTagRequests` MongoDB collection — TTL collection used to signal the driver which tags to poll. Inserting `{protocolSourceConnectionNumber, protocolSourceObjectAddress, expiresAt}` causes the driver to poll that address and update `realtimeData`.

---

## Domain Questions Answered

### 1. How do TIA Portal-equivalent tag browsers typically work?

TIA Portal's online monitor panel and tools like Ignition's Tag Browser, UaExpert (OPC UA), and Inductive Automation's OPC browser share a consistent UX contract:

- **Hierarchical tree at rest**: The top level shows blocks/datablocks/providers. No live data is loaded until a node is expanded or selected. This is the lazy-load contract.
- **Expand on demand**: Expanding a node triggers a protocol request (browse/type-info) for that node's children. Children appear with their name and type; leaf nodes (primitive variables) show the tag address alongside a value cell that starts empty or shows "—" until subscribed.
- **Live value column**: Once a leaf tag is visible (expanded), a live value column shows the current value polled from the PLC. Some tools poll only visible rows. Others show a spinner until first value arrives.
- **Data type column**: Always present. Operators and engineers need to know if they are looking at a Bool, Int, Real, or UDT member. In TIA Portal the type comes from the symbol table; in our case it comes from `Softdatatype` via `SoftdatatypeToAsdu()`.
- **Navigation from alarms**: TIA Portal WinCC and Ignition both support cross-navigation — clicking an alarm's origin block name opens that block in the tag monitor. This is an expected pattern at the PoC tier.

### 2. What are the table-stakes features for a DatablockBrowser?

From GUIBrowser reference implementation and industrial tool conventions:

- List of all datablocks by name (from `GetListOfDatablocks`), ordered alphabetically or by DB number.
- Each datablock shown as a clickable/expandable row with DB name and DB number.
- Clicking (or a "Browse" button) navigates to the TagTreeBrowser for that specific datablock, passing the `db_block_ti_relid` and `db_name` as route parameters.
- Connection selector — in a multi-PLC deployment, the user needs to choose which PLC's datablocks to view. The `protocolConnectionNumber` from `protocolConnections` is the discriminator.
- Load-on-page-enter: fetch the list when the page mounts. A loading spinner while the backend call resolves. An error state if the call fails.

### 3. What are the table-stakes features for a TagTreeBrowser?

From S7CommPlusGUIBrowser (the canonical reference for this codebase), OPC UA client UX (UaExpert, Ignition OPC browser), and TIA Portal online monitor:

- **Tree root = one datablock's type members**: Given a `db_block_ti_relid`, the root level shows all top-level variables in that block by calling `getTypeInfoByRelId`.
- **Lazy expansion**: UDT/struct members and array elements have a placeholder child ("Loading...") inserted at population time. When the user expands that node, the placeholder triggers a `getTypeInfoByRelId` call for the nested UDT's `RelationId`. This is exactly what `Form1.cs:AfterExpand` implements.
- **Array expansion inline**: Arrays are expanded eagerly at the level where they appear (as child nodes `varName[0]`, `varName[1]`, etc.), not lazily. This matches the GUIBrowser reference exactly. Multi-dimensional arrays use comma notation (`varName[0,0]`).
- **Data type column**: Every row shows the Softdatatype name (Bool, Int, Real, String, etc.). Derived from `Softdatatype` enum on `PVartypeListElement`.
- **Live value column**: Leaf-level primitive tags show a live value fetched from `realtimeData`. Non-leaf (UDT) nodes show "—" in the value column.
- **Tag address display**: Show the tag's S7CommPlus address (AccessSequence) so operators know the exact address for configuration or manual reference.
- **Node type icon**: TIA Portal uses icons to distinguish Bool, Int, Real, Struct, Array, FB instance. The GUIBrowser has a `SetImageKey()` method with 12+ image types. A web equivalent uses MDI icons (`mdi-toggle-switch`, `mdi-numeric`, `mdi-decimal`, `mdi-folder-outline`, etc.).

### 4. How should live values be displayed in a tag tree?

From the `activeTagRequests` + `realtimeData` pattern already established in this codebase:

- **Lookup by address**: `realtimeData` documents are keyed by `{protocolSourceConnectionNumber, protocolSourceObjectAddress}`. For a tag visible in the tree, the address is the AccessSequence from the browse result.
- **Two-tier resolution**: Tags already configured in `realtimeData` (polled by the driver) have live values immediately. Tags not yet configured need an `activeTagRequests` upsert to trigger polling — but this requires the driver to support `useActiveTagRequests`, which is already implemented.
- **Polling-on-demand is out of scope for PoC**: Triggering new tag subscriptions from the browser adds driver-side complexity. For v1.4, live values should be read-only from `realtimeData` for tags that the driver already polls. Tags not in `realtimeData` show "not configured."
- **Auto-refresh period**: A 5-second poll of the value endpoint is consistent with the alarms viewer. Operators expect near-real-time values, not true streaming.
- **Value format**: Show raw value for numerics. Show `true`/`false` for Bool. Show the string for string types. This matches `valueString` from `realtimeData` which the driver already populates.

### 5. What UX patterns work well for lazy tree expansion in industrial applications?

From OPC UA browser tools (UaExpert, Ignition OPC browser) and Vuetify v-treeview documentation:

- **"Loading..." placeholder**: Insert a dummy child with text "Loading..." when populating a parent node that has sub-members. On expand, replace the placeholder with real children. This prevents the expand arrow from disappearing (which would confuse users into thinking a node has no children) before the fetch completes.
- **Expand-on-click, not hover**: Industrial operators use deliberate clicks. Hover-triggered expansion creates accidental expansions in environments with imprecise pointing devices (touchscreens, trackballs).
- **Collapse does not re-fetch**: Once a node's children are loaded, collapsing and re-expanding should use the cached children. No need to re-fetch from the PLC unless an explicit refresh is requested. This is critical for perceived performance — OPC UA browsers universally cache expanded nodes in-session.
- **Spinner at node level**: Show a loading spinner next to the expanding node, not a full-page overlay. Full-page spinners block the tree while one node loads.
- **Depth limit**: In theory, UDTs can nest deeply. In practice, S7-1500 UDT nesting rarely exceeds 4-5 levels. No artificial depth limit is needed, but the implementation should handle circular references gracefully (check if `relTiId` was already expanded on this path).
- **Vuetify v-treeview**: The `load-children` prop supports async lazy loading. Returning a resolved Promise from `load-children` replaces the placeholder items with real children. This is the correct Vuetify mechanism — it avoids manual placeholder management and integrates with Vuetify's expand state.

### 6. What integration is expected between S7AlarmsViewer and TagTreeBrowser?

From TIA Portal WinCC cross-navigation patterns and the PROJECT.md v1.4 target:

- Clicking the `originDbName` cell in the alarms viewer opens TagTreeBrowser for that datablock in a new browser tab (`window.open`).
- The URL carries the datablock name and connection number as query parameters or route params: `/tag-tree?dbName=MyDB&connection=1`.
- TagTreeBrowser receives these parameters and pre-selects or auto-expands the matching datablock root node.
- No two-way navigation is required (TagTreeBrowser does not need a "back to alarms" button).

---

## Table Stakes

Features users expect for this milestone. Missing any makes the milestone incomplete.

| Feature | Why Expected | Complexity | Depends On |
|---------|--------------|------------|-----------|
| DatablockBrowser page — list all datablocks | Root entry point for the hierarchy; without it there is no navigation | LOW | New HTTP endpoint for datablock list (calls `GetListOfDatablocks`), new Vue component |
| DatablockBrowser in AdminUI main menu | Operators must be able to reach it from the nav drawer without knowing the URL | LOW | Vue Router entry + nav item in AdminUI |
| "Browse" action per datablock row — opens TagTreeBrowser | Table stakes navigation; the list is useless without a drill-down action | LOW | Router navigation with params |
| TagTreeBrowser page — lazy tree expansion of one datablock | Core of the milestone; mirrors TIA Portal online monitor | HIGH | New HTTP endpoint for type info, Vuetify v-treeview with load-children |
| Lazy expansion via `getTypeInfoByRelId` on node expand | Without lazy loading, the full deep tree is fetched at once — blocking and slow for large DBs | HIGH | HTTP endpoint wrapping `getTypeInfoByRelId` |
| Datablock name and DB number shown in DatablockBrowser | Operators identify DBs by both name and number; number is a TIA Portal convention | LOW | Fields already available in `DatablockInfo.db_name` and `.db_number` |
| Variable name and data type columns in TagTreeBrowser | Every industrial tag browser shows name + type; type is essential for value interpretation | LOW | `Softdatatype` available from `PVartypeListElement` |
| Array member expansion with index notation — `varName[0]` | S7-1500 DBs frequently use arrays of UDTs for recipe, setpoint, and logging structures; not supporting arrays makes the tree incomplete | MEDIUM | Logic from GUIBrowser Form1.cs lines 144-160 (1-dim) and 162-206 (multi-dim) |
| Live value column from `realtimeData` for configured tags | The core operator value: seeing what the PLC actually holds right now | MEDIUM | Backend query of `realtimeData` by `{connectionNumber, address}`; 5s poll in Vue |
| "Not configured" indicator for tags not in `realtimeData` | Operators must know when no live value is available (tag not polled by driver) | LOW | Live value fetch returns null/empty → display "—" |
| S7AlarmsViewer integration — `originDbName` click opens TagTreeBrowser | Operators investigating an alarm need to jump to the alarm's DB immediately | LOW | `window.open` with URL params from existing `originDbName` column |
| TagTreeBrowser accepts URL params (`dbName`, `connectionNumber`) | Navigation from alarms viewer passes these params; TagTreeBrowser must consume them | LOW | Vue Router `useRoute()` params |
| Connection selector in DatablockBrowser | Multi-PLC deployments exist; operators must choose which PLC to browse | LOW | List of connections from `protocolConnections` MongoDB collection |
| Loading spinner during fetch | Industrial tools universally show a spinner while PLC browse requests resolve; absence creates "is it broken?" confusion | LOW | Pure Vue — `v-progress-circular` while `loading.value === true` |
| Error state when browse fails | PLC offline or protocol error must surface clearly; silent empty tree misleads operators | LOW | HTTP endpoint returns error → Vue shows error message |

---

## Differentiators

Features beyond strict requirements that strengthen the PoC demonstration.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Node type icons (Bool, Int, Real, Struct, Array) | Matches TIA Portal visual language; engineers instantly recognize type without reading the type column | LOW | MDI icons mapped from `Softdatatype` categories. GUIBrowser `SetImageKey()` is the reference mapping. |
| Datablock search / filter in DatablockBrowser | PLC programs with 50+ datablocks are common; filter by name reduces time to target DB | LOW | Client-side filter on the loaded list; v-text-field bound to a search string, applied to `computed filteredDatablocks` |
| Expand-all for small DBs (depth limit 2) | Some operators want to see the full structure without clicking; useful for simple DBs with few members | MEDIUM | Only safe if guarded by a "small DB" heuristic (e.g., < 20 members at root) or a user-initiated "Expand All" button |
| Tag address column (AccessSequence) | Engineers configuring new tags need the exact address string; showing it in the tree eliminates the need to open a separate tool | LOW | `AccessSequence` field from `BrowseEntry`; only available for tags already mapped in `TagMapping.cs` — may not be available for pure type-info browse path |
| Live value auto-refresh indicator (last-updated timestamp) | Operators know if the displayed value is stale | LOW | `updatedAt` field already in `realtimeData` documents |

---

## Anti-Features

Features to explicitly NOT build for v1.4.

| Anti-Feature | Why Requested | Why Problematic for This PoC | What to Do Instead |
|--------------|---------------|------------------------------|-------------------|
| Write (force) tag values from the TagTreeBrowser | "I want to set a value from the browser" | Write-back requires command queue integration, SBO logic, access control, and PLC-side validation; dramatically expands scope | Read-only view; write commands remain in the existing command queue path |
| Real-time subscription / push updates for tree values | "I want values to update automatically without polling" | S7CommPlus subscription for arbitrary tag sets is complex; the existing `activeTagRequests` polling model is the PoC mechanism | 5-second poll on the displayed tags via `realtimeData` |
| Export tag list to CSV / Excel | "I want to document the PLC structure" | Out of scope for a browser diagnostic tool; adds file-download logic | Out of scope |
| Trigger new tag polling from browser (insert `activeTagRequests`) | "I want live values for tags not yet in `realtimeData`" | Requires driver-side handling of `useActiveTagRequests`; adds PLC load and driver coupling; scope creep for a read-only browse tool | Show "not configured" for tags absent from `realtimeData`; operator configures tags in the driver config |
| Full recursive auto-expand on page load | "Show me everything in one view" | `getTypeInfoByRelId` is one PLC request per node; auto-expanding a DB with 10+ UDT levels generates dozens of sequential PLC round-trips; blocks for seconds | Lazy expansion on user click |
| Multi-DB comparison view (side-by-side) | "Compare DB1 and DB2 structure" | No clear operator need in alarm-investigation context; complex layout | Single-DB focus; operator opens a second tab |
| Alarm origin DB auto-expand to the exact alarm tag | "Jump directly to the alarm variable in the tree" | Requires reverse-mapping from `cpuAlarmId` to a tag path — not currently stored; high complexity | Open TagTreeBrowser at the DB root; operator navigates to the relevant variable |
| Persist tree expansion state across browser refreshes | "Keep my navigation state" | Adds sessionStorage or URL complexity; tree state is ephemeral in all comparable industrial tools | Tree resets on page load; lazy load is fast enough that re-expanding is not painful |

---

## Feature Dependencies

```
[New HTTP endpoint: GET /Invoke/auth/listS7PlusDatablocks]
    └──enables──> [DatablockBrowser page — list populated]
    └──enables──> [Connection selector — connections list fetched from same backend session]

[New HTTP endpoint: GET /Invoke/auth/getS7PlusTypeInfo?relTiId=X&connectionNumber=Y]
    └──enables──> [TagTreeBrowser — lazy expansion of each node on AfterExpand]
    └──required by──> [Array expansion inline (1-dim and multi-dim)]
    └──required by──> [UDT nested expansion via HasRelation() nodes]

[DatablockBrowser page]
    └──navigates to──> [TagTreeBrowser page] (via router params: dbName, db_block_ti_relid, connectionNumber)

[S7PlusAlarmsViewerPage.vue — originDbName click]
    └──opens──> [TagTreeBrowser page] (window.open with URL query params)

[TagTreeBrowser page accepts URL params]
    └──requires──> [DatablockBrowser: db_block_ti_relid must be in the navigation payload]
    Note: db_block_ti_relid is NOT currently stored in alarm documents.
    The alarm document has dbNumber + originDbName. The TagTreeBrowser
    must either (a) resolve ti_relid from name via the datablock list endpoint,
    or (b) accept dbName and resolve on page load.

[realtimeData collection]
    └──enables──> [Live value column in TagTreeBrowser] (lookup by connectionNumber + address)
    └──requires──> [Tag must be configured and polled by driver — "not configured" fallback needed]

[Datablock name filter in DatablockBrowser — DIFFERENTIATOR]
    └──requires──> [DatablockBrowser list loaded]
    └──implemented as──> [client-side computed filter, no backend change]
```

### Dependency Notes

- **`db_block_ti_relid` is not in alarm documents**: Alarm documents store `dbNumber` and `originDbName`, not `db_block_ti_relid`. When navigating from alarms to TagTreeBrowser, the TagTreeBrowser must call the datablock list endpoint to resolve `db_block_ti_relid` by `db_name` match. This adds one network round-trip on TagTreeBrowser load-from-alarm-origin.
- **Live value address format**: `realtimeData.protocolSourceObjectAddress` uses the AccessSequence format (e.g., `8a0e0001.1.2.3`). The browse path from `getTypeInfoByRelId` returns `VarnameList` (variable names) and type info, not AccessSequences directly. Mapping tree nodes to `realtimeData` addresses requires the tag to already be registered in `realtimeData`. For the PoC, show live values only for tags where a `realtimeData` document exists with a matching address.
- **Backend architecture**: Both new endpoints (`listS7PlusDatablocks` and `getS7PlusTypeInfo`) must be in `server_realtime_auth/index.js` to reuse the existing MongoDB connection and auth middleware. They call C# driver logic indirectly — the Node.js layer queries a new MongoDB collection `s7plusDatablocks` that the C# driver populates, OR the Node.js layer calls a thin HTTP API on the C# driver. The simpler path for this PoC is: C# driver persists datablock list to `s7plusDatablocks` at startup (already done for `RelationIdNameMap`); type-info browse is done on-demand by a C# HTTP endpoint (new lightweight endpoint, not server_realtime_auth).
- **C# driver HTTP endpoint vs MongoDB round-trip**: `getTypeInfoByRelId` is a live PLC call (not a stored result). It cannot be served from MongoDB. The Node.js backend must either proxy to a C# HTTP server, or the browse service must be a separate C# ASP.NET endpoint. The existing pattern in this codebase is that json-scada server_realtime_auth is Node.js and drivers are C# background services with no HTTP listener. A new lightweight C# HTTP listener for browse requests is the cleanest approach without touching the existing driver structure.

---

## MVP Definition (v1.4)

### Phase Structure Recommendation

Based on dependencies, three phases:

**Phase A — Backend: DatablockBrowser data layer**
- C# driver persists `GetListOfDatablocks` result to a new `s7plusDatablocks` MongoDB collection at startup (add to `Program.cs` alongside the existing `RelationIdNameMap` build)
- Node.js `GET /Invoke/auth/listS7PlusDatablocks?connectionNumber=N` endpoint reading `s7plusDatablocks`
- Node.js endpoint for connections list (or reuse existing `protocolConnections` endpoint if one exists)

**Phase B — DatablockBrowser Vue page**
- `DatablockBrowserPage.vue` — fetches datablock list, v-data-table with Name + Number + Browse button
- Connection selector v-select
- Client-side name filter
- AdminUI router entry + nav drawer item

**Phase C — Backend + Vue: TagTreeBrowser**
- New C# HTTP listener (`DatablockBrowseService.cs` or similar) — lightweight ASP.NET minimal API, listens on a separate port (e.g. 5001), handles `GET /typeinfo?relTiId=X&connectionNumber=N`
- OR: persist type-info to MongoDB on demand (simpler but adds DB writes for every tree expand)
- `TagTreeBrowserPage.vue` — Vuetify v-treeview with `load-children` prop, live value column reading `realtimeData`
- S7PlusAlarmsViewerPage.vue: `originDbName` as clickable link

**Phase D — Integration + live values**
- `realtimeData` lookup from TagTreeBrowser for configured tags
- 5-second auto-refresh of visible tag values
- End-to-end navigation test: alarm → originDbName → TagTreeBrowser

### Launch With (strict MVP)

- [ ] `s7plusDatablocks` MongoDB collection populated by C# driver at startup
- [ ] `GET /Invoke/auth/listS7PlusDatablocks` endpoint in server_realtime_auth
- [ ] DatablockBrowserPage.vue: list with DB name + DB number + Browse button
- [ ] AdminUI nav entry for DatablockBrowserPage
- [ ] Backend type-info browse (per-node, on demand)
- [ ] TagTreeBrowserPage.vue: lazy tree, variable name + data type columns
- [ ] Array expansion (1-dim at minimum; multi-dim if time permits)
- [ ] Live value column: shows value from realtimeData if tag is configured; shows "—" if not
- [ ] S7PlusAlarmsViewerPage: originDbName as a link that opens TagTreeBrowserPage

### Add After Validation

- [ ] Multi-dimensional array expansion
- [ ] Node type icons (MDI icon per Softdatatype category)
- [ ] Datablock name filter in DatablockBrowser
- [ ] Last-updated timestamp on live values

### Future Consideration (v2+)

- [ ] Write-back / force value from browser
- [ ] Tag address (AccessSequence) column in tree for all nodes
- [ ] activeTagRequests insertion from browser (on-demand polling trigger)

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| `s7plusDatablocks` MongoDB collection | HIGH (all browser features depend on it) | LOW (extend existing Program.cs startup browse) | P1 |
| listS7PlusDatablocks API endpoint | HIGH (DatablockBrowser depends on it) | LOW (follow listS7PlusAlarms pattern) | P1 |
| DatablockBrowserPage.vue | HIGH (entry point to the hierarchy) | LOW (table + router nav) | P1 |
| AdminUI nav entry | HIGH (discoverability) | LOW (add one nav item) | P1 |
| Type-info browse backend | HIGH (TagTreeBrowser depends on it) | HIGH (requires C# HTTP endpoint or deferred-persist strategy) | P1 |
| TagTreeBrowserPage.vue with lazy expand | HIGH (core milestone feature) | HIGH (Vuetify v-treeview + async load-children) | P1 |
| Array expansion (1-dim) | MEDIUM (very common in S7-1500 programs) | MEDIUM (replicate GUIBrowser logic in Node.js response handler) | P1 |
| Live value column (realtimeData lookup) | HIGH (operator value) | MEDIUM (REST call per page refresh, match address) | P1 |
| originDbName click in AlarmsViewer | MEDIUM (UX integration proof point) | LOW (window.open with URL params) | P1 |
| Node type icons | LOW (cosmetic, not functional) | LOW | P2 |
| Datablock name filter | MEDIUM (usability for large PLC programs) | LOW (client-side filter) | P2 |
| Multi-dim array expansion | LOW (less common; 1-dim covers majority) | MEDIUM | P2 |
| Last-updated timestamp on values | LOW (informational) | LOW | P2 |

---

## Implementation Notes by Feature

### s7plusDatablocks MongoDB collection

- Location: `Program.cs`, immediately after the existing `GetListOfDatablocks` call that builds `RelationIdNameMap`
- Schema: `{ db_name: string, db_number: int, db_block_relid: uint (as BsonInt64), db_block_ti_relid: uint (as BsonInt64), connectionNumber: int, updatedAt: DateTime }`
- Write pattern: `ReplaceOneAsync` with upsert, keyed on `{connectionNumber, db_name}` — idempotent restart behavior
- Index: `{ connectionNumber: 1, db_name: 1 }` unique

### listS7PlusDatablocks API endpoint

- Pattern: identical to `listS7PlusAlarms` in index.js
- `GET /Invoke/auth/listS7PlusDatablocks?connectionNumber=N` → returns array of `{db_name, db_number, db_block_relid, db_block_ti_relid}` sorted by `db_name`
- Auth: `[authJwt.isAdmin]` — consistent with other S7Plus endpoints
- Confidence: HIGH (direct pattern match with existing code)

### Type-info browse backend (critical design decision)

- `getTypeInfoByRelId` is a live PLC call — cannot be served from a MongoDB cache without staleness risk
- Two options:
  1. **C# minimal HTTP API** (preferred for PoC): Add `DatablockBrowseService.cs` to S7CommPlusClient. Minimal ASP.NET Core app, single GET endpoint, listens on e.g. port 5001. Receives `relTiId + connectionNumber`, calls `srv.connection.getTypeInfoByRelId(relTiId)`, serializes `VarnameList + VartypeList` to JSON.
  2. **MongoDB on-demand cache**: On first expand, call C# driver via a command document; C# driver processes it and writes result to `s7plusTypeInfoCache` collection; Node.js polls for result. Complex handshake, high latency.
- Recommendation: Option 1. A minimal ASP.NET Core HTTP listener is ~30 lines and aligns with the PoC constraint of simplicity. Node.js proxies to `http://localhost:5001/typeinfo`.
- Confidence: MEDIUM (architectural pattern verified, not yet implemented)

### TagTreeBrowserPage.vue lazy expansion

- Vuetify component: `v-treeview` with `:load-children` async prop
- Root items: fetched from `listS7PlusDatablocks` for the selected connection; each item has `{ title: db_name, id: db_block_ti_relid, children: [] }` — `children: []` signals to v-treeview that children exist but are not loaded
- `load-children` callback: calls `GET /typeinfo?relTiId={item.id}&connectionNumber={conn}`, receives variable list, maps to child items
- Array items: expanded eagerly in the `load-children` callback (loop over array bounds, create child items with `[i]` suffix)
- UDT children: create child items with `children: []` placeholder if `HasRelation() === true`
- Leaf items: `children` omitted (no expand arrow)
- Confidence: HIGH (Vuetify v-treeview `load-children` pattern confirmed in Vuetify docs; GUIBrowser logic is the reference)

### Live value lookup

- After tree renders, for each visible leaf node: query `GET /realtimeData?connectionNumber=N&address=A`
- Or: batch query — POST array of addresses, returns map of `{address: {value, valueString, type, updatedAt}}`
- Simpler for PoC: query existing json-scada `queryJSON` endpoint or a new small endpoint in server_realtime_auth
- Poll every 5 seconds; update only the `value` column cells, not the tree structure
- Tags absent from `realtimeData` show "—" in value column

### originDbName click in S7PlusAlarmsViewerPage

- In the `item.originDbName` template slot: wrap in `<a @click.prevent="openTagBrowser(item)">{{ item.originDbName }}</a>` or a `v-btn` variant
- `openTagBrowser(alarm)`: `window.open('/tag-tree?dbName=' + alarm.originDbName + '&connectionNumber=' + alarm.connectionId, '_blank')`
- TagTreeBrowserPage mounts → reads `route.query.dbName` → calls `listS7PlusDatablocks`, finds matching entry, auto-starts browse for that DB
- Confidence: HIGH (window.open pattern; Vue Router query params)

---

## Sources

- `S7CommPlusDriver/src/S7CommPlusGUIBrowser/Form1.cs` — canonical reference for lazy expand, array handling, UDT recursion, SetImageKey() type mapping
- `S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs` lines 1184-1300 — `DatablockInfo`, `BrowseData` structs; `GetListOfDatablocks()` implementation
- `json-scada/src/S7CommPlusClient/Program.cs` lines 285-302 — existing `GetListOfDatablocks` call and `RelationIdNameMap` build pattern
- `json-scada/src/S7CommPlusClient/Common.cs` — `S7CP_connection` class, `RelationIdNameMap` field
- `json-scada/src/server_realtime_auth/index.js` — `listS7PlusAlarms`, `deleteS7PlusAlarms`, `ackS7PlusAlarm` patterns; `COLL_ACTIVE_TAG_REQUESTS` and `realtimeData` constants
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — existing viewer structure, filter patterns, auto-refresh, v-data-table usage
- `.planning/PROJECT.md` — v1.4 target features, constraints, validated history
- Vuetify v-treeview docs (vuetifyjs.com) — `load-children` prop behavior for async lazy loading (MEDIUM confidence — verified pattern exists)
- Ignition Tag Browser documentation (inductiveautomation.com) — search/filter, live value, data type column conventions (MEDIUM confidence — industry reference)
- OPC UA address space browser patterns (UaExpert, opcfoundation.github.io) — lazy expand, node caching, expand-on-click conventions (MEDIUM confidence — standard pattern)

---

*Feature research for: TIA Portal-equivalent hierarchical tag browsing — DatablockBrowser + TagTreeBrowser (v1.4)*
*Researched: 2026-03-26*

# Domain Pitfalls

**Domain:** Lazy PLC tag tree browsing added to existing S7CommPlus / Vue 3 / MongoDB SCADA system (v1.4)
**Researched:** 2026-03-26
**Confidence:** HIGH (all pitfalls grounded in codebase inspection + verified against official Vuetify issue tracker and S7CommPlusDriver source)

---

## Critical Pitfalls

### Pitfall 1: Browse Request on the Tag or Alarm Connection Causes PDU Interleaving

**What goes wrong:**
`S7CommPlusConnection` uses a single `Queue<MemoryStream>` protected by one `Mutex` per connection instance (`m_ReceivedPDUs` / `m_Mutex` in `S7CommPlusConnection.cs` lines 35–36). `WaitForNewS7plusReceived` polls this queue with a 2 ms sleep loop. If an ExploreRequest (type-info browse) is sent on a connection that is simultaneously receiving subscription notifications or tag data responses, the dequeue call in `WaitForNewS7plusReceived` will consume the wrong PDU. The browse response arrives but the browse caller dequeues a subscription notification instead; the browse returns an error or hangs until the 5 s `m_ReadTimeout`. The alarm or tag path then eventually processes the browse response, producing a parse failure or silent data corruption.

**Why it happens:**
The existing v1.2 architecture intentionally uses a second, separate `S7CommPlusConnection` (`alarmConn`) for the alarm subscription to avoid exactly this problem — it is a KEY DECISION in PROJECT.md: "Separate S7CommPlusConnection for alarm thread — avoids shared state and PDU interleaving." Routing on-demand browse through either the tag connection (`srv.connection`) or `alarmConn` reintroduces the same interleaving problem.

**Consequences:**
- Browse request times out or returns corrupt data.
- Tag polling may drop a read response (misdelivered to browse caller), causing one poll cycle of stale values.
- Sequence-number mismatch triggers `checkResponseWithIntegrity` error path (S7CommPlusConnection.cs line 367), which sets `m_LastError = errIsoInvalidPDU` and can terminate the connection.

**Prevention:**
Open a third, dedicated `S7CommPlusConnection` exclusively for browse operations. Follow the same pattern as `alarmConn`: connect, use, and either disconnect after each browse session or keep alive but idle. Do NOT reuse `srv.connection` or `alarmConn` for browse PDUs.

**Detection:**
- Browse HTTP endpoint returns timeout or empty results intermittently.
- Driver console shows `checkResponseWithIntegrity: ERROR! SeqenceNumber of Response` after a browse is triggered from the UI.
- Tag values show "stale" quality briefly after a browse operation.

**Phase to address:** The phase implementing the backend browse endpoint — before any HTTP handler calls `GetListOfDatablocks` or `getTypeInfoByRelId`.

---

### Pitfall 2: Blocking Browse Calls Hang the Node.js Event Loop

**What goes wrong:**
`S7CommPlusConnection.GetListOfDatablocks` and `getTypeInfoByRelId` are synchronous blocking operations (the C# driver polls with a 2 ms sleep loop for up to `m_ReadTimeout` = 5000 ms). When the Node.js backend exposes an HTTP endpoint that triggers a browse in real time, the Express request handler blocks for the full PLC round-trip. Node.js is single-threaded: a blocked handler freezes all other concurrent requests. For a PLC with hundreds of datablocks, `GetListOfDatablocks` can take multiple seconds (vendor documentation for similar drivers reports timeout values of 10 s are needed for large PLC configurations).

**Why it happens:**
The reference implementation `S7CommPlusGUIBrowser/Form1.cs` (line 61: `conn.GetListOfDatablocks(out dbInfoList)`) is a synchronous WinForms call — acceptable in a desktop GUI, fatal in a Node.js server. The json-scada pattern for adding new API calls is to add routes to `server_realtime_auth` (Node.js/Express).

**Consequences:**
- All in-flight API requests stall while browse is in progress, including the alarm viewer's 5 s auto-refresh.
- The alarm viewer shows a permanent spinner until the browse completes.
- Simultaneous browse requests from multiple tabs serialise in Node.js; each blocks for up to 5 s, totalling 10–15 s of unresponsiveness.

**Prevention:**
Pre-fetch and cache browse results in MongoDB. The C# driver writes the datablock list and type-info objects to a dedicated collection at startup (and optionally updates on demand via a MongoDB flag or HTTP trigger). The Node.js endpoint reads from MongoDB — a fast, non-blocking query. This is the same async-safe pattern used for `originDbName` enrichment and `GetListOfDatablocks` at startup in v1.2.

**Detection:**
- All API calls from the UI freeze for several seconds after a user expands a tree node.
- The alarm viewer pagination stops responding during tree expansion.
- `server_realtime_auth` logs show no output during the hang window.

**Phase to address:** Backend architecture phase — choose the caching strategy before writing any HTTP endpoint code.

---

### Pitfall 3: v-treeview `load-children` Fires on Node Select, Not Only on Expand

**What goes wrong:**
Vuetify 3's `v-treeview` fires the `load-children` callback when a node is selected, not only when expanded (confirmed in Vuetify issue #5543 and community reports). Simply clicking a datablock node label without expanding it triggers an HTTP browse request. With a slow PLC response this causes a visible freeze or spinner. Combined with selection events, `load-children` may fire twice for the same node before the first request completes.

**Why it happens:**
This is a known quirk inherited from Vuetify 2 that was not fixed in v3. The component fires the load callback on any event that causes child resolution, not only expand toggling.

**Consequences:**
- Unexpected PLC browse requests on node click.
- If `load-children` fires twice, both HTTP requests return, and the second response appends duplicate children (reported in Vuetify issue #8791: "selection-type='independent' + dynamically loaded children incompatibility").
- Race condition: two in-flight requests for the same datablock; the second response overwrites tree state set by the first, possibly reordering children.

**Prevention:**
Gate the `load-children` callback with a `loadedRelIds` Set. Before issuing the HTTP request, check whether the RelId is already in the Set. If it is, return immediately. Only add to the Set after a successful load. This prevents duplicate fetches regardless of what triggers the callback.

**Detection:**
- Browser DevTools network tab shows two identical `/browseTypeInfo?relId=...` requests fired back-to-back on node click.
- Tree shows duplicate child entries after a fast double-click on a datablock.

**Phase to address:** TagTreeBrowser Vue component — first version of the `load-children` implementation.

---

### Pitfall 4: Vuetify 3 v-treeview Performance Degrades with Large Node Lists and Deep Nesting

**What goes wrong:**
Vuetify 3.6.8 introduced a confirmed performance regression (issue #19919, May 2024) where `v-treeview` is significantly slower than `v-list` when expanding nodes with many children. A PLC datablock with hundreds of struct members or large arrays renders all child nodes as DOM elements on expand. In issue #21387 (Vuetify 3.8.4, May 2025), deep nesting (10+ levels) causes rendering glitches where nodes at lower levels become distorted, overlapping, or disappear entirely. S7-1500 FBs can legitimately be deeply nested (FB containing structs of structs with arrays).

**Why it happens:**
Vuetify 3 `v-treeview` does not implement virtual scrolling on the node list. Every visible node is a live Vue component consuming memory and triggering reactive updates.

**Consequences:**
- DatablockBrowser page stutters when a PLC has 200+ datablocks.
- TagTreeBrowser becomes unresponsive when a deeply nested FB is expanded.
- Nodes at depth 10+ may visually disappear (Vuetify bug #21387 signature).

**Prevention:**
- For the flat DatablockBrowser list (no nesting), use `v-list` with `v-virtual-scroll` instead of `v-treeview`. A plain list is faster and avoids all tree-specific bugs.
- For TagTreeBrowser, use lazy expansion exclusively: never pre-expand all nodes. If a struct member is an array of 1000 elements, cap display at a reasonable count (e.g. 100) with a "Show more" control rather than rendering all 1000 as tree nodes.
- Test on actual PLC data before committing to `v-treeview` for any component that shows more than ~200 leaf nodes expanded simultaneously.

**Detection:**
- DatablockBrowser page takes >1 s to render after the API response arrives.
- Chrome DevTools Performance tab shows long "Recalculate Style" and "Layout" tasks on expand.
- Deep FB expansion causes nodes to visually disappear.

**Phase to address:** DatablockBrowser and TagTreeBrowser Vue component phases — decide the component choice before building.

---

### Pitfall 5: Duplicate Node Names Cause Infinite Loop in v-treeview

**What goes wrong:**
Vuetify 3.7.0 introduced a confirmed bug (issue #20389, August 2024) where `v-treeview` enters an infinite loop when a tree contains a node and its parent that share the same name. This is a realistic PLC scenario: a datablock named "Motor" may contain a struct member also named "Motor" (common TIA Portal pattern when a DB holds an FB instance of the same name).

**Why it happens:**
Vuetify's internal tree node resolution uses the display name as part of the identity key when `:item-value` is not explicitly set to a unique field.

**Consequences:**
- Browser tab freezes completely when a user expands a datablock whose name matches a child member name.
- The freeze is unrecoverable without closing the tab.

**Prevention:**
Always set `:item-value="item => item.uniqueKey"` on `v-treeview` where `uniqueKey` is derived from the node's RelId and its position in the hierarchy (e.g. `relId + '_' + parentPath`), never from the display name. Every node must have a globally unique `value` field.

**Detection:**
- Browser tab hangs when expanding a datablock whose name matches a child member name.
- Chrome Task Manager shows CPU at 100% for the tab with no recovery.

**Phase to address:** TagTreeBrowser Vue component — the node schema must define a unique key before any tree rendering is attempted.

---

## Moderate Pitfalls

### Pitfall 6: JWT Authentication Not Available in the New Browser Window

**What goes wrong:**
`window.open('/tagbrowser?db=DB1')` opens a new browser tab that runs a fresh Vue app instance with an empty Pinia store. If AdminUI stores the JWT token exclusively in `sessionStorage`, the new window will not have it: `sessionStorage` is partitioned per top-level browsing context (each tab is its own partition). The TagTreeBrowser page loads but its first API call to `server_realtime_auth` receives a 401, and the navigation guard redirects to `/login`.

**Why it happens:**
This is a browser security boundary by design. `sessionStorage` is intentionally not shared across tabs. Only `localStorage` is shared across same-origin windows. Pinia stores are per-app-instance; the new tab's store starts empty.

**Consequences:**
- Clicking an alarm's origin DB name opens a new tab that immediately redirects to `/login`.
- The operator must log in twice.
- After logging in again, the `db` query parameter from `window.open` is lost in the redirect cycle.

**Prevention:**
Verify how AdminUI stores the auth token. If it uses `sessionStorage`, the options are: (a) switch to `localStorage` for the token — shared automatically across same-origin windows; (b) pass a short-lived token as a query param; (c) use `window.postMessage` from the opener after the child window signals readiness. Option (a) is the lowest-effort change. Preserve the `db` param through any login redirect by encoding it as a `redirect` param in the login URL.

**Detection:**
- New window shows the login page immediately after opening.
- Network tab in the new window shows 401 on the first API request.
- After completing login in the new window, the `db` query param is missing from the URL.

**Phase to address:** New-window navigation phase — verify storage mechanism before implementing the `window.open` call.

---

### Pitfall 7: getTypeInfoByRelId Returns null for Some Datablocks

**What goes wrong:**
`getTypeInfoByRelId` looks up a `PObject` from a list populated during `GetListOfDatablocks`. Some datablocks — system DBs, safety blocks, or blocks with partially decoded type info — return a `PObject` with `VarnameList == null` or with a count mismatch between `VartypeList.Elements` and `VarnameList.Names`. The reference implementation in `Form1.cs` line 136 guards against this: `if (pObj == null || pObj.VarnameList == null) return;`. The `Browser.cs` source also contains `// TODO: Special: ... Not known if it's a fixed or variable value.` (line 85), indicating not all type structures are fully resolved.

**Why it happens:**
`GetListOfDatablocks` returns all datablocks the PLC reports, including those the driver cannot fully decode. The browse path always has some blocks it cannot fully represent.

**Consequences:**
- HTTP 500 from the browse endpoint when a user expands a system datablock.
- The TagTreeBrowser tree shows an error indicator, which looks like a driver failure rather than an expected "no data" case.

**Prevention:**
Copy the null-guard pattern from `Form1.cs` into every browse handler. Return an empty children array (not an error) when `pObj == null` or `VarnameList == null`. Log the unresolvable RelId for diagnostics. This degrades gracefully: the node appears expandable and shows "No tags available" when opened.

**Detection:**
- HTTP 500 on `/browseTypeInfo?relId=...` for specific datablocks (e.g. system DBs like `DB65535`).
- Driver logs show `NullReferenceException` or `IndexOutOfRangeException` in the browse handler.

**Phase to address:** Backend browse endpoint — before any test against a real PLC.

---

### Pitfall 8: realtimeData Live-Value Lookup Has No Index on Symbolic Address

**What goes wrong:**
The TagTreeBrowser live-value column must look up current values from `realtimeData` by matching the PLC symbolic address (e.g. `"DB1.Motor.Speed"`) against `protocolSourceObjectAddress`. Without an index on this field, each lookup is a full collection scan. json-scada installations can have thousands of configured tags. Fetching live values for all visible tree nodes — either N single-document queries or one `$in` query — is slow without indexing.

**Why it happens:**
json-scada's existing `realtimeData` indexes are `{ _id }` (default) and driver-created operational indexes. The `{ createdAt: -1 }` index added in v1.3 is on `s7plusAlarmEvents`, not `realtimeData`. An index on `protocolSourceObjectAddress` is not created by the default json-scada schema.

**Consequences:**
- Live-value column refreshes slowly (>500 ms) as the configured tag count grows.
- Full collection scans compete with the driver's concurrent polling writes, increasing write latency.

**Prevention:**
Create a `{ protocolSourceObjectAddress: 1, protocolConnectionNumber: 1 }` compound index on `realtimeData` at backend startup, scoped to the connection number to avoid false matches across different protocol connections that may reuse the same address string format. Use the same `createIndex` idempotent pattern established in Phase 10.

**Detection:**
- `explain()` on a `realtimeData` find by `protocolSourceObjectAddress` shows `COLLSCAN`.
- Live-value lookup latency grows as more tags are added to json-scada.

**Phase to address:** Backend browse endpoint phase — create the index at server startup alongside the browse collection initialisation.

---

### Pitfall 9: Vue Router Navigation Guard Redirects the TagTreeBrowser Window and Loses the db Param

**What goes wrong:**
AdminUI's global navigation guard checks authentication before each route. When the TagTreeBrowser is opened via `window.open`, the new Vue app instance initialises with an empty Pinia store. The guard evaluates `authStore.isLoggedIn` as `false` (store is not yet hydrated) and redirects to `/login`. After login, Vue Router navigates back — but the original `?db=DB1` query parameter passed from the alarm viewer is absent from the post-login redirect, so the TagTreeBrowser opens with no datablock selected.

**Why it happens:**
Pinia stores are per-Vue-app-instance. A new tab starts with an empty store, even if `localStorage` contains a valid token. Unless the guard explicitly reads `localStorage` before evaluating, it will always redirect the first load.

**Consequences:**
- TagTreeBrowser always redirects to login or shows a blank page on first open.
- After login in the new window, the `db` param is gone — the operator must re-navigate manually.

**Prevention:**
The navigation guard must hydrate auth state from `localStorage` before evaluating `isLoggedIn`. The TagTreeBrowser route should preserve the `db` param through the redirect: encode it as a `redirect` param in the `/login` URL (`/login?redirect=/tagbrowser%3Fdb%3DDB1`), and restore it in the login success handler.

**Detection:**
- New window lands on the login page despite the main window being logged in.
- After login in the new window, the `db` query param is absent.

**Phase to address:** New-window navigation phase — before the `window.open` integration is wired up.

---

### Pitfall 10: Async Race in Tree Expansion — Rapid Expand-Collapse-Expand Produces Duplicate Children

**What goes wrong:**
A user expands a datablock node, immediately collapses it, then re-expands before the first HTTP browse request completes. `load-children` is called a second time for a node whose children are still loading. Both HTTP responses arrive and both attempt to populate the same node's children list. Vuetify `v-treeview` does not deduplicate children added via `load-children`; the node ends up with double the expected child count.

**Why it happens:**
Vuetify does not cancel or debounce `load-children` calls. The component has no built-in guard against concurrent load invocations for the same node.

**Consequences:**
- Tree node shows double or triple the expected child count after rapid expand-collapse-expand.
- Duplicate entries confuse operators and create ambiguity about which entry corresponds to which PLC tag.

**Prevention:**
Maintain a `loadingRelIds` Set alongside `loadedRelIds`. When `load-children` fires: if the RelId is already in `loadingRelIds`, return immediately (or return the cached promise). Set the entry in `loadingRelIds` at start; remove it in `finally`; add to `loadedRelIds` on success. This deduplicates in-flight requests regardless of trigger.

**Detection:**
- Tree node shows duplicate child names after rapid expand-collapse-expand.
- Browser DevTools network tab shows two concurrent requests to the same `/browseTypeInfo?relId=...` URL.

**Phase to address:** TagTreeBrowser Vue component — implement alongside the `loadedRelIds` guard from Pitfall 3.

---

## Minor Pitfalls

### Pitfall 11: window.open URL Construction — DB Name With Special Characters

**What goes wrong:**
PLC datablock names may contain spaces, slashes, or parentheses (e.g. `"Alarm DB"`, `"Line/1 (Main)"`). If the S7AlarmsViewer constructs the `window.open` URL without `encodeURIComponent`, the URL is malformed and `route.query.db` in the TagTreeBrowser receives a truncated or incorrect string.

**Prevention:**
Always use `encodeURIComponent(dbName)` when building the URL in `S7PlusAlarmsViewerPage.vue`. Vue Router's query parsing decodes percent-encoding automatically.

**Phase to address:** S7AlarmsViewer integration phase — trivial one-liner, easy to overlook.

---

### Pitfall 12: getTypeInfoByRelId Is Synchronous and Can Block the Browse Connection Thread

**What goes wrong:**
The GUIBrowser reference implementation (Form1.cs line 133) calls `getTypeInfoByRelId` directly on the UI thread. In the json-scada C# driver, if `getTypeInfoByRelId` is called from a thread shared with other work, the 5 s `m_ReadTimeout` wait blocks that thread for the full timeout on a non-responsive PLC.

**Prevention:**
Wrap all browse calls in `Task.Run(...)` with a `CancellationToken` tied to the HTTP request lifetime. This ensures that if the HTTP client disconnects, the browse thread is also cancelled. Use the dedicated browse connection (Pitfall 1), not any shared thread.

**Phase to address:** Backend browse endpoint phase — alongside the dedicated connection setup.

---

### Pitfall 13: New Browse Collection Name Must Be Defined as a Constant, Not an Inline String

**What goes wrong:**
If the collection name for cached browse results (e.g. `s7plusDatablockCache`) is inlined as a string literal in multiple files, a typo in one location silently creates a second collection in MongoDB and queries return empty results. The existing codebase uses named constants for all collection names in `Program.cs` (e.g. `RealtimeDataCollectionName`, `AlarmEventsCollectionName`).

**Prevention:**
Add `public static string DatablockCacheCollectionName = "s7plusDatablockCache";` to `Program.cs` alongside the existing constants. Reference only this constant everywhere. Create the collection with a `{ relId: 1 }` unique index at startup to prevent duplicate type-info documents.

**Phase to address:** Backend browse endpoint phase — define the constant before writing any collection access code.

---

### Pitfall 14: DatablockBrowser Startup GetListOfDatablocks Timeout on Large PLC

**What goes wrong:**
`GetListOfDatablocks` sends multiple messages to the PLC and waits for all datablock metadata to be returned. Vendor documentation for commercial S7 drivers reports that PLCs with a large number of datablocks can cause this browse to time out at the default timeout value — only the datablocks reported before the timeout are returned. The default `m_ReadTimeout` is 5000 ms. If the PLC has 300+ datablocks, the list may be silently truncated.

**Prevention:**
The `S7CP_connection.timeoutMs` config field (defaulting to 5000.0 in `Common.cs` line 67) is already passed into `Connect`. Verify that the `GetListOfDatablocks` call also uses this configurable timeout — and document in the milestone that PLC configurations with many DBs may need `timeoutMs` increased to 10000 or higher in the `protocolConnections` MongoDB document.

**Detection:**
- DatablockBrowser shows fewer datablocks than TIA Portal's project navigator.
- Driver log shows no error but datablock count is lower than expected.
- Increasing `timeoutMs` in the connection config adds missing datablocks to the list.

**Phase to address:** Backend startup browse phase — document the timeout configuration requirement.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Backend browse endpoint (C# driver side) | Pitfall 1: PDU interleaving on shared connection | Open dedicated third `S7CommPlusConnection` for all browse calls |
| Backend browse endpoint (C# driver side) | Pitfall 14: GetListOfDatablocks timeout on large PLC | Use configurable `timeoutMs`; document that >300 DB PLCs need 10 s+ |
| Backend browse endpoint (C# driver side) | Pitfall 12: getTypeInfoByRelId blocks thread | Wrap in `Task.Run` with `CancellationToken` |
| Backend browse endpoint (Node.js side) | Pitfall 2: blocking browse hangs Node event loop | Driver caches to MongoDB; Node reads cached data only |
| Backend browse endpoint (Node.js side) | Pitfall 7: null PObject crashes JSON serialisation | Copy null-guard from Form1.cs; return empty array, not HTTP 500 |
| Backend browse endpoint (Node.js side) | Pitfall 8: realtimeData lookup is a full collection scan | Create `{ protocolSourceObjectAddress:1, protocolConnectionNumber:1 }` compound index at startup |
| Backend browse endpoint (Node.js side) | Pitfall 13: collection name string collision | Define constant `s7plusDatablockCache`; never inline the string |
| DatablockBrowser Vue page | Pitfall 4: v-treeview slow render on large flat lists | Use `v-list` with `v-virtual-scroll` for flat datablock list |
| TagTreeBrowser Vue page | Pitfall 5: duplicate node names cause infinite loop | Set `:item-value` to unique relId-based key, never display name |
| TagTreeBrowser Vue page | Pitfall 3: load-children fires on select not only expand | Gate with `loadedRelIds` Set before issuing HTTP request |
| TagTreeBrowser Vue page | Pitfall 10: rapid expand-collapse race | Maintain `loadingRelIds` Set; deduplicate in-flight requests |
| TagTreeBrowser Vue page | Pitfall 4: deep nesting render bugs | Test with actual PLC FB depth; limit visible levels if >8 deep; consider pagination of large arrays |
| New-window navigation | Pitfall 6: JWT not available in new tab | Verify auth token is in `localStorage`, not `sessionStorage` |
| New-window navigation | Pitfall 9: navigation guard redirects and loses db param | Hydrate auth store from `localStorage` before guard evaluation; encode `db` param through redirect |
| New-window navigation | Pitfall 11: special chars in DB name URL | `encodeURIComponent(dbName)` in the `window.open` call |

---

## Sources

- **S7CommPlusDriver codebase (thomas-v2/S7CommPlusDriver):**
  - `S7CommPlusConnection.cs` — `m_ReceivedPDUs` Queue and `m_Mutex` (PDU queue per connection, lines 35–36); `WaitForNewS7plusReceived` 2 ms sleep-poll loop (line 116); `m_ReadTimeout = 5000` (line 48); `checkResponseWithIntegrity` sequence-number validation (line 360)
  - `Browser.cs` — `GetVarInfoList`, `AddBlockNode`, `BuildFlatList`, `AddFlatSubnodes`; TODO comment on ambiguous StructArray access-id (line 85)
  - `ExploreRequest.cs` — protocol message fields; `ExploreChildsRecursive`, `AddressList`
  - `S7CommPlusGUIBrowser/Form1.cs` — reference lazy expand: null-guard (line 136), synchronous `getTypeInfoByRelId` on UI thread (line 133), `GetListOfDatablocks` call pattern (line 61)

- **json-scada S7CommPlusClient codebase:**
  - `AlarmThread.cs` — separate `alarmConn` pattern; `alarmConn.Connect` on dedicated connection
  - `Program.cs` — collection name constants; `DataBufferLimit`; `BulkWriteLimit`
  - `Common.cs` — `S7CP_connection.timeoutMs` defaulting to 5000.0 (line 67)

- **json-scada server_realtime_auth:**
  - `index.js` — Express/Node.js architecture; `COLL_REALTIME = 'realtimeData'`; `createIndex` idempotent pattern established in Phase 10

- **Vuetify GitHub Issues (all confirmed as affecting Vuetify 3):**
  - [#19919 — VTreeview slow performance on expand (3.6.8, May 2024)](https://github.com/vuetifyjs/vuetify/issues/19919)
  - [#20389 — duplicate node/parent names cause infinite loop (3.7.0, August 2024)](https://github.com/vuetifyjs/vuetify/issues/20389)
  - [#21387 — deep nesting leads to rendering/hover problems (3.8.4, May 2025)](https://github.com/vuetifyjs/vuetify/issues/21387)
  - [#5543 — loadChildren fires on select not only expand](https://github.com/vuetifyjs/vuetify/issues/5543)
  - [#8791 — async children selection-type=independent incompatibility](https://github.com/vuetifyjs/vuetify/issues/8791)

- **Pinia cross-tab isolation:** Per-instance stores confirmed; `pinia-shared-state` community library documents the isolation as expected behaviour. `localStorage` is origin-partitioned and shared across same-origin windows; `sessionStorage` is tab-partitioned.

- **Browser sessionStorage vs localStorage partitioning:** MDN Web API documentation — `sessionStorage` is isolated per top-level browsing context; `window.open` creates a new top-level browsing context with no inherited `sessionStorage`.

- **S7 driver browse timeout:** devicewise.com Siemens S7 driver troubleshooting documentation — confirms multi-message browse sequence for large DB counts; recommends minimum 10 s timeout when PLC has many datablocks.

- **PROJECT.md KEY DECISIONS:** "Separate S7CommPlusConnection for alarm thread" rationale (v1.2) — directly applicable to browse connection isolation in v1.4.

---
*Pitfalls research for: lazy PLC tag tree browsing — DatablockBrowser + TagTreeBrowser (v1.4)*
*Researched: 2026-03-26*

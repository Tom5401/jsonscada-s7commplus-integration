# Pitfalls Research

**Domain:** Lazy tree loading, real value display, value writing, and non-datablock tag support added to existing S7CommPlus / Vue 3 / MongoDB TagTreeBrowser (v1.5 overhaul)
**Researched:** 2026-03-30
**Confidence:** HIGH — all pitfalls grounded in direct codebase inspection of TagTreeBrowserPage.vue, server_realtime_auth/index.js, TagsCreation.cs, TagMapping.cs, S7CommPlusGUIBrowser/Form1.cs, and verified against Vuetify 3 issue tracker

---

## Critical Pitfalls

### Pitfall 1: load-children Only Fires When children Is Exactly an Empty Array — null or undefined Means "Leaf"

**What goes wrong:**
Vuetify 3 `v-treeview` invokes the `load-children` callback only when a node is expanded and its children property is an empty array `[]`. If children is `null`, `undefined`, or the property is absent, the node is treated as a leaf and no expand toggle is shown. The current v1.4 `buildTree()` sets `children: isLast ? undefined : []` for non-leaf nodes. When migrating to lazy loading, any node whose backend-assigned children array starts as `undefined` will never get an expand toggle — the user cannot drill into it.

**Why it happens:**
Developers migrating from "build full tree upfront" to "lazy load children" naturally think of "no children yet loaded" as equivalent to `undefined`. Vuetify draws a hard line: `[]` signals "branch node, children not loaded yet"; `undefined` signals "leaf node, no expand toggle."

**Consequences:**
- Intermediate folders (UDT struct levels) appear as leaves with no expand arrow.
- The user cannot navigate into sub-levels despite the PLC having nested data there.
- The bug is invisible at the first level (which already works) and only surfaces at depth 2+.

**How to avoid:**
When building the initial skeleton items for a lazy tree, set `children: []` (empty array) for every folder/branch node — even if no children have been fetched yet. Never use `undefined` or `null` for branch children. Set `children: undefined` only for confirmed leaf nodes (tags with a protocolSourceObjectAddress).

**Warning signs:**
- Depth-2 nodes (struct members inside a datablock) show no expand arrow after the first lazy expand.
- `item.children === undefined` on a non-leaf item in a breakpoint inspection.

**Phase to address:** Lazy loading backend + frontend phase — the node shape must be specified before writing any tree-construction code.

---

### Pitfall 2: The Existing listS7PlusTagsForDb Query Cannot Be Used As-Is for Lazy Children

**What goes wrong:**
The current `listS7PlusTagsForDb` endpoint does:
```js
{ protocolSourceBrowsePath: { $regex: new RegExp('^' + escapedDbName + '(\\.|$)') } }
```
This returns ALL tags whose `protocolSourceBrowsePath` starts with the DB name — i.e. every tag in the entire datablock at once. Calling this same query per-level (e.g. for a sub-struct node) would return the entire subtree, not just direct children. A lazy backend endpoint must return only the direct children of the requested path, requiring a different query: an exact match on `protocolSourceBrowsePath` equal to the requested path. Without this change, lazy loading returns hundreds of documents instead of a handful of direct children.

**Why it happens:**
The v1.4 endpoint was designed for "load everything at once and buildTree() client-side." Reusing it for lazy loading without changing the query logic would make each level-expand as expensive as the original full-tree load.

**Consequences:**
- Memory and performance remain identical to the pre-overhaul full load — the overhaul achieves nothing.
- Frontend receives large payloads per expand click; `buildTree()` still runs over the full set.

**How to avoid:**
Add a new backend endpoint (e.g. `listS7PlusTagChildren`) that accepts a `parentPath` string parameter and queries:
```js
{ protocolSourceConnectionNumber: connectionNumber,
  protocolSourceBrowsePath: parentPath }
```
This is an exact equality match (not a regex), which uses an index and returns only direct children. The lazy load for a DB root level passes `parentPath = dbName`; expanding `DB.Struct` passes `parentPath = "DB.Struct"`.

**Warning signs:**
- Network tab shows large responses (hundreds of docs) on a single node expand.
- Expand of any level takes similar time to the original full-tree load.

**Phase to address:** Backend lazy-children endpoint — before writing any Vue `load-children` code.

---

### Pitfall 3: protocolSourceBrowsePath Is the Parent Path, Not the Node's Own Path

**What goes wrong:**
`protocolSourceBrowsePath = ov.path = ExtractPathFromName(name)` which strips the last dot-segment: `"DB1.Struct.Tag"` becomes `"DB1.Struct"`. This is documented as a known bug in PROJECT.md: "protocolSourceBrowsePath is parent path (not leaf path)." For a lazy-children query, the correct filter for "give me direct children of DB1.Struct" is `{ protocolSourceBrowsePath: "DB1.Struct" }`. But for top-level DB tags (tags directly under the DB root, e.g. `"DB1.Tag"`), the path is `"DB1"` — which is the DB name itself, not the full path to the node. The confusion is that the same field plays the role of "parent path" for ALL tag types, which is correct for the exact-match query but not intuitive.

**Why it happens:**
`ExtractPathFromName` always strips the last segment. This is consistent but non-obvious. A developer might assume `protocolSourceBrowsePath` is the node's own path and build the filter as `{ protocolSourceBrowsePath: "DB1.Struct.Tag" }` (the full tag path), which returns zero results.

**Consequences:**
- Lazy-children queries return empty results for all levels.
- The expand toggle shows a spinner that resolves to zero children.

**How to avoid:**
Document and enforce: the lazy-children endpoint's `parentPath` parameter maps directly to `protocolSourceBrowsePath` (the parent, not the leaf). When the user expands node with id `"DB1.Struct"`, the query is `{ protocolSourceBrowsePath: "DB1.Struct" }` — NOT `"DB1.Struct.*"` or `"DB1.Struct.something"`. Verify with a direct MongoDB query before coding the endpoint.

**Warning signs:**
- Querying `realtimeData` with `protocolSourceBrowsePath: "DB1.Struct.Tag"` returns 0 results.
- Querying with `protocolSourceBrowsePath: "DB1.Struct"` returns the expected leaf documents.

**Phase to address:** Backend lazy-children endpoint — verify this invariant with a direct MongoDB query on actual data before writing the endpoint.

---

### Pitfall 4: value Is Numeric (0.0 / 1.0) — Displaying TRUE/FALSE Requires stateTextTrue/False

**What goes wrong:**
The `realtimeData` collection stores all values as BSON double (`value: 0.0` or `value: 1.0` for digital tags). The TagTreeBrowser v1.4 displays `item.value` directly in a `<code>` element, showing `0` or `1` instead of `FALSE`/`TRUE`. To match tabular view behavior, the display must check `doc.type === 'digital'` and use `doc.stateTextTrue` / `doc.stateTextFalse`. For S7CommPlusClient-created tags, these fields are always populated: `TagsCreation.cs` line 279 sets `stateTextFalse = "FALSE"` and `stateTextTrue = "TRUE"` for digital types. However, tags from other protocols may have empty `stateTextTrue`/`stateTextFalse` — a fallback to `"TRUE"/"FALSE"` is needed.

**Why it happens:**
`doc.value` is the path of least resistance. Developers glance at the displayed value, see `1` instead of `TRUE`, and assume it "just needs a format function" without considering the `stateText` fields.

**Consequences:**
- Digital tags show `0` / `1` instead of `FALSE` / `TRUE` — requirement is explicitly not met.
- For REAL/INT tags, numeric display is correct, so the bug is only visible on Bool types.

**How to avoid:**
In the leaf `value` display:
```js
displayValue(doc) {
  if (doc.type === 'digital') {
    return doc.value !== 0
      ? (doc.stateTextTrue || 'TRUE')
      : (doc.stateTextFalse || 'FALSE')
  }
  return doc.value
}
```
This mirrors how `server_realtime_auth/index.js` (line 2000) converts digital values to boolean in the OPC read response.

**Warning signs:**
- Leaf value column shows `0` or `1` for Bool tags.
- `doc.type === 'digital'` is true but value display bypasses stateText lookup.

**Phase to address:** Real value display phase — the display format function must be in place before any value-refresh code is considered done.

---

### Pitfall 5: Value Writing Must Go Through the Existing opcApi WriteRequest — Not a New Endpoint

**What goes wrong:**
The requirement is "open the existing tabular view push-value window." The tabular viewer (`tabular.html`) uses `WebSAGE.g_win_cmd = window.open(...)` to open a legacy command dialog — a separate HTML page (`cmd.html` or equivalent). This is not a Vue component and cannot be directly imported. Developers attempting to replicate the write logic in TagTreeBrowser from scratch will re-implement the OPC UA WriteRequest pattern (ServiceCode 671) incorrectly, miss the `canSendCommands` authorization check in `server_realtime_auth`, or bypass the `DoInsertCommandAsSOE` SOE log entry — both of which are critical for production SCADA audit trails.

**Why it happens:**
The push-value path in `server_realtime_auth/index.js` (line 754: `case opc.ServiceCode.WriteRequest`) requires: (1) the tag's numeric `_id` or string `tag` as `NodeId.Id`; (2) a `Value` object with `Type` (double = 5 or string = 12) and `Body`; (3) JWT auth in the request header. This is the standard json-scada command interface but it is not exposed as a simple REST endpoint — it is wrapped in the OPC UA message envelope. Developers who do not study `server_realtime_auth/index.js` fully will miss this and write a simpler but non-functional endpoint.

**Consequences:**
- New write endpoint bypasses user authorization check (`canSendCommands`).
- SOE log entries (`COLL_SOE`) are not created — no audit trail.
- UserActions queue is not updated — no record in the `userActions` collection.
- The write silently fails if the tag's `_id` numeric is not found in `realtimeData`.

**How to avoid:**
Reuse the existing OPC WriteRequest infrastructure: construct the proper `{ ServiceId: 671, Body: { NodesToWrite: [...] } }` envelope from TagTreeBrowser and POST it to `/Invoke/auth`. This is the same call the tabular viewer makes. Alternatively, open the existing legacy tabular push-value window (if it exists as a standalone page) using `window.open` with the tag address encoded in the URL — the same approach `S7PlusAlarmsViewerPage.vue` uses to open TagTreeBrowser. Before building any custom dialog, check whether the existing `cmd.html` or equivalent can be opened with just a tag name parameter.

**Warning signs:**
- Write action succeeds from UI but `rtDatabase` shows no updated `value` field.
- `userActions` collection has no entry after a write.
- `soeData` collection has no entry after a write.
- `server_realtime_auth` logs show no "Command" log line after the write.

**Phase to address:** Value writing phase — study `server_realtime_auth/index.js` lines 1143–1402 before writing a single line of write-path code.

---

### Pitfall 6: Non-Datablock Tags (Inputs/Outputs/Merker) Have No protocolSourceBrowsePath Parent Path — They Are Root-Level

**What goes wrong:**
For datablock tags, `varInfo.Name` is `"DB1.Struct.Tag"` and `ExtractPathFromName` yields `"DB1.Struct"` as the parent path. For non-datablock tags (Inputs, Outputs, Merker), the S7CommPlusDriver `Browser.cs` assigns names without a DB prefix. Based on `GUIBrowser/Form1.cs` lines 77–93, these areas correspond to RelIds `0x90010000` (Inputs), `0x90020000` (Outputs), `0x90030000` (Merker). Their `varInfo.Name` values do not start with a `DB` prefix. The `ExtractPathFromName` for a leaf like `"I0.0"` returns `""` (empty string), meaning `protocolSourceBrowsePath = ""`. The `listS7PlusTagsForDb` query filters by `protocolSourceBrowsePath: { $regex: '^dbName(\\.|$)' }` — it will never return these tags because their browse path is `""`.

**Why it happens:**
The entire v1.4 backend and frontend assumes tags belong to a DB (filtering by `dbName` prefix on `protocolSourceBrowsePath`). Non-datablock tags break this assumption silently — they exist in `realtimeData` but are never returned by any existing endpoint.

**Consequences:**
- MArea/QArea/IArea tags are present in MongoDB but invisible in both DatablockBrowser and TagTreeBrowser.
- Any count of "all S7Plus tags" from the existing endpoint will be understated.
- The lazy-children endpoint for `parentPath = ""` would return ALL non-datablock tags in one response (potentially thousands of Merker bits).

**How to avoid:**
Treat the non-datablock areas as top-level virtual folders in DatablockBrowser: "Inputs", "Outputs", "Merker" (matching GUIBrowser line 77–93). For the backend, add a separate query path: when `parentPath` is one of these area names, query `realtimeData` where `protocolSourceBrowsePath = ""` and filter by the area prefix in `ungroupedDescription` (e.g. `ungroupedDescription` starts with `"I"` for Inputs). Alternatively, when the S7CommPlusClient stores these tags, populate `protocolSourceBrowsePath` with a dedicated area identifier (e.g. `"_IArea"`, `"_QArea"`, `"_MArea"`) to make them queryable by the same path-based endpoint.

**Warning signs:**
- S7CommPlusClient logs show "Browse returned N variables" but DatablockBrowser only accounts for DB tags.
- Direct MongoDB query: `db.realtimeData.find({ protocolSourceBrowsePath: "" })` returns documents.
- Tags with `protocolSourceObjectAddress` matching `I`, `Q`, or `M` addressing pattern are missing from the tree.

**Phase to address:** Non-datablock tag support phase — understand the address schema before writing any backend query, and decide whether to fix at storage time (driver) or at query time (endpoint).

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Reuse existing `listS7PlusTagsForDb` for lazy loading with client-side depth filtering | No new backend endpoint | Returns full subtree per expand — still loads all docs, no performance gain | Never — the whole point of lazy loading is fewer docs per request |
| Mutate `treeItems` directly on each refresh instead of diffing | Simple code | Collapses all expanded nodes on every 5 s refresh; disrupts user navigation | Never — expand state loss is immediately visible and disruptive |
| Display `doc.value` raw for all types | Zero code | Shows 0/1 for digital tags; explicit requirement says TRUE/FALSE | Never — stated requirement |
| Use `doc.tag` (the `connName;address` string) as the `item-value` key in v-treeview | Obvious choice | `doc.tag` contains semicolons and special chars; Vuetify may behave unexpectedly; breaks if `tag` is not unique across levels | Only if verified unique and Vuetify parses it without issues |
| Skip index creation for `protocolSourceBrowsePath` field | No migration step | Full COLLSCAN on every lazy expand request; degrades with tag count | Acceptable in PoC on <10k tags, but document the gap |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Vuetify v-treeview `load-children` | Return the loaded children from the callback | Mutate the item's `children` array in-place; `load-children` receives the item, not a return value |
| `load-children` + `v-model:opened` | Programmatically setting `openedNodes` after a lazy load resets the opened state | Do not touch `openedNodes` inside `load-children`; let Vuetify manage open state; only write `openedNodes` for programmatic auto-expand at mount time |
| Vuetify 3.10 v-treeview `load-children` | Assume the callback is called once per node per session | The callback is called once per "children is empty array" state; if children are cleared (e.g. on refresh), the callback fires again — guard with a `loadedSet` |
| opcApi WriteRequest | Send the tag's `protocolSourceObjectAddress` as `NodeId.Id` | Send the tag's `tag` field (string) or numeric `_id` as `NodeId.Id`; the command lookup is by `{ _id: parseInt(id) }` or `{ tag: id }`, not by address |
| 5 s auto-refresh of leaf values | Re-fetch the full subtree and rebuild from scratch | Call `patchLeafValues()` to update only `item.value` in-place on already-loaded leaf nodes; preserve expanded state |
| MongoDB `protocolSourceBrowsePath` query | Use `$regex` prefix match for the children query | Use exact equality match for direct children: `{ protocolSourceBrowsePath: parentPath }`; regex adds no value and prevents index use when parentPath contains special chars |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Full-subtree query per expand (reusing existing endpoint) | Expand of a large DB takes 500 ms+ and returns hundreds of docs | New endpoint with exact `protocolSourceBrowsePath` equality match | Immediately on any DB with >50 tags |
| No index on `protocolSourceBrowsePath` | Lazy-children endpoint is slow; COLLSCAN grows with tag count | Create `{ protocolSourceConnectionNumber: 1, protocolSourceBrowsePath: 1 }` compound index at server startup | Noticeable at >5 k tags; problematic at >50 k |
| Value refresh re-fetches all tags for the DB (existing `refreshValues`) | 5 s refresh calls `listS7PlusTagsForDb` which returns all docs; grows with DB size | Scope the value refresh to only fetch tags for visible/expanded paths | At >500 tags in one DB |
| Rendering all loaded children at once in v-treeview | Expand of a struct with 500 array members freezes the UI for 1–2 s | Cap array member display at 100 with "show more" pagination | At >100 direct children |
| `touchExpandedLeafTags` sends all visible leaf addresses every 5 s | Bulk upsert on `activeTagRequests` grows in size as more nodes are expanded | Touch only changed (newly expanded) leaf sets, not all visible leaves on every timer tick | At >50 expanded leaves |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| New lazy-children endpoint without `[authJwt.isAdmin]` guard | Unauthenticated access to full realtimeData tag list for any connection | All new endpoints in `server_realtime_auth` must include `[authJwt.isAdmin]` as middleware, matching the v1.4 pattern for `listS7PlusTagsForDb` |
| Value write endpoint that skips `canSendCommands(req)` check | Any authenticated user can write PLC values regardless of role | Use the existing `opc.ServiceCode.WriteRequest` path in `opcApi`, which already enforces `canSendCommands` at line 1147 |
| Exposing `_id` (numeric point key) in tree node data without need | Enables direct numeric ID enumeration of realtimeData points | Only expose `tag`, `protocolSourceObjectAddress`, and display fields in the lazy-children response; omit `_id` unless needed for write |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Expand state collapses on 5 s value refresh | Operator expands 3 levels, waits 5 s, tree resets to root — must re-expand manually | Patch leaf `value` fields in-place using `patchLeafValues()`; never reassign `treeItems.value` or `openedNodes` during refresh |
| No loading indicator per node during lazy expand | User clicks expand, nothing happens for 300 ms — clicks again, fires two requests | Use a per-node `loading: true` flag shown as a spinner in the node prepend slot |
| Write dialog opened from a node where no command tag exists | Dialog opens but write fails silently — no error shown | Check that the leaf node's `commandOfSupervised` ID is non-zero before showing a "Write" button; hide the button for read-only tags |
| Non-datablock areas not listed in DatablockBrowser | Operator cannot find I/Q/M tags at all — silent omission looks like "they're not configured" | Add virtual "Inputs", "Outputs", "Merker" rows in DatablockBrowser with the same "Browse Tags" pattern as DB rows |
| Value refresh continues after tag tree tab is closed | `setInterval` in `onMounted` still fires; browser tab memory and network requests persist indefinitely if `onUnmounted` is not reached (e.g. route reuse without full unmount) | Always clear the interval in `onUnmounted`; the v1.4 code already does this correctly — do not remove it during refactor |

---

## "Looks Done But Isn't" Checklist

- [ ] **Lazy loading at every level:** Verify that expanding depth-3 nodes (struct inside struct) also triggers `load-children` — not just depth-1. Confirm `children: []` is set on all intermediate folder nodes returned by the lazy endpoint.
- [ ] **TRUE/FALSE display:** Check that `type === 'digital'` tags show `FALSE` / `TRUE` (or configured stateText), not `0` / `1`. Verify analog tags still show numeric values (no regression).
- [ ] **Value refresh scoped to visible leaves:** After expanding 2 levels, confirm the 5 s refresh only touches the `activeTagRequests` for leaves in the expanded sub-paths — not all leaves from the original full-load.
- [ ] **Write path authorization:** Trigger a write from TagTreeBrowser while logged in as a non-admin role with `sendCommands: false`. Verify the write is rejected with an appropriate error, not silently dropped.
- [ ] **Non-datablock tags visible:** After enabling non-datablock support, query `db.realtimeData.find({ protocolSourceConnectionNumber: X, protocolSourceBrowsePath: "" })` and confirm the count matches what DatablockBrowser shows for Inputs/Outputs/Merker.
- [ ] **Expand state preserved through refresh:** Expand 3 levels, wait 10 s (two refresh cycles), confirm all 3 levels remain expanded.
- [ ] **load-children deduplication:** Rapidly expand-collapse-expand a node. Confirm the child count remains correct (no duplicates).
- [ ] **connectionNumber vs connectionId consistency:** Any new endpoint that accepts `connectionNumber` in the query string must use the same field name as the existing endpoints — `connectionNumber` (integer), NOT `connectionId` (known naming inconsistency in PROJECT.md technical debt).

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| load-children fires, children never appear (children was undefined not []) | LOW | Change branch node shape to always set `children: []`; no schema migration needed |
| Existing listS7PlusTagsForDb used for lazy load — still loads full tree | LOW | Add new endpoint with exact-match query; retire old reuse attempt |
| protocolSourceBrowsePath misunderstood — query returns zero results | LOW | Add MongoDB explain() in dev console; pivot to correct exact-match query immediately |
| Value refresh resets expand state | MEDIUM | Restore patchLeafValues() approach; regression introduced in the lazy-load refactor |
| Write bypasses auth check | HIGH | Remove custom endpoint; route through opcApi WriteRequest; audit any writes made during the gap |
| Non-datablock tags with protocolSourceBrowsePath="" silently missing | MEDIUM | Either update driver to write area-specific browse path, or add area-aware query in endpoint; no data loss, just visibility gap |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Pitfall 1: children must be [] not undefined | Lazy loading phase (backend + Vue) | Expand depth-3 node; confirm expand toggle appears |
| Pitfall 2: full-subtree query reuse | Lazy loading backend endpoint phase | Count docs in network response for a single expand: must be direct children only |
| Pitfall 3: protocolSourceBrowsePath is parent path | Lazy loading backend endpoint phase | Direct MongoDB query verification before coding |
| Pitfall 4: value display for digital tags | Real value display phase | Confirm Bool tag shows FALSE/TRUE, not 0/1 |
| Pitfall 5: write must go through opcApi | Value writing phase | Check userActions collection after write; check SOE log |
| Pitfall 6: non-datablock tags have empty browse path | Non-datablock tag support phase | Direct MongoDB count on `protocolSourceBrowsePath: ""` before and after |

---

## Sources

- **TagTreeBrowserPage.vue (v1.4):** Direct inspection — `buildTree()` uses `children: isLast ? undefined : []` (line 63); `refreshValues()` calls `listS7PlusTagsForDb` for full-DB reload; `patchLeafValues()` correct in-place pattern
- **server_realtime_auth/index.js:** `listS7PlusTagsForDb` endpoint (line 488–513) — regex prefix query; `opcApi WriteRequest` handler (line 754, 1143–1402) — `canSendCommands`, SOE insert, `UserActionsQueue`; digital value conversion pattern (line 1995–2003)
- **TagsCreation.cs (S7CommPlusClient):** `protocolSourceBrowsePath = ov.path` (line 192, 265); `ungroupedDescription = ov.display_name` (line 202, 275); `stateTextFalse = "FALSE"`, `stateTextTrue = "TRUE"` for digital (line 279)
- **TagMapping.cs (S7CommPlusClient):** `ExtractPathFromName()` (line 135) — strips last dot-segment from varInfo.Name
- **S7CommPlusGUIBrowser/Form1.cs:** Inputs (`0x90010000`), Outputs (`0x90020000`), Merker (`0x90030000`) as top-level nodes (lines 77–93); lazy expand pattern with `getTypeInfoByRelId` (line 133); null guard (line 136)
- **PROJECT.md:** `protocolSourceBrowsePath is the parent path (not leaf path)` — explicitly documented known bug; `connectionId vs connectionNumber naming inconsistency` — technical debt
- **Vuetify 3 issue tracker (verified against Vuetify 3.10 in use):**
  - [#20450 — load-children not working after upgrade (3.7.1)](https://github.com/vuetifyjs/vuetify/issues/20450) — loadChildren must be async
  - [#19453 — TypeError props.loadChildren(...) is undefined (3.5.11)](https://github.com/vuetifyjs/vuetify/issues/19453) — loadChildren must be async function
  - [#19919 — VTreeview slow performance on expand (3.6.8)](https://github.com/vuetifyjs/vuetify/issues/19919) — re-render on opened state change
  - [#19983 — empty children array treated as having children (3.6.8)](https://github.com/vuetifyjs/vuetify/issues/19983) — empty array shows expand arrow; workaround: `item-children` function returning null for empty arrays
  - [#10175 — load-children fires only once per empty array](https://github.com/vuetifyjs/vuetify/issues/10175) — must populate children in callback or it fires again on re-expand
  - [MongoDB $regex prefix query performance](https://www.mongodb.com/docs/manual/reference/operator/query/regex/) — anchored prefix regex uses index but exact equality is faster

---
*Pitfalls research for: v1.5 TagTreeBrowser overhaul — lazy tree loading, real value display, value writing, non-datablock tags*
*Researched: 2026-03-30*

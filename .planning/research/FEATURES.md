# Feature Research

**Domain:** TagTreeBrowser Overhaul — lazy loading, boolean display, value write, non-datablock tags (v1.5)
**Researched:** 2026-03-30
**Confidence:** HIGH — all findings verified from the live codebase (TagTreeBrowserPage.vue, TagsCreation.cs, TagMapping.cs, MongoCommands.cs, index.js); no external speculation.

---

## Context: What v1.4 Delivered (Baseline)

TagTreeBrowserPage.vue currently:
- Eagerly loads all tags for one datablock on page open via `listS7PlusTagsForDb` (returns full flat list)
- Builds the tree client-side by splitting `ungroupedDescription` path on "."
- Displays `item.value` (numeric double) in the value column — shows `0`/`1` for booleans, not `TRUE`/`FALSE`
- Refreshes ALL tags for the DB on a 5s timer, patches leaf values in-place
- Has no write capability
- Only handles tags where `protocolSourceBrowsePath` matches a DB name — MArea/QArea tags are invisible because they are never stored in `s7plusDatablocks`

The four v1.5 features fix these four specific gaps.

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features where the current v1.4 behavior is observably broken or incomplete. Operators notice immediately.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| TRUE/FALSE display for boolean tags | Operator tool; "0" and "1" are meaningless for Bool/BBool PLC tags. The tabular view already shows TRUE/FALSE. Having two views show different formats for the same value is confusing. | LOW | `valueString` field in the MongoDB document holds "True"/"False" (C# `bool.ToString()` capitalisation). The driver sets `stateTextTrue = "TRUE"` and `stateTextFalse = "FALSE"` on creation. The tree template uses `item.value` (double). Fix: `item.type === 'digital' ? (item.value ? 'TRUE' : 'FALSE') : item.value`. No backend change. |
| Lazy tree loading — do not load all 100k+ tags at once | Every major tag browser (TIA Portal, Ignition, OPC UA clients) defers loading until a node is expanded. Loading a 100k-tag DB eagerly would freeze the browser tab. | MEDIUM | Requires: (1) new backend endpoint returning only the direct children of a given path, (2) Vuetify 3 `load-children` prop on `v-treeview` — fires when an item whose `children` is an empty array `[]` is expanded; the callback fetches and pushes children into `item.children`; items with `children: undefined` are leaves (no expand arrow). Vuetify 3.10 `load-children` is confirmed working (GitHub issue #20450 closed as "works"). |
| Value refresh scoped to expanded/visible nodes | With lazy loading, most nodes are never expanded — there is no point refreshing their values. Refreshing all tags in a large DB wastes driver polling capacity. | LOW | The v1.4 `touchExpandedLeafTags()` mechanism already sends active-tag-requests only for expanded leaves. With lazy loading, the 5s refresh must fetch values only for the current open path's loaded descendants, not the whole DB. The refresh call should target the currently-loaded visible leaves, not re-call `listS7PlusTagsForDb` for the entire DB. |

### Differentiators (Competitive Advantage)

Features that go beyond fixing bugs and add genuine operator capability.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Write/push values from tree leaf nodes | Operators can test PLC I/O directly from the alarm-origin workflow — no context switch to tabular view. | MEDIUM | The write infrastructure already exists: `commandsQueue` collection, `S7CommPlusClient` `MongoCommands.cs` watches it via change stream and calls `srv.connection.WriteValues()`. The gap is the frontend. The tabular view's write path uses a legacy jQuery popup (`dlgcomando.html`) opened via `window.open` — it requires `window.opener.WebSAGE` context and is incompatible with the Vue SPA. A new minimal `v-dialog` is needed. The dialog needs: tag display name, current value, type (digital/analog/string), input field matched to type, confirm button. It should insert to `commandsQueue` via a new admin-guarded endpoint (consistent pattern with other s7plus endpoints). Each supervised tag has a paired command tag with `commandOfSupervised` pointing to the supervised tag's `_id`; the write target is the command tag's `protocolSourceObjectAddress` (same address as the supervised tag, `origin: "command"`). |
| Non-datablock tags (MArea, QArea) in DatablockBrowser | MArea (Merker/memory bits) and QArea (output area) tags are browsed by the driver and stored in `realtimeData` with `protocolSourceBrowsePath = "MArea"` or `"QArea"`. They don't appear in `s7plusDatablocks` (only actual DB objects are stored there), so operators cannot browse them via the current UI. | MEDIUM | Two sub-tasks: (1) Driver change: after `GetListOfDatablocks`, also upsert any non-DB area prefixes present in `realtimeData` for that connection into `s7plusDatablocks`. Non-DB areas have `db_number: -1` (sentinel) to distinguish them from numbered datablocks. (2) UI: DatablockBrowserPage shows them in the list; `db_number: -1` renders as "—" or a label like "Area". No changes to `listS7PlusTagsForDb` or TagTreeBrowserPage — the existing `protocolSourceBrowsePath` prefix regex already handles any prefix including "MArea", "QArea". Passing `dbName=MArea` works today if the entries exist. |

### Anti-Features (Commonly Requested, Often Problematic)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Reuse `dlgcomando.html` popup for tree writes | It already handles digital/analog/string input | Requires `window.opener.WebSAGE` global — tightly coupled to the tabular view's legacy context. Opening it from the Vue SPA tree requires injecting WebSAGE globals. Fragile and unmaintainable. | Build a minimal Vue `v-dialog` that inserts directly to `commandsQueue` via a backend endpoint. |
| Load all tags eagerly, then show the lazy UI | Simpler than a new endpoint — just reuse `listS7PlusTagsForDb` and hide behind a spinner | Defeats the "100k+ tag DB" goal in PROJECT.md. Loading everything just to build a tree that shows only the first level is wasteful. The 5s full-DB reload already causes brief freezes on large DBs. | New endpoint returning direct children of a given path only. One MongoDB query scoped to depth. |
| Auto-refresh all tags in DB on 5s timer (keep v1.4 behaviour) | Keeps all values current even for collapsed nodes | With lazy loading, most nodes are never expanded. Polling their values wastes driver capacity and generates MongoDB load for data nobody is viewing. | Scope the 5s refresh to `openedNodes` — only expanded leaves. Already done in v1.4 via `touchExpandedLeafTags()`; just ensure it works with the new dynamic node structure. |
| Real-time push (WebSocket/SSE) for live values | Lower latency than 5s polling | Significant infrastructure complexity for a PoC. AdminUI has no Vue-native WebSocket path for per-tag value subscriptions. | Retain 5s polling scoped to visible/expanded leaves. Acceptable for operator use. |
| Ack feedback in write dialog (wait for PLC confirmation) | Show the operator when the write was confirmed by the PLC | Requires polling `commandsQueue` until `delivered: true`, or a change stream subscription from the browser. Adds round-trip complexity. The tabular write path also has an ack feedback path but it is fragile (spinner edge case noted in PROJECT.md technical debt). | Show "command sent" on insert success. Match the minimal PoC contract. |

---

## Feature Dependencies

```
TRUE/FALSE boolean display
    └──depends on──> valueString field (already in MongoDB, driver already populates it)
    └──no new backend needed

Lazy tree loading
    └──requires──> New backend endpoint (listS7PlusTagChildren or new query param on existing)
    │                  └──returns direct children of a given path (one depth level)
    └──requires──> Vuetify 3.10 load-children prop on v-treeview
    └──enables──> Scoped value refresh (only expanded descendants of currently-loaded path)
    └──replaces──> Current: load ALL tags for DB eagerly, build tree client-side

Value refresh scoped to visible nodes
    └──depends on──> Lazy tree loading (defines which nodes are "loaded")
    └──reuses──> Existing touchExpandedLeafTags() mechanism (already in v1.4)

Write/push from tree leaf nodes
    └──requires──> New minimal Vue v-dialog component
    └──requires──> New backend endpoint to insert to commandsQueue
    └──depends on──> Supervised tag has commandOfSupervised != 0 (paired command tag)
    └──reuses──> Existing commandsQueue + MongoCommands.cs write path (no driver change)
    └──requires──> listS7PlusTagsForDb response includes _id field (needed to look up command tag)

Non-datablock tags (MArea/QArea)
    └──requires──> C# driver change: upsert non-DB area prefixes into s7plusDatablocks at startup
    └──reuses──> Existing listS7PlusDatablocks endpoint (no query change)
    └──reuses──> Existing listS7PlusTagsForDb endpoint (already handles any prefix)
    └──reuses──> Existing TagTreeBrowserPage (no change needed — accepts any dbName)
```

### Dependency Notes

- **Write requires the supervised tag's `_id`**: `listS7PlusTagsForDb` already returns full documents including `_id` and `commandOfSupervised`. When the user clicks write on a leaf, the dialog receives the full document. The backend endpoint for write looks up the command tag by `supervisedOfCommand == supervised._id AND protocolSourceConnectionNumber == conn`, gets its `protocolSourceObjectAddress`, and inserts to `commandsQueue`.

- **Lazy loading requires one new backend endpoint**: The simplest approach is a new query parameter `parentPath` on `listS7PlusTagsForDb` that returns only tags whose `ungroupedDescription` matches `^{parentPath}\.[^.]+$` (direct children one level below `parentPath`). This avoids a new endpoint and keeps the backend change minimal. The tree then loads level-by-level on demand.

- **MArea/QArea discovery**: The driver already iterates `varInfoList` from `Browse()` and stores every variable in `realtimeData`. Non-DB tags appear there with `protocolSourceBrowsePath = "MArea"` or `"QArea"`. The driver just needs to also upsert `{ db_name: "MArea", db_number: -1, connectionNumber: N }` into `s7plusDatablocks` when it encounters non-DB prefixes. This is a small addition to the startup logic.

- **TRUE/FALSE is completely independent**: No backend change, no new endpoint, no driver change. Single-line template change.

---

## MVP Definition

All four features are in scope for v1.5 as stated in PROJECT.md. Ordering below reflects implementation dependency (cheapest and most independent first).

### Launch With (v1.5)

- [ ] **TRUE/FALSE display** — Frontend-only, one-line template change, zero risk.
- [ ] **Lazy tree loading** — New `parentPath` query param on `listS7PlusTagsForDb` (or new endpoint) + Vuetify `load-children` integration + scoped value refresh.
- [ ] **Write/push from tree nodes** — New Vue `v-dialog` + new backend endpoint inserting to `commandsQueue`.
- [ ] **Non-datablock tags (MArea/QArea)** — Driver startup change + DatablockBrowserPage renders `db_number: -1` as "—".

### Add After Validation (v1.x)

- [ ] **Write ack feedback** — Poll `commandsQueue` for `delivered: true` and show confirmation in the dialog. Low priority — PoC contract is "command sent."
- [ ] **Refresh rate tuning** — Configurable 5s interval per session. Low PoC value.

### Future Consideration (v2+)

- [ ] **WebSocket/SSE for live values** — Real-time push. Requires server infrastructure changes.
- [ ] **Tag search within tree** — Filter to a specific tag by name. Useful for 100k+ DBs.
- [ ] **FB type name column** — Requires `GetTypeInformation` browse per DB; explicitly out of scope per PROJECT.md.

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| TRUE/FALSE boolean display | HIGH | LOW | P1 |
| Lazy tree loading | HIGH | MEDIUM | P1 |
| Scoped value refresh (lazy-aware) | MEDIUM | LOW (bundled with lazy) | P1 |
| Write/push from tree nodes | HIGH | MEDIUM | P1 |
| Non-datablock tags (MArea/QArea) | MEDIUM | MEDIUM | P1 |

**Priority key:**
- P1: Must have for v1.5 (all four features are in the milestone's stated scope per PROJECT.md)

---

## Detailed Implementation Notes

### TRUE/FALSE Display

The `listS7PlusTagsForDb` response includes both `value` (double: 0.0/1.0) and `valueString` (string: "True"/"False", C# capitalisation) for every document. Bool and BBool tags have `type: "digital"`.

TagTreeBrowserPage.vue currently shows `{{ item.value }}` in the template append slot. Fix:

```
{{ item.type === 'digital' ? (item.value ? 'TRUE' : 'FALSE') : item.value }}
```

This matches the tabular view's `stateTextTrue = "TRUE"` and `stateTextFalse = "FALSE"` stored at tag creation. Alternatively use `item.valueString.toUpperCase()` — but C# "True"/"False" would need normalisation. The value-based conditional is safer and always correct.

The same patch should apply to `patchLeafValues()` to keep the value consistent on 5s refresh — the refresh already replaces `node.value` from the doc. The display fix is purely in the template (computed display value from `node.value` and `node.type`), so it updates automatically when `node.value` is patched.

### Lazy Tree Loading

Current flow: `loadTree()` calls `listS7PlusTagsForDb` → gets all N tags → `buildTree()` builds the full hierarchy in memory → `treeItems.value = [root]`. For 100k tags, this is a large payload and a slow DOM render.

New flow: On page load, call a "get root level items" query (children of the DB root). On expand, call "get children of this path." Vuetify handles the rest via `load-children`.

Backend change — add optional `parentPath` query param to `listS7PlusTagsForDb`:
- If `parentPath` absent: existing behaviour (return all tags) — preserves backward compatibility.
- If `parentPath` present: return only docs where `ungroupedDescription` matches `^{parentPath}\.[^.]+$` — exactly one dot-segment below `parentPath`. The regex already uses escaped DB name; add `parentPath` as another filter.

Frontend change in `TagTreeBrowserPage.vue`:
- Initial load: fetch with `parentPath = dbName` — gets root-level children of the DB node.
- `buildInitialItems(docs, dbName)`: builds only the first level (items are either leaves or folder stubs with `children: []`).
- `load-children` async prop: given an item, fetch `listS7PlusTagsForDb?dbName=X&connectionNumber=N&parentPath={item.id}`. Push returned items as children.
- Items whose full path equals `ungroupedDescription` of a leaf tag → leaf (no `children`). Items that have children in the DB → stub with `children: []`.

Determining whether a folder stub has sub-children: the initial fetch returns docs at depth 1. If a segment at depth 1 appears as a prefix in the full flat list, it has children. However, with lazy loading we cannot know upfront. The simplest approach: if a path segment appears in docs as a `protocolSourceBrowsePath` prefix in `realtimeData`, it has children. The backend can return a `hasChildren` flag, or the frontend can assume all non-leaf items returned have children (items not matched as leaves by the initial query are folders — they get `children: []`).

A cleaner backend approach: add a second endpoint or query mode that returns "unique direct child names at path" — distinct next-segment values without full documents. This is lighter but requires more DB query complexity. For the PoC, fetching the direct-children documents (each is one leaf tag) and inferring folder structure from `ungroupedDescription` depth is acceptable.

Vuetify `load-children` pitfalls: the prop must be a function `(item) => Promise<void>` that mutates `item.children` in place and resolves. It fires only once per item (children cached). To re-trigger, reset `item.children = []`. The existing `v-model:opened` and `openedNodes` tracking for `touchExpandedLeafTags` continues to work — expand events still fire.

### Write/Push from Tree Leaf Nodes

The write infrastructure in `MongoCommands.cs` accepts `commandsQueue` inserts with:
```
{
  protocolSourceConnectionNumber: int,
  protocolSourceObjectAddress: string,   // the accessSequence address
  protocolSourceASDU: string,            // e.g. "Bool", "Int", "Real"
  value: double,
  valueString: string,
  timeTag: ISODate,
  delivered: false
}
```

The driver's change stream picks this up, resolves the address to an `ItemAddress` from `AddressCache`, converts the value via `ConvertCommandToPValue`, and calls `srv.connection.WriteValues()`.

Frontend:
- A "write" button (pencil icon) appears in the append slot of leaf nodes in TagTreeBrowserPage.
- Clicking it opens a `v-dialog` with the tag name, current value, a type-appropriate input (v-checkbox for digital, v-text-field for analog/string), and a "Write" confirm button.
- The dialog emits a write action that POSTs to a new endpoint `POST /Invoke/auth/writeS7PlusTag`.

Backend endpoint `writeS7PlusTag`:
- Admin-guarded (`[authJwt.isAdmin]`).
- Receives `{ connectionNumber, protocolSourceObjectAddress, asdu, value, valueString }`.
- Validates fields.
- Inserts one document to `commandsQueue` with `delivered: false, timeTag: new Date()`.
- Returns `200 { ok: true }`.
- No polling for ack result in this endpoint — fire-and-forget.

The `_id` of the supervised tag is in the `listS7PlusTagsForDb` response. The dialog receives the full item (which includes `_id`, `commandOfSupervised`, `protocolSourceObjectAddress`, `protocolSourceASDU`). For the write, `protocolSourceObjectAddress` is used directly — it is the same for both the supervised tag and its paired command tag.

Note: `commandsEnabled` must be `true` for the connection, and `commandOfSupervised != 0` must hold for the supervised tag (meaning a command tag was created). If `commandOfSupervised == 0`, the tag was created without a paired command tag (commands disabled at browse time). The UI should disable the write button for those tags.

### Non-Datablock Tags (MArea/QArea)

In S7CommPlus, the `Browse()` call returns `VarInfo` entries for all browsable variables, including:
- Datablocks: `varInfo.Name` starts with a DB name, e.g. `"DB_Conveyor.Speed"` → `protocolSourceBrowsePath = "DB_Conveyor"`
- Memory area: `varInfo.Name` like `"MArea.MW100"` → `protocolSourceBrowsePath = "MArea"`
- Output area: `varInfo.Name` like `"QArea.QB0"` → `protocolSourceBrowsePath = "QArea"`

These are all stored in `realtimeData` today. The only gap is `s7plusDatablocks` — it gets populated by `GetListOfDatablocks()` which only returns actual DB-type objects.

Driver change in `Program.cs` (startup, after `GetListOfDatablocks` and `BrowseAndCreateTags`):
1. Query `realtimeData` for distinct `protocolSourceBrowsePath` values for this `connectionNumber`.
2. For each path that does not match any known `db_name` in the `RelationIdNameMap`, upsert into `s7plusDatablocks` with `{ db_name: path, db_number: -1, connectionNumber: N }`.
3. Use `upsert: true, filter: { connectionNumber: N, db_name: path }` — idempotent.

This ensures MArea and QArea entries appear in `listS7PlusDatablocks` results alongside real datablocks.

DatablockBrowserPage.vue: render `db_number === -1` as `"—"` in the DB Number column. No other change.

TagTreeBrowserPage: no change needed — it already passes `dbName` as the path prefix, and `listS7PlusTagsForDb` already handles any `protocolSourceBrowsePath` prefix.

---

## Sources

- Codebase: `TagTreeBrowserPage.vue` — `item.value` display, `buildTree`, `patchLeafValues`, `touchExpandedLeafTags`, Vuetify v-treeview usage
- Codebase: `TagsCreation.cs` — `stateTextTrue: "TRUE"`, `stateTextFalse: "FALSE"`, `GetTypeFromAsdu` (Bool/BBool → "digital"), `commandOfSupervised` field creation
- Codebase: `TagMapping.cs` — `ConvertReadValue`: `case bool b: dblValue = b ? 1.0 : 0.0; strValue = b.ToString()` — confirms 0/1 in `value`, "True"/"False" in `valueString`
- Codebase: `MongoCommands.cs` — full command write path; `commandsQueue` document structure; `AddressCache` resolution; `ConvertCommandToPValue`
- Codebase: `index.js` (server_realtime_auth) lines 488-513 — `listS7PlusTagsForDb` query using `$regex` on `protocolSourceBrowsePath`
- Codebase: `dlgcomando.html` — legacy tabular write popup using `window.opener.WebSAGE`; confirmed incompatible with Vue SPA
- Codebase: `tag.model.js` — `commandOfSupervised` field; confirms `value` is Double, `valueString` is String
- Vuetify 3 docs: `v-treeview` `load-children` prop — fires on expand of items with `children: []`; fires only once per item (children cached)
- GitHub vuetifyjs/vuetify issue #20450 — `load-children` in Vuetify 3.7.1 reported broken, closed as "works"; Vuetify 3.10 in use has no known blocking issue

---

*Feature research for: TagTreeBrowser Overhaul (v1.5) — lazy loading, boolean display, value write, MArea/QArea*
*Researched: 2026-03-30*

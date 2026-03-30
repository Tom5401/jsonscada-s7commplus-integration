# Architecture Research

**Domain:** TagTreeBrowser Overhaul — Lazy Loading, Real Value Display, Value Write, Non-Datablock Tags
**Researched:** 2026-03-30
**Confidence:** HIGH (all findings sourced from actual codebase inspection)

## Standard Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          AdminUI (Vue 3 + Vuetify 3.10)                 │
├───────────────────────────┬─────────────────────────────────────────────┤
│  DatablockBrowserPage.vue │  TagTreeBrowserPage.vue (MODIFIED)          │
│  (MODIFIED — add non-DB   │                                             │
│   area rows)              │  v-treeview                                 │
│                           │    :load-children="loadNodeChildren"        │
│  listProtocolConnections  │    :items="rootNodes"                       │
│  listS7PlusDatablocks     │    v-model:opened="openedNodes"             │
│  [+ virtual area rows]    │                                             │
│                           │  PushValueDialog.vue (NEW — reuses OPC     │
│                           │  write API from tabular.html pattern)       │
├───────────────────────────┴─────────────────────────────────────────────┤
│                   HTTP  /Invoke/auth/*                                  │
├───────────────────────────┬─────────────────────────────────────────────┤
│  EXISTING ENDPOINTS       │  NEW ENDPOINT                               │
│                           │                                             │
│  listS7PlusDatablocks     │  listS7PlusChildNodes                       │
│  listS7PlusTagsForDb      │    GET ?connectionNumber=N&path=P           │
│  touchS7PlusActiveTagReq  │    Query: realtimeData WHERE                │
│                           │      protocolSourceConnectionNumber = N     │
│                           │      protocolSourceBrowsePath = P (exact)   │
│                           │    Returns: direct children only            │
├───────────────────────────┴─────────────────────────────────────────────┤
│                    Node.js server_realtime_auth                         │
│                    (index.js — single file, all endpoints inline)       │
├─────────────────────────────────────────────────────────────────────────┤
│                         MongoDB                                         │
│  realtimeData          s7plusDatablocks     activeTagRequests           │
│  (tags + values)       (DB list per conn)   (TTL — driver reads these)  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Status |
|-----------|----------------|--------|
| `TagTreeBrowserPage.vue` | Tree rendering, lazy expand, value refresh, write trigger | MODIFIED |
| `DatablockBrowserPage.vue` | DB list + non-DB area rows (MArea, QArea, IArea) | MODIFIED |
| `PushValueDialog.vue` (new) | OPC write command dialog, reusable from tree leaf nodes | NEW |
| `listS7PlusChildNodes` endpoint | Returns direct children for a given parent path | NEW |
| `listS7PlusTagsForDb` endpoint | Existing — keep for value refresh polling of leaf tags | UNCHANGED |
| `touchS7PlusActiveTagRequests` | Existing — keep for TTL extension on visible leaves | UNCHANGED |

---

## Recommended Project Structure

No new folders required — all changes go into existing files plus one new component:

```
json-scada/src/AdminUI/src/components/
├── TagTreeBrowserPage.vue        # Major rewrite (lazy loading + write + value display)
├── DatablockBrowserPage.vue      # Add non-DB area rows (client-side virtual rows)
└── PushValueDialog.vue           # NEW: extracted write dialog component

json-scada/src/server_realtime_auth/
└── index.js                      # Add listS7PlusChildNodes endpoint (~35 lines)
```

### Structure Rationale

- **PushValueDialog.vue as separate file:** The write dialog needs to be triggered from a tree leaf
  context. Keeping it separate makes TagTreeBrowserPage.vue readable and the dialog reusable if
  another page later needs the same pattern.
- **No new backend files:** The entire backend lives in index.js by convention. The new endpoint
  follows the exact same inline pattern as the other five S7Plus endpoints immediately above it.

---

## Architectural Patterns

### Pattern 1: Vuetify `load-children` for On-Demand Subtree Fetch

**What:** v-treeview fires `loadChildren(item.raw)` exactly once per node when that node is expanded
and its `children` array is empty (`[]`). The function must mutate `item.children` in place (push
items into the array) — Vuetify detects the populated array and renders children.

**When to use:** Every non-leaf node in the tree. Nodes that are leaves have `children: undefined`
(omit the key). Nodes with children to load have `children: []` initially.

**Trade-offs:**
- Fires only once per node (Vuetify caches: once children are populated, the callback is not
  re-invoked on re-expand). This is correct — tag structure does not change at runtime.
- Vuetify 3.7.1 had a regression in load-children (GitHub issue #20450). Installed version is
  3.10 which post-dates that fix. Verify on first integration run.

**Implementation sketch:**
```javascript
// In TagTreeBrowserPage.vue setup()

const rootNodes = ref([])  // replaces treeItems

async function loadNodeChildren(rawItem) {
  // rawItem is the raw item object from the tree (item.raw in Vuetify internals)
  const path = rawItem.id  // e.g. "MyDB" or "MyDB.Struct1"
  const res = await fetch(
    `/Invoke/auth/listS7PlusChildNodes?connectionNumber=${connectionNumber.value}&path=${encodeURIComponent(path)}`
  )
  const children = await res.json()
  // Mutate in place — Vuetify detects this via reactivity on the original array reference
  rawItem.children.push(...children)
}
```

### Pattern 2: Exact `protocolSourceBrowsePath` Match for Direct Child Query

**What:** The existing `listS7PlusTagsForDb` uses a `$regex` prefix match on
`protocolSourceBrowsePath` to fetch ALL tags under a DB. The new `listS7PlusChildNodes` returns
only DIRECT children of a given path.

**Key insight from codebase:** `protocolSourceBrowsePath` is set by `ExtractPathFromName()` in
`TagMapping.cs` as everything in `ungroupedDescription` before the last dot. So for
`ungroupedDescription = "MyDB.Struct1.Tag1"`, `protocolSourceBrowsePath = "MyDB.Struct1"`.

A direct child query is an exact match on `protocolSourceBrowsePath`:

```javascript
// Backend: listS7PlusChildNodes endpoint in index.js
const docs = await db.collection(COLL_REALTIME).find({
  protocolSourceConnectionNumber: connectionNumber,
  protocolSourceBrowsePath: path,         // exact match — direct children only
}).project({
  ungroupedDescription: 1,
  type: 1, value: 1, valueString: 1,
  stateTextTrue: 1, stateTextFalse: 1,
  protocolSourceObjectAddress: 1,
  _id: 1,
}).toArray()
```

**Why this works at every level:**
- Expanding "MyDB": path = "MyDB" → returns tags whose parent is exactly "MyDB"
- Expanding "MyDB.Struct1": path = "MyDB.Struct1" → returns tags whose parent is "MyDB.Struct1"

**Folder vs leaf determination:** A child node is a folder if any document in `realtimeData` has
`protocolSourceBrowsePath = child.ungroupedDescription`. Check with a `distinct` query:

```javascript
// After fetching direct children, determine which are folders
const childPaths = docs.map(d => d.ungroupedDescription)
const folderPaths = await db.collection(COLL_REALTIME).distinct(
  'protocolSourceBrowsePath',
  {
    protocolSourceConnectionNumber: connectionNumber,
    protocolSourceBrowsePath: { $in: childPaths },
  }
)
const folderSet = new Set(folderPaths)
// Build response: folders get children: [], leaves get children: undefined (omitted)
```

Both queries hit the same compound index on `{protocolSourceConnectionNumber, protocolSourceBrowsePath}`.

### Pattern 3: Real Value Display — `type` + `stateTextTrue/False` + `valueString`

**What:** Replace `doc.value` (raw double) with a human-readable string that matches the tabular
viewer behavior.

**Key fields from `realtimeData` schema (all available in existing documents):**
- `type`: `"digital"`, `"analog"`, `"string"`, `"json"`
- `value`: numeric (0.0 / 1.0 for digital, float for analog)
- `valueString`: string representation (e.g. `"0"`, `"1"`, float string)
- `stateTextTrue`: `"TRUE"` for digital tags (set by driver via TagsCreation.cs)
- `stateTextFalse`: `"FALSE"` for digital tags

**Display logic** (matches tabular.html pattern):
```javascript
function formatLeafValue(doc) {
  if (doc.type === 'digital') {
    return doc.value ? (doc.stateTextTrue || 'TRUE') : (doc.stateTextFalse || 'FALSE')
  }
  if (doc.type === 'string' || doc.type === 'json') {
    return doc.valueString || String(doc.value)
  }
  // analog: prefer valueString (may include unit info from driver)
  return doc.valueString || String(doc.value)
}
```

**Implementation:** Add `stateTextTrue`, `stateTextFalse`, `valueString` to projections in both
the new `listS7PlusChildNodes` endpoint and in the existing `listS7PlusTagsForDb` endpoint's
response (currently returns all fields — no change needed there). Update `patchLeafValues()` to
also patch `stateTextTrue` / `stateTextFalse` (static after initial load but should be present).

### Pattern 4: Value Write — OPC Write API via `PushValueDialog.vue`

**What:** json-scada's write path uses the OPC UA-inspired HTTP endpoint at `/Invoke/` with
`ServiceId: 676` (WriteRequest). The backend inserts commands into `commandsQueue`; the driver
picks them up and writes to the PLC.

**Data required from the leaf node's realtimeData document:**
- `_id` (numeric point key) — NodeId in the write request
- `tag` — display name in dialog
- `type` — determines dialog input type
- `value` / `valueString` — current value to pre-populate
- `stateTextTrue` / `stateTextFalse` — button labels for digital commands

**OPC write request structure** (from tabular.html lines 875–910):
```javascript
{
  ServiceId: 676,
  Body: {
    RequestHeader: { Timestamp, RequestHandle, TimeoutHint: 1500,
                     ReturnDiagnostics: 2, AuthenticationToken: null },
    NodesToWrite: [{
      NodeId: { IdType: 1, Id: pointKey, Namespace: 2 },
      AttributeId: 13,
      Value: { Type: 11, Body: numericValue }   // Double
      // or: Value: { Type: 12, Body: stringValue }  // String
    }]
  }
}
// POST to /Invoke/  (NOT /Invoke/auth/ — uses session cookie, not admin JWT)
```

**Key constraint:** The write endpoint is at `/Invoke/` without the auth prefix. It uses the same
session cookie authentication as the rest of the AdminUI. The user must have `sendCommands: true`
in their role.

**Integration in TagTreeBrowserPage.vue:**
```vue
<!-- template -->
<PushValueDialog v-model="showPushDialog" :tag-doc="selectedLeafDoc" />

<!-- leaf append slot -->
<template #append="{ item }">
  <template v-if="item.raw.isLeaf">
    <v-chip size="x-small">{{ item.raw.type }}</v-chip>
    <code class="ml-2 text-caption">{{ formatLeafValue(item.raw) }}</code>
    <v-btn icon="mdi-pencil" size="x-small" variant="text"
           @click.stop="openPushDialog(item.raw)" />
  </template>
</template>
```

### Pattern 5: Non-Datablock Tags — MArea, QArea, IArea as Virtual Root Nodes

**What:** `S7CommPlusConnection.Browse()` adds IArea, QArea, MArea, S7Timers, S7Counters as root
browse nodes alongside DBs (confirmed in S7CommPlusConnection.cs lines 878–882). The driver sets
`VarInfo.Name` = `"MArea.M0_0"` etc., so these tags already follow the exact dot-path convention.

**Consequence in MongoDB:**
- `ungroupedDescription` = `"MArea.M0_0"`
- `protocolSourceBrowsePath` = `"MArea"`

The `listS7PlusChildNodes?path=MArea` query works with no backend changes. The area tags exist in
`realtimeData` — they are just not surfaced in `DatablockBrowserPage.vue`.

**Integration:** Append virtual rows after fetching the DB list in `DatablockBrowserPage.vue`:

```javascript
const VIRTUAL_AREAS = ['IArea', 'QArea', 'MArea', 'S7Timers', 'S7Counters']

// After fetching datablocks:
const areaRows = VIRTUAL_AREAS.map(name => ({
  db_name: name,
  db_number: null,
  isVirtualArea: true,
}))
datablocks.value = [...realDbs, ...areaRows]
```

The "Browse Tags" button for area rows opens TagTreeBrowserPage with `?db=MArea&connectionNumber=N` —
the tree page treats it identically to a DB. If no tags exist for that area, the tree shows empty.

---

## Data Flow

### Flow 1: Initial Tree Load (Lazy — Root Level Only)

```
User opens TagTreeBrowserPage (?db=MyDB&connectionNumber=1)
    |
onMounted()
    |
Fetch /Invoke/auth/listS7PlusChildNodes?connectionNumber=1&path=MyDB
    |
Backend: realtimeData.find({ protocolSourceConnectionNumber:1,
                              protocolSourceBrowsePath:"MyDB" })
         + distinct for folder detection
    |
Returns: [{ id:"MyDB.Tag1", name:"Tag1", isLeaf:true, children:undefined, ... },
           { id:"MyDB.Struct1", name:"Struct1", isLeaf:false, children:[] }]
    |
rootNodes = [{ id:"MyDB", name:"MyDB", children: [Tag1, Struct1] }]
    |
v-treeview renders root; auto-expand fires load-children for root
    → root already has children (from initial load), so load-children is NOT triggered for root
    → Struct1 has children:[] → load-children will fire when user expands Struct1
```

### Flow 2: On-Demand Child Load (Any Depth)

```
User expands "MyDB.Struct1"
    |
Vuetify detects children:[] on Struct1 → fires loadNodeChildren(Struct1.raw)
    |
Fetch /Invoke/auth/listS7PlusChildNodes?connectionNumber=1&path=MyDB.Struct1
    |
Backend returns child nodes with folder/leaf classification
    |
Struct1.raw.children.push(...childNodes)   // mutate in place
    |
Vuetify re-renders Struct1 with its children
Value refresh picks up any new leaf nodes
```

### Flow 3: Value Refresh (5-Second Poll)

```
refreshTimer fires every 5 seconds
    |
Fetch /Invoke/auth/listS7PlusTagsForDb?connectionNumber=1&dbName=MyDB
    (existing endpoint — fetches ALL tag values for the DB)
    |
patchLeafValues(rootNodes, freshDocs)
    walk populated (lazy-loaded) tree, update value/valueString on each matching leaf
    |
touchS7PlusActiveTagRequests(visibleLeafAddresses)
    extends TTL for leaves under currently open nodes
```

### Flow 4: Value Write

```
User clicks write icon on leaf node
    |
openPushDialog(item.raw)
selectedLeafDoc = { _id, tag, type, value, valueString, stateTextTrue, stateTextFalse }
showPushDialog = true
    |
PushValueDialog renders appropriate input
User confirms
    |
POST /Invoke/  { ServiceId: 676, NodesToWrite: [{ NodeId.Id: _id, Value: ... }] }
    |
server_realtime_auth OPC handler validates sendCommands permission
    → inserts into commandsQueue MongoDB collection
    |
S7CommPlusClient driver polls commandsQueue
    → writes to PLC → value propagates back to realtimeData on next read cycle
```

---

## Integration Points

### New vs Modified — Explicit Summary

| Component / Endpoint | Action | Change Description |
|---|---|---|
| `TagTreeBrowserPage.vue` | MODIFIED | Remove `buildTree()` + full-load; replace with `loadNodeChildren()` + `load-children` prop; add `formatLeafValue()`; add write button per leaf; keep `patchLeafValues()` with added fields |
| `DatablockBrowserPage.vue` | MODIFIED | Append 5 virtual area rows after DB list fetch |
| `PushValueDialog.vue` | NEW FILE | Write dialog receiving tag doc, posting OPC write, showing ack/error |
| `listS7PlusChildNodes` | NEW ENDPOINT | ~35 lines in index.js; exact browsePath match + folder detection; add `idx_conn_browsepath` index |
| `listS7PlusTagsForDb` | UNCHANGED | Still used for value polling |
| `touchS7PlusActiveTagRequests` | UNCHANGED | Still used for TTL extension |
| `listS7PlusDatablocks` | UNCHANGED | Area rows appended client-side |

### MongoDB Index Addition

```javascript
// Add to startup index-ensure block in index.js
await db.collection(COLL_REALTIME).createIndex(
  { protocolSourceConnectionNumber: 1, protocolSourceBrowsePath: 1 },
  { name: 'idx_conn_browsepath' }
)
```

This index is idempotent. It benefits both the new exact-match query and the existing regex-prefix
query in `listS7PlusTagsForDb`.

---

## Suggested Build Order

| Step | Work Item | Dependency | Layer |
|---|---|---|---|
| 1 | Add `{protocolSourceConnectionNumber, protocolSourceBrowsePath}` index to `realtimeData` | None | Backend |
| 2 | Add `listS7PlusChildNodes` endpoint (exact match + folder detection) | Step 1 | Backend |
| 3 | Rewrite `TagTreeBrowserPage.vue` lazy loading (replace buildTree with load-children + initial root load) | Step 2 | Frontend |
| 4 | Fix value display in tree (`formatLeafValue()`, include stateText fields in projections) | Step 3 | Frontend |
| 5 | Add virtual area rows to `DatablockBrowserPage.vue` | Step 2 (tree handles any root) | Frontend |
| 6 | Create `PushValueDialog.vue`, wire write button into tree leaf append slot | Steps 3 + 4 | Frontend |

Steps 4 and 5 are independent of each other and can be done in parallel after step 3.

---

## Anti-Patterns

### Anti-Pattern 1: Full-Load at Initial Open

**What people do:** Keep `listS7PlusTagsForDb` as the initial data source and build the full tree
client-side with `buildTree()`.

**Why it's wrong:** 100k+ tags → multi-MB JSON → browser hang → possible OOM. The entire purpose
of v1.5 is to eliminate this.

**Do this instead:** Load only the direct children of the root node on open via `listS7PlusChildNodes`.
Deeper levels load on demand as the user expands.

### Anti-Pattern 2: Regex Prefix Match in the New Endpoint

**What people do:** Reuse the existing `$regex: '^dbName(\\.|$)'` pattern from `listS7PlusTagsForDb`
in the new child endpoint.

**Why it's wrong:** Returns all descendants, not just direct children. Expanding the root of a
100k-tag DB downloads all 100k tags at once — same problem as anti-pattern 1.

**Do this instead:** Exact string match on `protocolSourceBrowsePath`. Returns only the direct
children of the expanded node.

### Anti-Pattern 3: Assigning a New Array Instead of Mutating In Place

**What people do:** In `loadNodeChildren`, `rawItem.children = fetchedChildren` (assignment).

**Why it's wrong:** Vuetify tracks the original empty array reference for reactivity. Replacing it
breaks the reactive connection — children do not render.

**Do this instead:** `rawItem.children.push(...fetchedChildren)` — mutate the existing array
reference.

### Anti-Pattern 4: Using `children: undefined` for Folder Nodes

**What people do:** Return all nodes without a `children` key, expecting the tree to figure out
expandability later.

**Why it's wrong:** Vuetify v-treeview only fires `load-children` when `item.children.length === 0`
AND `loadChildren` prop is set. A node with `children: undefined` (no key) renders as a leaf with
no expand arrow and never triggers lazy loading.

**Do this instead:** Folder nodes: `children: []`. Leaf nodes: omit `children` entirely.

### Anti-Pattern 5: Special-Casing Non-DB Areas in the Backend

**What people do:** Add an `isArea` flag or separate endpoint for MArea/QArea queries.

**Why it's wrong:** Unnecessary — the dot-path convention is identical for all root nodes. `path=MArea`
works the same as `path=MyDB` in `listS7PlusChildNodes`. No special handling required.

**Do this instead:** Virtual area rows in the DatablockBrowserPage open TagTreeBrowserPage with
`?db=MArea`. The tree page is agnostic about whether the root is a DB or an area.

---

## Sources

- `TagTreeBrowserPage.vue` (inspected 2026-03-30): current buildTree / patchLeafValues / touchExpandedLeafTags implementation
- `DatablockBrowserPage.vue` (inspected 2026-03-30): browseDatablock integration point
- `server_realtime_auth/index.js` lines 460–560 (inspected 2026-03-30): existing S7Plus endpoint patterns, index-creation pattern
- `TagMapping.cs` `ExtractPathFromName()` (inspected 2026-03-30): confirms protocolSourceBrowsePath = parent path convention
- `TagsCreation.cs` `newRealtimeDoc()` (inspected 2026-03-30): confirms ungroupedDescription = full dot path; stateTextTrue/False = "TRUE"/"FALSE" for digital type
- `S7CommPlusConnection.cs` lines 877–882 (inspected 2026-03-30): confirms IArea/QArea/MArea/S7Timers/S7Counters area names as root browse nodes
- `VTreeviewChildren.js` Vuetify 3.10 node_modules (inspected 2026-03-30): confirms `loadChildren(item.raw)` fires when `item.children.length === 0`; mutation in-place required
- `tabular.html` lines 872–930 (inspected 2026-03-30): OPC WriteRequest ServiceId=676 structure for push-value

---
*Architecture research for: TagTreeBrowser Overhaul (v1.5)*
*Researched: 2026-03-30*

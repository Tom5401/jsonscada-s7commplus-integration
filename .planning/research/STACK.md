# Stack Research

**Domain:** TagTreeBrowser Overhaul — v1.5 additions to existing json-scada AdminUI
**Researched:** 2026-03-30
**Confidence:** HIGH (all key claims verified against installed source or direct code inspection)

---

## Scope

This document covers only the **new stack surface** needed for v1.5 features. Everything validated
in v1.4 (Vue 3, Vuetify 3.10, vue-router 4, MongoDB driver for Node.js, Express in
server_realtime_auth) is unchanged and re-confirmed below.

---

## Installed Versions (confirmed, unchanged from v1.4)

| Component | Version | Source |
|-----------|---------|--------|
| Vuetify | 3.10.12 | `node_modules/vuetify/package.json` |
| Vue 3 | ^3.4.31 | `AdminUI/package.json` |
| vue-router | ^4.4.0 | `AdminUI/package.json` |
| Vite | ^7.3.1 | `AdminUI/package.json` |
| vite-plugin-vuetify | ^2.0.3 | `AdminUI/package.json` |

**No new npm packages are required for v1.5.**

---

## Feature 1: Lazy Tree Loading (load-children)

### Stack: Vuetify 3.10.12 v-treeview with `loadChildren` prop — already installed

**Verified from installed source** (`VTreeviewChildren.js:15,60-64`, `VTreeview.d.ts:449-451`):

```js
// Trigger condition in VTreeviewChildren.js line 60-64:
if (item?.children?.length === 0) {
  isLoading.add(item.value);
  await props.loadChildren(item.raw);  // receives the raw item object
}
```

TypeScript signature confirmed:

```ts
// VTreeview.d.ts line 449:
loadChildren?: (item: unknown) => Promise<void>
```

**Contract for implementation:**

- Every expandable (non-leaf) node must be initialized with `children: []` — empty array, not `undefined`.
- The `loadChildren` callback receives `item.raw` (the plain JS object you put in the items array).
- Inside the callback, **push** fetched children into `item.children` — do NOT assign `item.children = newArray`. Array mutation preserves the reference Vuetify's internal model tracks. Replacement silently breaks reactivity.
- Return a Promise (async function satisfies this). Vuetify ignores the resolved value.
- Vuetify displays a loading spinner automatically while the Promise is pending.
- Once `children.length > 0`, the callback will NOT fire again for that node (one-shot lazy load per node).

**New backend endpoint required:**

`GET /Invoke/auth/listS7PlusTagChildren?connectionNumber=N&parentPath=X`

Returns the direct children of `parentPath` — both sub-folder path segments and leaf tag documents. This replaces the v1.4 pattern of fetching all tags for a DB upfront, which does not scale to 100k+ tag databases.

Query pattern for the endpoint:

```js
// Fetch all tags whose ungroupedDescription is directly under parentPath
// (i.e. starts with parentPath + '.' and has no further dots in the remaining segment).
// Use ungroupedDescription (full hierarchical path) — not protocolSourceBrowsePath (parent path).
const docsUnder = await db.collection('realtimeData').find({
  protocolSourceConnectionNumber: connectionNumber,
  ungroupedDescription: { $regex: new RegExp('^' + escapedParentPath + '\\.') }
}).toArray()
// Then group by next segment to build folder/leaf list and return.
```

This follows the exact same auth-guard pattern as `listS7PlusTagsForDb` (index.js:488). The `ungroupedDescription` field is used for path navigation, consistent with how v1.4's `buildTree()` was written.

---

## Feature 2: Real Value Display (boolean/enum formatting)

### Stack: No new library — pure Vue computed logic using existing realtimeData fields

Fields already present in every `realtimeData` document (confirmed in `TagsCreation.cs`):

| Field | Content | Example |
|-------|---------|---------|
| `type` | `"digital"` \| `"analog"` \| `"string"` \| `"json"` | `"digital"` |
| `value` | numeric (0 or 1 for digital, float for analog) | `1` |
| `stateTextTrue` | label for value=1 on digital tags | `"TRUE"` |
| `stateTextFalse` | label for value=0 on digital tags | `"FALSE"` |
| `valueString` | string representation for string/json types | `""` |

The S7CommPlusClient driver sets `stateTextTrue = "TRUE"` and `stateTextFalse = "FALSE"` for all
Bool/BBool tags (`TagsCreation.cs:206-208`). This is the same source the tabular viewer uses
(`tabular.html:1865-1867`).

Display logic (pure Vue, no library):

```js
function formatValue(item) {
  if (item.type === 'digital') {
    return item.value ? (item.stateTextTrue || 'TRUE') : (item.stateTextFalse || 'FALSE')
  }
  if (item.type === 'string') return item.valueString || ''
  return item.value
}
```

The `listS7PlusTagChildren` endpoint must project these fields: `type`, `value`, `stateTextTrue`,
`stateTextFalse`, `valueString`, `protocolSourceObjectAddress`, `ungroupedDescription`, `_id`,
`commandOfSupervised`, `origin`.

---

## Feature 3: Value Writing from TagTreeBrowser

### Stack: Native Vuetify v-dialog + existing `/Invoke/auth` OPC WriteRequest endpoint

**The `dlgcomando.html` popup is NOT reusable from Vue.** It is a legacy jQuery popup opened via
`window.open()` from `tabular.html` and communicates with its parent via `window.opener` DOM
manipulation (`tabular.html:372,768`). This pattern requires the caller to literally be
`tabular.html` in the parent window context. Called from a Vue SPA route, `window.opener` is null
and all communication silently fails.

**Correct approach:** A native `v-dialog` inside `TagTreeBrowserPage.vue` that calls the existing
OPC WriteRequest endpoint directly.

Write request wire format (confirmed from `tabular.html:875-960`):

```js
// POST /Invoke/auth  (the main OPC API endpoint)
{
  "ServiceId": 671,  // opc.ServiceCode.WriteRequest
  "Body": {
    "NodesToWrite": [{
      "NodeId": { "IdType": 1, "Id": <tag._id as number>, "Namespace": 2 },
      "AttributeId": 13,
      "Value": { "Type": 11, "Body": <numericValue> }
    }]
  }
}
```

For digital tags: convert TRUE → 1, FALSE → 0 before sending (pattern from `tabular.html:952-953`).

The backend handler for this is already live at `index.js:754` (WriteRequest case). It is
authenticated, guarded for user command rights, and writes to `commandsQueue` MongoDB collection.

**Fields needed from the tag document for the dialog:**

| Field | Use |
|-------|-----|
| `_id` | NodeId.Id in the write request |
| `type` | Determines input widget (select for digital, number input for analog) |
| `stateTextTrue` / `stateTextFalse` | Labels in digital select dropdown |
| `commandOfSupervised` | If > 0, write goes to the command tag `_id` (not the supervised tag). Start v1.5 by checking `origin === 'supervised'` and dispatching to `commandOfSupervised` when non-zero. |

No new npm packages. Vuetify `v-dialog` + `v-text-field` / `v-select` covers the UX. Fetch is
already used throughout AdminUI pages.

---

## Feature 4: Non-Datablock Tag Support (MArea, QArea, IArea)

### Stack: No new library — backend query change + DatablockBrowserPage.vue UI addition

**How non-datablock areas are structured (confirmed from driver source):**

`S7CommPlusConnection.Browse()` (`S7CommPlusConnection.cs:878-880`) adds five block nodes alongside
datablocks:

```csharp
vars.AddBlockNode(eNodeType.Root, "IArea",    Ids.NativeObjects_theIArea_Rid,    0x90010000);
vars.AddBlockNode(eNodeType.Root, "QArea",    Ids.NativeObjects_theQArea_Rid,    0x90020000);
vars.AddBlockNode(eNodeType.Root, "MArea",    Ids.NativeObjects_theMArea_Rid,    0x90030000);
vars.AddBlockNode(eNodeType.Root, "S7Timers", Ids.NativeObjects_theS7Timers_Rid, 0x90050000);
vars.AddBlockNode(eNodeType.Root, "S7Counters",Ids.NativeObjects_theS7Counters_Rid, 0x90060000);
```

These produce `realtimeData` documents where:
- `ungroupedDescription` = `"MArea.Var1"`, `"QArea.Output1"`, etc.
- `protocolSourceBrowsePath` = `"MArea"`, `"QArea"`, etc. (the root area name)

These are NOT stored in `s7plusDatablocks` (that collection only holds datablocks from
`GetListOfDatablocks`, which returns only DB-type blocks). The area names are constant and
known for S7-1200/S7-1500.

**What changes:**

1. `DatablockBrowserPage.vue`: Add a second section or v-divider group showing the 5 static area
   names (`IArea`, `QArea`, `MArea`, `S7Timers`, `S7Counters`). Each row opens
   `TagTreeBrowserPage` in a new tab with `db=MArea&connectionNumber=N`. No new backend endpoint
   is needed — area names are hardcoded constants for this PLC family. If filtering to only areas
   with actual tags is needed in a later phase, add a `listS7PlusNonDbAreas?connectionNumber=N`
   endpoint that queries distinct root `ungroupedDescription` prefixes.

2. `TagTreeBrowserPage.vue`: No structural change needed. The lazy-load endpoint handles any
   root path (e.g. `parentPath=MArea`) identically to a DB path. The `db=` query param name
   may be renamed to `rootPath=` for clarity, but this is cosmetic.

3. The `listS7PlusTagChildren` endpoint (from Feature 1) handles area-rooted paths with the same
   `ungroupedDescription` regex query — no special-casing required.

---

## Installation

```bash
# No new packages needed.
# All capabilities are present in the currently installed stack.
```

---

## Alternatives Considered

| Recommended | Alternative | Why Not |
|-------------|-------------|---------|
| `loadChildren` with array mutation (`push`) | Replace children ref (`item.children = newArr`) | Replacement breaks Vuetify's internal item model; the component tracks the original array reference. Confirmed from source. |
| Native v-dialog write dialog in Vue | Reuse `dlgcomando.html` popup | dlgcomando relies on `window.opener` DOM manipulation; silently fails when called from a Vue SPA route (no opener context). |
| New `listS7PlusTagChildren` endpoint | Reuse `listS7PlusTagsForDb` with client-side tree build | Fetching all tags per DB defeats lazy loading; 100k+ tag DB would load everything on first expand. |
| Static area name list in DatablockBrowserPage | New distinct-query endpoint per connection | For S7-1200/S7-1500 the 5 area names are stable; a distinct query adds complexity without benefit in the PoC. Optional for later phases if empty-area filtering is needed. |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `item.children = fetchedArray` (reference replacement) | Breaks v-treeview internal model; node appears loaded but children don't render | `item.children.push(...fetchedArray)` (in-place mutation) |
| `dlgcomando.html` via window.open from TagTreeBrowser | Requires window.opener context only available inside tabular.html; returns null from SPA | Native v-dialog in TagTreeBrowserPage.vue calling `/Invoke/auth` WriteRequest directly |
| `listS7PlusTagsForDb` for per-node lazy load | Fetches entire DB subtree at once; negates lazy loading benefit | New `listS7PlusTagChildren` endpoint returning only direct children of a path |
| Initializing expandable nodes with `children: undefined` or `children: null` | Vuetify only triggers loadChildren when `children.length === 0`; missing or null children property skips the callback | Initialize all expandable nodes with `children: []` |

---

## Version Compatibility

| Package | Version | Notes |
|---------|---------|-------|
| vuetify | 3.10.12 | `loadChildren` prop confirmed in `VTreeviewChildren.d.ts:60` and `.js:15,61-64`. Trigger: `children.length === 0`. |
| vue | 3.4.x | Array mutation (`push`) is reactive. No `Vue.set` needed in Vue 3. |
| vue-router | 4.4.x | No change — existing `?db=` and `?connectionNumber=` query params are sufficient. |

---

## Sources

- `node_modules/vuetify/lib/components/VTreeview/VTreeviewChildren.js:15,60-64` — loadChildren prop definition and trigger condition (HIGH confidence — direct installed source inspection)
- `node_modules/vuetify/lib/components/VTreeview/VTreeview.d.ts:449-451` — `loadChildren: (item: unknown) => Promise<void>` type signature (HIGH confidence — direct installed source)
- `json-scada/src/S7CommPlusClient/TagsCreation.cs:206-208,279-280` — stateTextTrue/False = "TRUE"/"FALSE" for digital type; rtData schema fields confirmed (HIGH confidence — project source)
- `json-scada/src/S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs:878-880` — IArea/QArea/MArea/S7Timers/S7Counters as named block nodes with their Rid constants (HIGH confidence — project source)
- `json-scada/src/AdminUI/public/tabular.html:875-960,952-953,1865-1867` — OPC WriteRequest wire format, digital value conversion, boolean display pattern (HIGH confidence — project source)
- `json-scada/src/server_realtime_auth/index.js:754,488` — WriteRequest handler at line 754; listS7PlusTagsForDb query pattern at line 488 (HIGH confidence — project source)
- [Vuetify GitHub issue #20450](https://github.com/vuetifyjs/vuetify/issues/20450) — load-children regression in 3.7.1, resolved as works-as-intended; project is on 3.10.12 (MEDIUM confidence — community issue)

---
*Stack research for: v1.5 TagTreeBrowser Overhaul*
*Researched: 2026-03-30*

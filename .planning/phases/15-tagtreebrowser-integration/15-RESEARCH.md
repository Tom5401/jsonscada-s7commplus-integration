# Phase 15: TagTreeBrowser & Integration - Research

**Researched:** 2026-03-26
**Domain:** Vue 3 / Vuetify 3.10 tree component, client-side path parsing, live-poll pattern
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Auto-expand the first level on load — direct children of the DB (top-level structs and tags) are immediately visible without any user click.
- **D-02:** Intermediate struct nodes (inferred from child paths, no own realtimeData doc) show folder icon + name only. No value, no type, no count badge. Clean separation: containers are folders, leaf tags show data.
- **D-03:** Each leaf tag node shows three pieces of information:
  1. `type` — from the `type` field on the realtimeData doc (e.g., "analog", "digital")
  2. `value` — raw value from `realtimeData.value` (no stateTextTrue/stateTextFalse formatting)
  3. `protocolSourceObjectAddress` — the full address string (e.g., `"DBName".SubStruct.Tag`) for copy-paste into TIA Portal
- **D-04:** When `originDbName === ""`, render as plain text (dash or blank) — no link, no chip. Only non-empty `originDbName` values become clickable links.
- **D-05:** Re-fetch `listS7PlusTagsForDb` every 5 seconds (same periodic pattern as alarm viewer). Update only the `value` field on leaf nodes in-place to preserve expand/collapse state across refreshes. Also call `touchS7PlusActiveTagRequests` for all currently expanded leaf tags on each refresh cycle to prevent TTL expiry.
- **D-06:** Follow S7PlusAlarmsViewerPage conventions: `v-container fluid`, `density="compact"`, `class="elevation-1"`, cookie-based auth (`/Invoke/auth/...`).

### Claude's Discretion

- Tree component choice: use `v-treeview` (Vuetify 3.10, already available) if it fits the lazy expand + in-place value update requirements cleanly; otherwise a recursive component is acceptable. Researcher/planner to decide based on v-treeview's open-state API.
- Path parsing: use `protocolSourceBrowsePath` (e.g., `DBName.SubStruct.Tag`) for building the hierarchy — strip the leading DB name segment, then split on `.` for remaining depth. TAGTREE-01 describes `protocolSourceObjectAddress` format for illustration, but `protocolSourceBrowsePath` is the cleaner field (no quote stripping needed).

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| TAGTREE-01 | Operator sees configured tags for a datablock displayed as a lazy-expanding tree, tree structure derived by parsing `protocolSourceObjectAddress` strings | Path parsing algorithm using `protocolSourceBrowsePath`; v-treeview open state API |
| TAGTREE-02 | Operator sees live tag values for expanded leaf tags, auto-refreshing every 5 seconds; expanding a node triggers `touchS7PlusActiveTagRequests` | setInterval pattern from S7PlusAlarmsViewerPage; touch endpoint confirmed at lines 517–555 |
| TAGTREE-03 | Operator sees the data type of each leaf tag in the tree (from `type` field in realtimeData) | `type` field confirmed in TagsCreation.cs (`[BsonDefaultValue("digital")]`) |
| TAGTREE-04 | `TagTreeBrowserPage` accepts `?db=<name>&connectionNumber=<N>` query params on load | `useRoute()` from Vue Router; confirmed pattern in CONTEXT.md canonical refs |
| INTEGRATION-01 | In `S7PlusAlarmsViewerPage`, `originDbName` cell is a clickable link opening TagTreeBrowserPage in new tab | Confirmed `window.open(url, '_blank')` pattern; `/#/s7plus-tag-tree` hash URL required; `item.connectionId` is the connection number |
</phase_requirements>

---

## Summary

Phase 15 is a pure-frontend phase: build `TagTreeBrowserPage.vue` and wire the alarms viewer. No backend changes are required — all three API endpoints (`listS7PlusTagsForDb`, `touchS7PlusActiveTagRequests`, route `/s7plus-tag-tree`) were delivered in Phases 13 and 14. The router does NOT yet have a `/s7plus-tag-tree` route entry — that must be added in this phase. The DashboardPage already has the `datablockBrowser` shortcut card (pointing to `/s7plus-datablocks`) so no dashboard change is needed for this phase.

The core technical challenge is tree construction: flatten the array of realtimeData docs returned by `listS7PlusTagsForDb`, parse each `protocolSourceBrowsePath` by stripping the leading DB-name segment and splitting on `.`, then accumulate into a nested-node structure keyed by path prefix. On periodic refresh, the same flat array is re-fetched and only leaf `.value` fields are updated in-place; the tree structure is not rebuilt so expand/collapse state is preserved.

The `v-treeview` component in Vuetify 3.10 supports `v-model:opened` for controlling which nodes are open and provides an `item` slot for custom rendering — making it viable for this use case. However, because in-place value updates on leaf nodes must not collapse the tree, the implementation must maintain the `opened` set externally and never replace the entire items array on refresh (only mutate leaf `.value` fields). A known Vuetify 3.9+ bug causes opened items to auto-activate in production builds; this is cosmetic and does not affect functionality for this read-only tree.

**Primary recommendation:** Use `v-treeview` with `v-model:opened` and a custom `item` slot. Build the tree once on mount; on each 5-second tick mutate only leaf value fields, never replace the root items array.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Vuetify | 3.10 (pinned) | `v-treeview`, icons, layout | Already installed; v-treeview fully available in 3.x |
| Vue 3 | ^3.4.31 | Composition API (`ref`, `watch`, `onMounted`, `onUnmounted`) | Project standard |
| Vue Router | ^4.4.0 | `useRoute()` for query params | Project standard |
| @mdi/font | 7.4.47 | `mdi-folder`, `mdi-tag` icons | Already loaded |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| vue-i18n | ^11.1.2 | i18n keys for new UI strings | Add `tagTreeBrowser` key to en.json |

### Installation
No new packages needed — all dependencies are already in package.json.

---

## Architecture Patterns

### Recommended Project Structure
```
json-scada/src/AdminUI/src/
├── components/
│   └── TagTreeBrowserPage.vue   (NEW — primary deliverable)
├── router/
│   └── index.js                 (MODIFY — add /s7plus-tag-tree route)
├── components/
│   └── S7PlusAlarmsViewerPage.vue  (MODIFY — originDbName link)
└── locales/
    └── en.json                  (MODIFY — add tagTreeBrowser key)
```

### Pattern 1: Route Registration (add to router/index.js)
**What:** Import the new component and push a route entry into the `routes` array.
**When to use:** Every new AdminUI page needs this exact pattern.
**Example:**
```javascript
// Source: json-scada/src/AdminUI/src/router/index.js (existing pattern)
import TagTreeBrowserPage from '../components/TagTreeBrowserPage.vue'

// In routes array:
{ path: '/s7plus-tag-tree', component: TagTreeBrowserPage },
```

### Pattern 2: Query-param Read on Mount
**What:** Use `useRoute()` to read `route.query.db` and `route.query.connectionNumber` inside `onMounted`.
**When to use:** Any page that receives parameters from a caller via URL.
**Example:**
```javascript
// Source: Vue Router 4 docs + CONTEXT.md D-04
import { useRoute } from 'vue-router'
const route = useRoute()
onMounted(async () => {
  document.documentElement.style.overflowY = 'scroll'
  const dbName = route.query.db
  const connectionNumber = parseInt(route.query.connectionNumber, 10)
  await loadTree(dbName, connectionNumber)
})
```

### Pattern 3: 5-second Auto-refresh (copy from S7PlusAlarmsViewerPage)
**What:** `setInterval` in `onMounted`, `clearInterval` in `onUnmounted`. On tick: re-fetch tags, update leaf values only, touch active tags.
**When to use:** Any page with live auto-poll.
**Example:**
```javascript
// Source: json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue (line 346-354)
let refreshTimer = null
onMounted(async () => {
  document.documentElement.style.overflowY = 'scroll'
  await loadTree(dbName, connectionNumber)
  refreshTimer = setInterval(() => refreshValues(dbName, connectionNumber), 5000)
})
onUnmounted(() => {
  document.documentElement.style.overflowY = 'hidden'
  if (refreshTimer) clearInterval(refreshTimer)
})
```

### Pattern 4: Tree Construction from Flat realtimeData Docs
**What:** Build a recursive node structure from the flat array returned by `listS7PlusTagsForDb`.
**When to use:** Once on load; never repeated on refresh.
**Algorithm:**
```
For each realtimeData doc:
  browsePath = doc.protocolSourceBrowsePath  // e.g. "DBName.SubStruct.Tag"
  segments = browsePath.split('.')
  // segments[0] is the DB name — skip it
  remaining = segments.slice(1)              // e.g. ["SubStruct", "Tag"]
  Walk the tree to insert a node for each segment.
  If remaining.length === 1: leaf node (attach doc reference for value/type/address)
  Otherwise: create intermediate folder nodes as needed.
```
**Preserve expand/collapse on refresh:** Keep tree items array in a `ref`. On refresh, walk leaf nodes and mutate `.value` in-place — never replace the array reference.

### Pattern 5: touchS7PlusActiveTagRequests Call
**What:** POST array of `{connectionNumber, protocolSourceObjectAddress}` for all currently expanded leaf tags.
**When to use:** On node expand AND on each 5-second refresh tick for all currently visible leaf tags.
**Example:**
```javascript
// Source: json-scada/src/server_realtime_auth/index.js lines 517-555
const touchTags = async (connectionNumber, leafTags) => {
  if (!leafTags.length) return
  await fetch('/Invoke/auth/touchS7PlusActiveTagRequests', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(
      leafTags.map(tag => ({
        connectionNumber,
        protocolSourceObjectAddress: tag.protocolSourceObjectAddress,
      }))
    ),
  })
}
```

### Pattern 6: v-treeview with Custom Item Slot (Vuetify 3)
**What:** Use `v-treeview` with `items`, `item-value`, `item-title`, `v-model:opened`, and a scoped `item` slot.
**Key props:**
- `:items="treeItems"` — reactive array of nested node objects
- `item-value="id"` — unique key per node (use full path string)
- `item-title="name"` — display label
- `v-model:opened="openedNodes"` — Set/Array of open node IDs; maintain externally to survive refreshes
- `item-children="children"` — property name for child array

**Custom slot for leaf vs. folder differentiation:**
```html
<!-- Source: Vuetify 3 v-treeview slot API (vuetifyjs.com/en/api/v-treeview) -->
<v-treeview
  :items="treeItems"
  item-value="id"
  item-title="name"
  item-children="children"
  v-model:opened="openedNodes"
>
  <template #prepend="{ item }">
    <v-icon v-if="item.isLeaf">mdi-tag</v-icon>
    <v-icon v-else>mdi-folder</v-icon>
  </template>
  <template #append="{ item }">
    <template v-if="item.isLeaf">
      <v-chip size="x-small" class="ml-1">{{ item.type }}</v-chip>
      <code class="ml-2 text-caption">{{ item.value }}</code>
      <span class="ml-2 text-caption text-medium-emphasis font-monospace">{{ item.address }}</span>
    </template>
  </template>
</v-treeview>
```

### Pattern 7: Alarms Viewer originDbName Link
**What:** Replace plain `originDbName` cell with a conditional `<a>` or `<router-link>` that opens TagTreeBrowserPage in a new tab.
**Key detail:** Use `window.open` with `/#/s7plus-tag-tree` hash prefix (createWebHashHistory requires `#`). The `connectionId` field on each alarm document is the connection number.
**Example:**
```html
<!-- Source: DatablockBrowserPage.vue browseDatablock() + CONTEXT.md specifics -->
<template #[`item.originDbName`]="{ item }">
  <template v-if="item.originDbName">
    <a
      :href="`/#/s7plus-tag-tree?db=${encodeURIComponent(item.originDbName)}&connectionNumber=${item.connectionId}`"
      target="_blank"
      @click.stop=""
    >{{ item.originDbName }}</a>
  </template>
  <template v-else>
    <span>-</span>
  </template>
</template>
```

### Anti-Patterns to Avoid
- **Replacing items array on refresh:** `treeItems.value = buildTree(docs)` on every tick destroys expand state. Instead, walk existing leaf nodes and patch `.value`.
- **Using `item-children` as a lazy-load callback for v-treeview:** Vuetify 3's `item-children` prop is a function only when doing async-lazy expansion. For this phase, the full flat tag list is fetched once and built client-side — pass all children synchronously.
- **Building the URL without the `/#` hash prefix:** `createWebHashHistory` routes all require the `#` fragment. Using `/s7plus-tag-tree` (without `/#`) will cause a 404 in a new tab.
- **Using `sessionStorage` for auth:** Auth uses HTTP-only cookie set by the server at `/Invoke/auth/signin`. Cookie persists across tabs automatically. No localStorage/sessionStorage token copying needed.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Tree open-state binding | Custom `openedSet` + click handler | `v-model:opened` on `v-treeview` | Built-in reactive sync; toggling works out of the box |
| Icon selection | Custom icon map | `mdi-folder` + `mdi-tag` already loaded via @mdi/font | Zero-cost, already in bundle |
| Cross-tab auth | JWT copy to localStorage | HTTP cookie (already set) | Cookie is sent to same origin automatically; no cross-tab auth issue |

**Key insight:** The hard part is client-side path parsing and in-place leaf value mutation — no library needed beyond what's already installed.

---

## Runtime State Inventory

> Not applicable — this is a greenfield frontend page addition. No rename/migration involved.

---

## Common Pitfalls

### Pitfall 1: Tree Rebuilt on Every Refresh — Expand State Lost
**What goes wrong:** Calling `treeItems.value = buildTree(newDocs)` on every 5-second tick resets `openedNodes` visually (or forces a reconcile that collapses everything).
**Why it happens:** Replacing the array reference triggers Vue's full re-render; v-treeview loses its open-state unless `openedNodes` is externally maintained AND the new item objects have the same `id` values.
**How to avoid:** Build the tree once on mount. On refresh, walk `treeItems` to find leaf nodes by `id` and mutate `leaf.value = updatedValue`. Keep `openedNodes` ref untouched.
**Warning signs:** Tree collapses to root on every poll cycle.

### Pitfall 2: Hash URL Missing `/#` Prefix
**What goes wrong:** `window.open('/s7plus-tag-tree?db=...', '_blank')` opens a blank 404 page.
**Why it happens:** `createWebHashHistory` appends all routes as `/#/...`; without the `#` the server sees a real URL path it does not serve.
**How to avoid:** Always prefix: `/#/s7plus-tag-tree?db=...`. Confirmed in `DatablockBrowserPage.vue` line 87: `const url = \`/#/s7plus-tag-tree?db=...\``.
**Warning signs:** New tab shows blank AdminUI with no route match.

### Pitfall 3: protocolSourceBrowsePath Segment Count
**What goes wrong:** A tag at DB root level (one segment after stripping the DB name) has `remaining.length === 1` and IS a leaf. A DB name with a dot in it could cause over-splitting.
**Why it happens:** Simple `split('.')` on the browse path is ambiguous if DB names or tag names contain dots.
**How to avoid:** The DB name is always `segments[0]` (the first `.`-separated token). Strip it and trust the rest as the depth. Real S7+ DB names and tag names do not contain dots (TIA Portal convention). If needed, compare against the known `dbName` query param before splitting.
**Warning signs:** Tags appear nested deeper than expected, or root-level tags show as struct folders.

### Pitfall 4: Auth in New Tab — Resolved (Cookie-based)
**What goes wrong (feared):** New tab opens TagTreeBrowserPage unauthenticated.
**Why it was a concern:** STATE.md blocker note raised the question of JWT token storage location.
**Resolution (CONFIRMED):** AdminUI uses HTTP cookie auth set by `/Invoke/auth/signin`. Cookie is sent automatically to the same origin in a new tab — no token-copy mechanism needed. `localStorage` is only used for locale preference (`STORAGE_KEY` in `i18n.js`). There is no JWT in localStorage or sessionStorage.
**Warning signs:** Would only appear as 401 responses on all `/Invoke/auth/` calls in the new tab.

### Pitfall 5: v-treeview auto-activation Bug in Production (Vuetify 3.9+)
**What goes wrong:** In production builds, expanding a node may also mark it "activated" (selected highlight) automatically — even without a click.
**Why it happens:** Known Vuetify 3 issue (GitHub issue #21790); the `useNested` composable conflates open and active state in certain production tree reconcile paths.
**How to avoid:** This phase does not use activation (selection) at all — no `v-model:activated` binding. The bug is cosmetic only; it does not break value display or expand/collapse logic.
**Warning signs:** Nodes show blue highlight after expand, but no functional regression.

### Pitfall 6: touchS7PlusActiveTagRequests Requires Non-Empty Array
**What goes wrong:** Calling the touch endpoint with an empty array returns a 400 error.
**Why it happens:** Backend explicitly validates: `if (!Array.isArray(items) || items.length === 0)` → 400.
**How to avoid:** Guard the call: `if (expandedLeafTags.length > 0) { await touchTags(...) }`.
**Warning signs:** Console 400 errors on first load before any nodes are expanded.

### Pitfall 7: Missing i18n Key for Dashboard/Nav
**What goes wrong:** If `DashboardPage.vue` or any nav component references `$t('dashboard.tagTreeBrowser')` and the key is absent, it shows the raw key string.
**How to avoid:** Add `"tagTreeBrowser": "Tag Tree Browser"` under `dashboard` in `en.json`. Note: DashboardPage already has a `datablockBrowser` shortcut — TagTreeBrowser is NOT a top-level dashboard card (it is opened via DatablockBrowserPage row click or alarms viewer link). No new dashboard shortcut needed for Phase 15.

---

## Code Examples

### Tree Node Shape (internal data model)
```javascript
// Internal node object — NOT a realtimeData doc directly
const treeNode = {
  id: 'DBName.SubStruct',       // full path string — unique key
  name: 'SubStruct',            // display label (last segment)
  isLeaf: false,
  children: [
    {
      id: 'DBName.SubStruct.Tag',
      name: 'Tag',
      isLeaf: true,
      type: 'analog',           // from realtimeData.type
      value: 123.45,            // from realtimeData.value (mutated on refresh)
      address: '"DBName".SubStruct.Tag', // from realtimeData.protocolSourceObjectAddress
      children: [],
    }
  ]
}
```

### Tree Build Algorithm (client-side)
```javascript
// Build tree from flat docs array; call once on mount
function buildTree(docs, dbName) {
  const root = { id: dbName, name: dbName, isLeaf: false, children: [] }
  for (const doc of docs) {
    const browsePath = doc.protocolSourceBrowsePath  // e.g. "DBName.SubStruct.Tag"
    const segments = browsePath.split('.')
    // segments[0] is DB name; skip
    const remaining = segments.slice(1)
    if (remaining.length === 0) continue
    let current = root
    for (let i = 0; i < remaining.length; i++) {
      const seg = remaining[i]
      const fullId = segments.slice(0, i + 2).join('.')
      const isLast = i === remaining.length - 1
      let child = current.children.find(c => c.id === fullId)
      if (!child) {
        child = {
          id: fullId,
          name: seg,
          isLeaf: isLast,
          children: [],
          ...(isLast ? {
            type: doc.type,
            value: doc.value,
            address: doc.protocolSourceObjectAddress,
          } : {})
        }
        current.children.push(child)
      } else if (isLast) {
        // Update value if node already exists (shouldn't happen at build time)
        child.value = doc.value
      }
      current = child
    }
  }
  return root
}
```

### In-Place Value Refresh (called every 5 seconds)
```javascript
// Update leaf values without rebuilding tree; preserves openedNodes
function patchLeafValues(treeRoot, freshDocs) {
  const docMap = new Map(freshDocs.map(d => [d.protocolSourceBrowsePath, d.value]))
  function walk(node) {
    if (node.isLeaf) {
      const newVal = docMap.get(node.id)  // node.id === browsePath
      if (newVal !== undefined) node.value = newVal
    } else {
      for (const child of node.children) walk(child)
    }
  }
  walk(treeRoot)
}
```

### Get All Expanded Leaf Tags (for touchS7PlusActiveTagRequests)
```javascript
// Collect leaf nodes whose path is "under" an opened ancestor
function getExpandedLeafTags(treeRoot, openedSet) {
  const result = []
  function walk(node, parentOpen) {
    const isOpen = parentOpen || openedSet.has(node.id)
    if (node.isLeaf && parentOpen) {
      result.push({ address: node.address, browsePath: node.id })
    } else {
      for (const child of node.children) walk(child, isOpen)
    }
  }
  // Root's children are visible if root is considered "open" by the auto-expand on load
  for (const child of treeRoot.children) walk(child, openedSet.has(treeRoot.id))
  return result
}
```

### Auto-expand First Level (set openedNodes on mount)
```javascript
// D-01: auto-expand direct children of the DB root after tree is built
const openedNodes = ref(new Set([treeRoot.id]))
// This opens the root node, revealing first-level children immediately
```

### API Call: listS7PlusTagsForDb
```javascript
// Source: json-scada/src/server_realtime_auth/index.js lines 488-513
const res = await fetch(
  `/Invoke/auth/listS7PlusTagsForDb?connectionNumber=${connectionNumber}&dbName=${encodeURIComponent(dbName)}`
)
const docs = await res.json()
// docs is an array of full realtimeData documents
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Vuetify 2 `v-treeview` (v-slot:label) | Vuetify 3 `v-treeview` with `prepend`/`append` slots | Vuetify 3.x | Slot names changed; `item-children` is now a string prop, not a function |
| JWT in localStorage for cross-tab | HTTP cookie (implicit, same-origin) | This project from the start | No extra auth work needed for window.open |

**Deprecated/outdated:**
- Vuetify 2 `v-treeview` API (`:open.sync`, `v-slot:label`): replaced by `v-model:opened` and `#prepend`/`#append` scoped slots in Vuetify 3.

---

## Open Questions

1. **v-treeview `item-children` type in Vuetify 3.10**
   - What we know: In Vuetify 3 the `item-children` prop accepts a string (property name) to find children on each item, or a function for async lazy-loading.
   - What's unclear: Whether passing `children: []` (empty array) on a leaf node causes v-treeview to render an expansion arrow. If so, leaf nodes need `children` omitted or set to `undefined` rather than `[]`.
   - Recommendation: Test during implementation. If empty `children: []` shows an expand arrow on leaves, use `undefined` or omit the property entirely and set `item-children="children"` — v-treeview should treat missing property as no children.

2. **openedNodes type: Set vs. Array**
   - What we know: Vuetify 3 `v-model:opened` docs show it as an array. Vue's reactivity system tracks Set mutations in Vue 3.2+ but Vuetify may bind to array internally.
   - What's unclear: Whether `ref(new Set([...]))` or `ref([...])` is the correct type for `v-model:opened`.
   - Recommendation: Use `ref([dbName])` (array containing root node id) for `v-model:opened` to be safe; convert to Set for efficient lookup internally if needed.

---

## Environment Availability

Step 2.6: SKIPPED — this phase is purely frontend code changes. No new external tools, services, runtimes, CLIs, or databases are required. All backend API endpoints are confirmed present (Phase 13). The AdminUI dev/build toolchain (Node, Vite) was confirmed working in Phase 14.

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | None configured (no jest.config, no vitest.config, no test/ directory found) |
| Config file | none |
| Quick run command | `cd json-scada/src/AdminUI && npm run build` (build smoke test) |
| Full suite command | `cd json-scada/src/AdminUI && npm run build` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| TAGTREE-01 | Tree hierarchy built from protocolSourceBrowsePath | manual | navigate to `/#/s7plus-tag-tree?db=X&connectionNumber=N` and verify tree renders | N/A |
| TAGTREE-02 | Live value refresh every 5s + touchS7PlusActiveTagRequests called | manual | expand node, wait 5s, observe value update and network requests | N/A |
| TAGTREE-03 | Leaf nodes show `type` field | manual | inspect leaf node for type chip | N/A |
| TAGTREE-04 | Page accepts query params | manual | open via DatablockBrowserPage "Browse Tags" button | N/A |
| INTEGRATION-01 | originDbName is clickable link in alarms viewer | manual | observe link in S7PlusAlarmsViewerPage, click to open new tab | N/A |

### Sampling Rate
- **Per task commit:** `npm run build` — verify no compile errors
- **Per wave merge:** `npm run build` — full compile clean
- **Phase gate:** `npm run build` green + manual smoke test of tree render before `/gsd:verify-work`

### Wave 0 Gaps
- No unit test framework is configured in this project. All verification is build-time (TypeScript/Vue compile) + manual browser smoke test.
- None — existing infrastructure (build only) covers the phase.

---

## Project Constraints (from CLAUDE.md)

CLAUDE.md does not exist in the working directory. No additional project-level directives found. All constraints come from CONTEXT.md decisions (D-01 through D-06) and from the established codebase conventions observed in existing pages.

**Observed conventions (HIGH confidence — read directly from source):**
- Pages: `PascalCase.vue` in `src/components/`
- Script style: `<script setup>` with Composition API (`ref`, `watch`, `onMounted`, `onUnmounted`)
- Fetch: always `/Invoke/auth/...`; no Authorization header — relies on HTTP cookie
- Layout: `<v-container fluid>` wrapper
- Table density: `density="compact"` with `class="elevation-1"`
- Overflow toggle: `document.documentElement.style.overflowY = 'scroll'` in `onMounted`, `'hidden'` in `onUnmounted`
- New-tab navigation: `window.open('/#/route?params', '_blank')`
- No TypeScript — plain JavaScript `.vue` files

---

## Sources

### Primary (HIGH confidence)
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — setInterval refresh pattern, fetch pattern, template slot patterns
- `json-scada/src/AdminUI/src/components/DatablockBrowserPage.vue` — window.open hash URL construction, useRoute pattern, connection fetch
- `json-scada/src/AdminUI/src/router/index.js` — route registration pattern; confirmed `/s7plus-tag-tree` NOT yet registered
- `json-scada/src/AdminUI/src/components/DashboardPage.vue` — shortcut cards; confirmed `datablockBrowser` card already present, no TagTreeBrowser card needed
- `json-scada/src/server_realtime_auth/index.js` lines 488–560 — confirmed listS7PlusTagsForDb and touchS7PlusActiveTagRequests endpoints
- `json-scada/src/S7CommPlusClient/TagsCreation.cs` — confirmed `protocolSourceBrowsePath`, `protocolSourceObjectAddress`, `type`, `value` field names
- `json-scada/src/AdminUI/package.json` — confirmed Vuetify 3.10, @mdi/font 7.4.47, Vue Router 4.4.0

### Secondary (MEDIUM confidence)
- [Vuetify 3 v-treeview API](https://vuetifyjs.com/en/api/v-treeview) — v-model:opened, item-children prop, slot names
- [Vuetify 3 treeview component docs](https://vuetifyjs.com/en/components/treeview/) — open-strategy, lazy expand

### Tertiary (LOW confidence)
- [GitHub vuetifyjs/vuetify issue #21790](https://github.com/vuetifyjs/vuetify/issues/21790) — auto-activation bug in Vuetify 3.9+ production builds (cosmetic only)

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — read directly from package.json and existing source files
- Architecture: HIGH — based on direct reading of canonical reference files; tree algorithm derived from stated protocolSourceBrowsePath format
- v-treeview API details: MEDIUM — official docs confirmed component exists and v-model:opened is supported; slot names confirmed from Vuetify 3 docs
- Pitfalls: HIGH (pitfalls 1–4, 6–7 from direct code reading); MEDIUM (pitfall 5 from community issue report)

**Research date:** 2026-03-26
**Valid until:** 2026-04-25 (stable libraries, internal project code)

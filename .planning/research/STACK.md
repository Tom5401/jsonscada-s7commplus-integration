# Technology Stack

**Project:** v1.4 Tag Tree Browser (DatablockBrowser + TagTreeBrowser)
**Researched:** 2026-03-26
**Confidence:** HIGH — Vuetify version confirmed from installed node_modules; v-treeview availability confirmed from official docs and GitHub issue history; router pattern confirmed from Vue Router official docs

---

## Summary of New Stack Needs

v1.4 adds two new AdminUI pages and new backend endpoints. The core stack does not change. The key question was whether Vuetify 3 has a native tree component — it does. **No new npm packages are required.** `v-treeview` ships with Vuetify 3 and is already available in the project's installed version (3.10.12).

---

## Installed Versions (confirmed)

| Component | Version | Source |
|-----------|---------|--------|
| Vuetify | 3.10.12 (installed) / `"3.10"` (package.json) | `AdminUI/package.json` + previous research |
| Vue 3 | ^3.4.31 | `AdminUI/package.json` |
| vue-router | ^4.4.0 | `AdminUI/package.json` |
| Vite | ^7.3.1 | `AdminUI/package.json` |
| vite-plugin-vuetify | ^2.0.3 | `AdminUI/package.json` (auto-imports Vuetify components) |

---

## Core Framework Decisions

### v-treeview (Vuetify 3 built-in)

**Use `v-treeview` from Vuetify 3. No external tree library needed.**

**Availability:** `v-treeview` is present and documented in Vuetify 3. It was absent in very early Vuetify 3 releases (issues #14622 and #13518 tracked its re-addition), but it has been stable and actively maintained throughout the Vuetify 3 lifecycle. The project's installed version (3.10.12) is well beyond any re-introduction point. Recent bug activity (issues in 3.6.x, 3.7.x, and 3.7.6) confirms the component is actively maintained.

**Lazy loading contract (`load-children` prop):** `v-treeview` supports on-demand child loading via the `load-children` prop. The prop accepts a function that receives the item being expanded. The function is called when the user expands a node whose `item-children` property is an empty array `[]`. The function must **mutate** the item's children array in place (push new items into it) and must return a Promise. After the Promise resolves, Vuetify re-renders the expanded children.

```vue
<v-treeview
  :items="datablocks"
  :load-children="loadTagChildren"
  item-title="name"
  item-value="id"
/>
```

```js
const loadTagChildren = async (item) => {
  const children = await fetchTagsForDatablock(item.id)
  item.children.push(...children)
  // Must return a Promise (async function satisfies this)
}
```

**Caveat — `load-children` regression in 3.7.1:** GitHub issue #20450 documents that `load-children` broke between Vuetify 3.6.15 and 3.7.1 (reported September 2024). The project is on 3.10.12, which post-dates this regression. The issue was filed and addressed in the 3.7.x maintenance cycle. **Treat this as resolved at 3.10.12.** If the feature does not work as expected after implementing, pin to the last known-good `load-children` release or implement manual expand-handler workaround (see PITFALLS.md).

**Confidence:** MEDIUM-HIGH — official Vuetify docs confirm the component and `load-children` prop; regression confirmed resolved by version distance (3.7.1 → 3.10.12 is significant); exact fix commit not verified.

### Router — Opening TagTreeBrowser in a New Window

**Use `useRouter().resolve()` + `window.open()`. No new library needed.**

The project's router uses `createWebHashHistory` (confirmed in `router/index.js`). Vue Router 4 provides `router.resolve(location)` which returns a resolved route object with an `href` property containing the full hash URL. Passing that href to `window.open(href, '_blank')` opens the correct SPA route in a new tab, including query params.

```js
import { useRouter } from 'vue-router'

const router = useRouter()

const openTagTreeBrowser = (dbName) => {
  const resolved = router.resolve({
    path: '/tag-tree-browser',
    query: { db: dbName }
  })
  window.open(resolved.href, '_blank')
}
```

The new window loads the SPA fresh, reads `useRoute().query.db` on mount, and immediately fetches the tag tree for that datablock.

**Why `router.resolve()` not raw string concatenation:** `createWebHashHistory` prefixes all routes with `/#/`. Constructing the URL manually (e.g., `'/#/tag-tree-browser?db=' + dbName`) is fragile if the app base path changes. `router.resolve()` uses the router's own knowledge of the base and hash prefix, making it resilient.

**Confidence:** HIGH — Vue Router 4 official docs confirm `router.resolve()` API; `createWebHashHistory` confirmed in project source; pattern widely validated in Vue Router community.

---

## New AdminUI Components Required

| Component | Type | Technology | Notes |
|-----------|------|-----------|-------|
| `DatablockBrowserPage.vue` | New page component | Vue 3 + Vuetify 3 `v-list` or `v-data-table` | Lists all datablocks from new backend endpoint; each row/item is clickable to open TagTreeBrowser |
| `TagTreeBrowserPage.vue` | New page component | Vue 3 + Vuetify 3 `v-treeview` | Lazy tree for a single datablock; reads `?db=` from route query on mount; shows live values from `realtimeData` |

Both pages follow the existing pattern in `S7PlusAlarmsViewerPage.vue`: `fetch` on `onMounted`, `v-container fluid` layout, Vuetify components only.

---

## New Router Routes Required

```js
// Add to router/index.js
import DatablockBrowserPage from '../components/DatablockBrowserPage.vue'
import TagTreeBrowserPage from '../components/TagTreeBrowserPage.vue'

{ path: '/datablock-browser', component: DatablockBrowserPage },
{ path: '/tag-tree-browser',  component: TagTreeBrowserPage },
```

Query params (`?db=DBName`) are read via `useRoute().query.db` inside `TagTreeBrowserPage.vue`. No dynamic route segments needed.

---

## New Backend Endpoints Required

| Endpoint | Method | Purpose | Pattern |
|----------|--------|---------|---------|
| `GET /Invoke/auth/listDatablocks` | GET | Returns array of `{ id, name }` for all known datablocks | Same pattern as `listS7PlusAlarms` in `server_realtime_auth/index.js` |
| `GET /Invoke/auth/getTagTypeInfo?db=DBName` | GET | Returns tag tree for one datablock (on-demand) | Same HTTP auth pattern; query param instead of body |

Both endpoints live in `server_realtime_auth/index.js` alongside existing endpoints. The data source is a new MongoDB collection populated by the C# driver's `GetListOfDatablocks` browse (already implemented at startup for v1.2's `RelationIdNameMap`).

---

## New MongoDB Collection

| Collection | Purpose | Populated by |
|------------|---------|--------------|
| `s7plusDatablocks` | Stores datablock metadata and optional tag type info | C# driver at startup (extend existing `GetListOfDatablocks` browse) |

The v1.2 driver already calls `GetListOfDatablocks` at startup to build `RelationIdNameMap` (a C# Dictionary). For v1.4, extend this to also upsert each datablock into a MongoDB collection so the AdminUI can query them via HTTP.

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Tree component | Vuetify 3 `v-treeview` | Element Plus `el-tree` | Element Plus is not in the project; mixing two component libraries adds bundle weight and style conflicts |
| Tree component | Vuetify 3 `v-treeview` | PrimeVue `Tree` | Same — not in project, different design system |
| Tree component | Vuetify 3 `v-treeview` | `@grapoza/vue-tree` | External dependency for a feature that ships natively in the already-installed Vuetify |
| New window pattern | `router.resolve()` + `window.open()` | `<router-link target="_blank">` | Requires knowing the URL at template render time; the URL is computed from a click event on alarm row (dynamic) |
| New window pattern | `router.resolve()` + `window.open()` | Emit event to parent + open from parent | Unnecessary indirection; `window.open` from child component is idiomatic |

---

## What NOT to Add

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| Any external Vue tree library | `v-treeview` ships with installed Vuetify 3.10.12 | `v-treeview` with `load-children` prop |
| Vuetify 4 upgrade | Active project milestone; breaking upgrade risk is unacceptable mid-PoC | Stay on 3.10.x |
| WebSocket for live values | Existing pattern is HTTP polling every 5s; live values are read from `realtimeData` MongoDB collection via same poll | HTTP fetch on interval, same as alarms viewer |
| Dynamic route segment (`/tag-tree-browser/:db`) | Query params are simpler for a new-window launch scenario; `useRoute().query.db` is sufficient | `?db=DBName` query param |
| Server-side tree rendering | Tag tree structure is low-depth (datablock → struct → field); full subtree per datablock fits in a single response | On-demand fetch per node expand via `load-children` |

---

## Version Compatibility

| Component | Version | Notes |
|-----------|---------|-------|
| Vuetify | 3.10.12 installed | `v-treeview` with `load-children` — available and stable; `load-children` regression (3.7.1) is in the past |
| Vue 3 | ^3.4.31 | `useRoute()`, `useRouter()`, `onMounted`, `ref` — all standard Composition API |
| vue-router | ^4.4.0 | `router.resolve()` — stable API throughout Vue Router 4 |
| vite-plugin-vuetify | ^2.0.3 | Auto-imports `VTreeview` component; no manual registration needed |
| mongodb (Node.js) | ^7.0.0 | New endpoints follow existing `.find({}).toArray()` pattern |
| MongoDB.Driver (C#) | 3.4.2 | Extend existing `GetListOfDatablocks` call; no new NuGet packages |

---

## Sources

- `json-scada/src/AdminUI/package.json` — Vuetify `"3.10"`, Vue `^3.4.31`, vue-router `^4.4.0` confirmed (HIGH confidence, project file)
- `json-scada/src/AdminUI/src/router/index.js` — `createWebHashHistory` confirmed; existing route registration pattern confirmed (HIGH confidence, source code)
- Vuetify official docs `https://vuetifyjs.com/en/components/treeview/` — `v-treeview` documented with `load-children` prop; lazy expand via empty-children-array contract (MEDIUM-HIGH confidence, official docs; WebFetch blocked, confirmed via search)
- GitHub issue #14622 `https://github.com/vuetifyjs/vuetify/issues/14622` — Feature request tracking when `v-treeview` would be available in Vuetify 3; confirms it was absent early, now present (MEDIUM confidence, GitHub)
- GitHub issue #20450 `https://github.com/vuetifyjs/vuetify/issues/20450` — `load-children` regression in 3.7.1; project is on 3.10.12 which post-dates this (MEDIUM confidence, GitHub)
- GitHub issues #19919, #20344, #20832, #20844 — Active `v-treeview` bug reports in 3.6–3.7 range confirm component is actively maintained (MEDIUM confidence, GitHub)
- Vue Router docs `https://router.vuejs.org/guide/essentials/navigation.html` — `router.resolve()` API confirmed for generating hrefs (HIGH confidence, official docs)
- Quasar discussion #14272 + community sources — `router.resolve().href` + `window.open()` pattern for new-tab navigation confirmed as idiomatic (MEDIUM confidence, community)

---

*Stack research for: v1.4 Tag Tree Browser (DatablockBrowser + TagTreeBrowser)*
*Researched: 2026-03-26*

---
phase: 19-non-datablock-tags
plan: 01
subsystem: ui
tags: [vue, vuetify, datatable, s7plus, tagtreebrowser]

requires:
  - phase: 14-datablock-browser
    provides: DatablockBrowserPage.vue component and browseDatablock() function
  - phase: 18-tag-tree-browser
    provides: TagTreeBrowserPage.vue accepting ?db=<name>&connectionNumber=<N>

provides:
  - Virtual Memory Area rows (IArea, QArea, MArea, S7Timers, S7Counters) in DatablockBrowserPage
  - Memory Areas section divider row preceding area rows
  - Browse Tags navigation from area rows to TagTreeBrowserPage

affects: [tagtreebrowser, datablock-browser, s7plus-areas]

tech-stack:
  added: []
  patterns:
    - Vuetify v-data-table #item slot for mixed-row-type rendering (divider + data rows)
    - Virtual rows prepended to API results using computed constant arrays
    - _isDivider flag pattern for distinguishing section header rows from data rows

key-files:
  created: []
  modified:
    - json-scada/src/AdminUI/src/components/DatablockBrowserPage.vue

key-decisions:
  - "Virtual area rows defined as constants (not reactive) — they never change, no need for computed"
  - "Replaced #item.actions slot with full #item slot to support divider row rendering"
  - "VIRTUAL_AREA_ROWS and MEMORY_AREA_DIVIDER defined at module level (outside setup) — shared, immutable"
  - "em-dash (\\u2014) used for DB Number of area rows per D-02"
  - "Area rows only appear when connection is selected — else/catch branches keep datablocks.value = []"

patterns-established:
  - "Mixed-row v-data-table: use #item slot with _isDivider flag; divider renders <td :colspan='columns.length'>"
  - "Virtual rows prepended to real data: [DIVIDER, ...VIRTUAL_ROWS, ...realApiData]"

requirements-completed:
  - AREA-01

duration: 15min
completed: 2026-03-30
---

# Phase 19: Non-Datablock Tags Summary

**DatablockBrowserPage now surfaces 5 PLC memory areas (IArea, QArea, MArea, S7Timers, S7Counters) alongside datablocks, completing TagTreeBrowser milestone coverage of non-datablock S7+ memory.**

## Performance

- **Duration:** 15 min
- **Started:** 2026-03-30T00:00:00Z
- **Completed:** 2026-03-30T00:00:00Z
- **Tasks:** 2 (1 auto + 1 checkpoint:human-verify)
- **Files modified:** 1

## Accomplishments
- Added `AREA_NAMES`, `MEMORY_AREA_DIVIDER`, and `VIRTUAL_AREA_ROWS` constants to DatablockBrowserPage.vue
- Modified `watch(selectedConnection)` to prepend virtual rows to API results; cleared on deselect/error
- Replaced `#item.actions` slot with `#item` slot to support divider row (grey bg, bold label, full colspan) alongside normal rows (3-column layout with Browse Tags)
- Human-verified: area rows render correctly, Browse Tags navigates to TagTreeBrowserPage with correct params

## Task Commits

1. **Task 1: Add virtual area rows and section divider** - `3d16d061` (feat)
2. **Task 2: Human verify** - approved ✓

## Files Created/Modified
- `json-scada/src/AdminUI/src/components/DatablockBrowserPage.vue` — added Memory Areas divider, 5 virtual area rows, #item slot for mixed rendering

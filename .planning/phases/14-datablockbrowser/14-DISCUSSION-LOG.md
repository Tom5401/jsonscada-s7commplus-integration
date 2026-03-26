# Phase 14: DatablockBrowser - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-26
**Phase:** 14-datablockbrowser
**Areas discussed:** Connection selector default, TagTreeBrowser URL path, Row action UX

---

## Connection Selector Default

| Option | Description | Selected |
|--------|-------------|----------|
| Auto-select first (Recommended) | Most installs have 1 PLC — load datablocks immediately, no manual step | |
| Placeholder — manual pick | Show empty state until operator picks a connection explicitly | ✓ |
| URL param + auto-select fallback | ?conn=N pre-fills on load; auto-selects first if no param — enables deep linking | |

**User's choice:** Placeholder — manual pick
**Notes:** Operator must explicitly choose a connection. Avoids silently showing data for the wrong PLC.

---

## TagTreeBrowser URL Path

| Option | Description | Selected |
|--------|-------------|----------|
| /s7plus-tag-tree (Recommended) | Follows /s7plus-alarms convention. Query: ?db=encodeURIComponent(db_name)&connectionNumber=N | ✓ |
| /s7plus-datablock-tags | More explicit path. Same query params. | |

**User's choice:** `/s7plus-tag-tree`
**Notes:** Agreed to use `encodeURIComponent(db_name)` for the query param. Vue Router decodes on `route.query.db` access in Phase 15.

---

## Row Action UX

| Option | Description | Selected |
|--------|-------------|----------|
| Explicit "Browse Tags" button (Recommended) | Per-row button in an Actions column — consistent with S7PlusAlarmsViewerPage, discoverable | ✓ |
| Whole-row click | Entire row is clickable, cursor:pointer + hover highlight — no extra column, minimal UI | |
| Both (row click + button) | Row click AND a Browse Tags button per row | |

**User's choice:** Explicit "Browse Tags" button
**Notes:** Consistent with per-row button pattern established in S7PlusAlarmsViewerPage (Ack, Delete buttons).

---

## Claude's Discretion

- Loading/empty state handling: follow S7PlusAlarmsViewerPage patterns
- How to populate connection selector options: researcher to investigate (existing endpoint or derive from datablock list)
- Column layout details beyond `db_name` and `db_number`

## Deferred Ideas

None raised during discussion.

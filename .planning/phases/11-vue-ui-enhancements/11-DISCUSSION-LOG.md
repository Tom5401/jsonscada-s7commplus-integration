# Phase 11: Vue UI Enhancements - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-25
**Phase:** 11-vue-ui-enhancements
**Areas discussed:** Ack indicator, Ack All button, Priority default sort, Column layout

---

## Ack Indicator (VIEWER-03)

| Option | Description | Selected |
|--------|-------------|----------|
| Icon in new column | mdi-bell / mdi-information icon column | |
| Chip in new column | 'Ack req' / 'Info only' v-chip column | |
| Inline in Acknowledge column | Extend existing column | ✓ |

**User's choice:** Inline in existing Acknowledge column. `isAcknowledgeable === false` → dash; `isAcknowledgeable === true` → existing Ack/spinner/checkmark behavior unchanged.
**Notes:** User specified exact behavior: false = dash, true = same as current. No new column needed.

---

## Ack All Button (VIEWER-05)

| Option | Description | Selected |
|--------|-------------|----------|
| Filter row, next to Delete Filtered | Same v-row as filters and Delete Filtered | ✓ |
| Above table, separate row | Dedicated bulk actions row | |

**User's choice:** Filter row, next to Delete Filtered.
**Notes:** No additional clarifications.

---

## Priority Default Sort (VIEWER-02)

| Option | Description | Selected |
|--------|-------------|----------|
| Pre-sorted descending (highest first) | Table loads with highest priority at top | |
| Unsorted on load (user clicks to sort) | Natural API order on load, click to sort | ✓ |

**User's choice:** Unsorted on load — user clicks Priority column header to sort.
**Notes:** Keeps current default behavior, no `:sort-by` prop needed.

---

## Column Layout

| Option | Description | Selected |
|--------|-------------|----------|
| Source \| Timestamp \| Priority \| Status \| Acknowledge \| Delete \| ... | Identity+when+criticality first | ✓ |
| Timestamp \| Source \| Priority \| Status \| Acknowledge \| Delete \| ... | Chronological focus first | |
| Keep current order, append Priority at end | Minimal disruption | |

**User's choice:** Source \| Timestamp \| Priority \| Status \| Acknowledge \| Delete \| Alarm class \| Event text \| ID \| Origin DB Name \| DB Number \| Additional text 1/2/3.
**Notes:** No additional clarifications.

---

## Claude's Discretion

- Exact confirmation dialog wording for Ack All
- `formatTimestamp()` implementation (manual Date construction)

## Deferred Ideas

None.

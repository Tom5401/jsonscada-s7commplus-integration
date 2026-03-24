# Phase 8: Frontend — Delete Buttons + Origin Columns - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-24
**Phase:** 08-frontend-delete-buttons-origin-columns
**Areas discussed:** Origin column placement, Actions column design, Confirmation strategy, Error feedback

---

## Origin Column Placement

| Option | Description | Selected |
|--------|-------------|----------|
| After 'ID' column | Group origin data near cpuAlarmId — all identifiers together | ✓ |
| At the end of the table | After Additional text columns — less disruptive | |
| At the start (after Source) | Surfaces origin prominently but breaks reading order | |

**User's choice:** After 'ID' column
**Notes:** None

---

| Option | Description | Selected |
|--------|-------------|----------|
| Show RelationId as-is | Satisfies ORIGIN-08; useful for debugging | |
| Hide RelationId | Show only DB Name + DB Number; saves width | ✓ |

**User's choice:** Hide RelationId
**Notes:** Raw 64-bit integer has no operator value. ORIGIN-08 intentionally dropped.

---

## Actions Column Design

| Option | Description | Selected |
|--------|-------------|----------|
| New 'Actions' column at end | Dedicated column at far right with trash icon | |
| Inline with existing ack column | Delete icon next to ack icon — saves a column | |
| Leftmost column | Actions at start of row | |
| Other | Right of ack column, own separate column | ✓ |

**User's choice:** Delete column to the right of the Ack column, its own separate column
**Notes:** User specified exact placement verbatim.

---

| Option | Description | Selected |
|--------|-------------|----------|
| Trash icon button | mdi-delete, compact, consistent with icon pattern | ✓ |
| Text button 'Delete' | More explicit, more horizontal space | |
| Small red chip/button | Visually distinct but heavier | |

**User's choice:** Trash icon button

---

| Option | Description | Selected |
|--------|-------------|----------|
| Same row as filters | Third v-col in existing v-row | ✓ |
| Separate row below filters | Second v-row | |
| Above the data table only | Standalone row above table | |

**User's choice:** Same row as filters

---

## Confirmation Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| No confirm for normal rows | Only Coming+unacked gets warning | ✓ |
| Always confirm | Dialog for every single-row delete | |

**User's choice:** No confirm for normal rows

---

| Option | Description | Selected |
|--------|-------------|----------|
| Always confirm bulk delete | Confirmation dialog before bulk call | ✓ |
| No confirmation, just delete | Immediate bulk delete on click | |

**User's choice:** Always confirm bulk delete

---

| Option | Description | Selected |
|--------|-------------|----------|
| Warning + Delete/Cancel | Warning dialog with "Delete anyway" (red) + Cancel | ✓ |
| Requires extra ack check | Block delete entirely for Coming+unacked | |

**User's choice:** Warning dialog — "This alarm is still active on the PLC" with Delete anyway + Cancel

---

## Error Feedback on Failed Delete

| Option | Description | Selected |
|--------|-------------|----------|
| Show snackbar + roll back row | Re-insert row + v-snackbar error message | |
| Roll back silently | Re-insert row without message | |
| Let 5s refresh restore it | No rollback — auto-refresh handles it | ✓ |

**User's choice:** Let 5s refresh restore it
**Notes:** Keeps component simple; failed deletes are self-healing.

---

## Claude's Discretion

- Dialog implementation approach (v-dialog vs reactive state object)
- Bulk delete body shape when both filters are "All" (send all visible `_id`s)
- Column widths / alignment

## Deferred Ideas

- RelationId column (ORIGIN-08) — explicitly dropped by user decision

# Phase 19: Non-Datablock Tags - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-30
**Phase:** 19 — Non-Datablock Tags

---

## Areas Discussed

### Area Selection

**Question:** Which areas do you want to discuss for Non-Datablock Tags?
**Selected:** Both (db_number column display + visual separation)

---

### Area 1: db_number column for virtual area rows

**Question:** What should appear in the `db_number` cell for area rows?

**Options presented:**
- `—` (dash/em-dash) — explicit "not applicable" signal
- Leave blank (empty string)
- Area abbreviation (e.g., `M`, `I`, `Q`)
- You decide

**User selected:** `—` (dash/em-dash)

**Decision recorded:** D-02 — `db_number` cell shows `—` for all 5 virtual area rows.

---

### Area 2: Visual separation of area rows

**Question:** Should operators be able to tell at a glance that the 5 area rows are a different category from real datablocks?

**Options presented:**
- No separation — identical appearance (db_number dash is the only difference)
- Section divider row — a non-data row with label "Memory Areas" above the 5 area rows
- Subtle background tint — area rows get a light grey row background
- You decide

**User selected:** Section divider row

**Follow-up clarification:** User additionally specified that area rows should appear at the **top of the first page**, before real datablocks — overriding the ROADMAP's "append after" wording.

**Decision recorded:** D-01 — Area rows at top, preceded by "Memory Areas" section divider row.

---

## Decisions Summary

| ID | Decision | Source |
|----|----------|--------|
| D-01 | Virtual area rows at top of table; "Memory Areas" section divider precedes them | User |
| D-02 | `db_number` cell shows `—` for all 5 area rows | User |
| D-03 | Area names: IArea, QArea, MArea, S7Timers, S7Counters | Agent discretion |
| D-04 | Divider row mechanism: agent's choice (prepend to array or slot) | Agent discretion |
| D-05 | Re-uses existing `browseDatablock()` for area rows | Agent discretion |

# Phase 9: Driver Enrichment - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Session:** 2026-03-25
**Mode:** Interactive discuss-phase

---

## Area: Gray Area Selection

**Q:** Which areas do you want to discuss for Phase 9: Driver Enrichment?

**Options presented:**
- Field placement
- Log message update
- Nothing — looks clear

**Selected:** Field placement, Log message update + freeform: "I want to be able to set per alarmClass if alarms from that alarmClass are acknowledgeable. For example, 37 and 39 are also acknowledgeable."

---

## Area: AckClasses

**Q:** How should the set of acknowledgeable alarm classes be defined?

**Options presented:**
1. Hardcoded set in C# (Recommended) — HashSet near AlarmClassNames, recompile to change
2. Config file (json-scada.json) — read from connection config at startup
3. Use alarmClassName name match — check class name string keywords

**Selected:** Hardcoded, defined near AlarmClassNames (lines 194–201). User specified: "If possible I would like to define it in the same function."

**Rationale:** PoC simplicity wins. Config file adds parsing complexity not warranted for a PoC. User also identified the specific location (near existing class lookup structure).

**Decision recorded:** `static readonly HashSet<ushort> AcknowledgeableClasses = { 33, 37, 39 }` alongside `AlarmClassNames`.

---

## Area: Field Placement

**Q:** Where should `isAcknowledgeable` be placed in the BsonDocument?

**Options presented:**
1. Near alarmClass/alarmClassName (Recommended)
2. Near ackState

**Selected:** Near alarmClass/alarmClassName (Recommended)

**Rationale:** `isAcknowledgeable` is derived from `alarmClass`, so grouping them makes the derivation relationship clear.

---

## Area: Log Message

**Q:** Should the alarm-written log line show resolved or raw alarmText after DRIVER-02?

**Options presented:**
1. Yes — log resolved text (Recommended)
2. No — keep raw text

**Selected:** No — keep raw text

**Rationale:** Raw text shows what came from the PLC, which is useful for protocol-level debugging. The resolved text is what gets stored in MongoDB and can be inspected there.

---

*Log generated: 2026-03-25*

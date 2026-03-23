# Phase 4: Ack Write-Back - Context

**Gathered:** 2026-03-19
**Status:** Ready for planning

<domain>
## Phase Boundary

Close the full acknowledgement loop: an operator clicks Acknowledge on an unacknowledged alarm row in the S7Plus Alarms Viewer, the command travels Vue → Express → commandsQueue → C# driver → PLC, and the PLC applies the ack. MongoDB subsequently reflects `ackState: true` for that alarm.

The Wireshark spike (capturing TIA Portal's actual ack PDU) is the first deliverable of this phase and gates all implementation — no ack send code before the spike doc and prototype C# call exist.

</domain>

<decisions>
## Implementation Decisions

### Spike deliverable (gates implementation)
- Produce a decoded fields doc committed to `.planning/phases/04-ack-write-back/` — file must exist before any pipeline implementation begins
- Doc must contain: decoded `SetVariableRequest` fields (alarm object ID, attribute ID, value encoding) **plus** a prototype C# method that constructs and sends the PDU (not yet wired to commandsQueue)
- The prototype proves the PDU works against a live PLC before the full pipeline is wired up

### Acknowledge button UX
- The `Acknowledge` column in `S7PlusAlarmsViewerPage.vue` replaces the `mdi-close` (red) icon with a small `Ack` button for unacknowledged rows (`ackState: false`)
- Button: `v-btn size="x-small" variant="tonal"` — compact, fits the table cell
- Acknowledged rows (`ackState: true`) keep their existing `mdi-check` (green) icon — no change
- **Pending state**: spinner (`v-progress-circular` small) replaces button text; button is disabled to prevent duplicate sends
- **On failure** (POST error or PLC rejects): re-enable the `Ack` button; log `console.warn` only — no toast, no extra infrastructure (PoC)

### Post-ack confirmation
- Button stays in spinner pending state until the regular `setInterval` poll (5000ms) fires and `fetchAlarms()` returns a document with `ackState: true` for that alarm
- **If poll returns but `ackState` is still `false`**: exit pending state and re-enable the `Ack` button — operator can retry
- No targeted re-fetch or optimistic update — wait for the next poll cycle (simple, consistent with existing refresh pattern)

### commandsQueue document shape for alarm ack
- Use ASDU type `"s7plus-alarm-ack"` to distinguish alarm ack commands from tag writes in the same `commandsQueue` collection — no new collection, no schema changes
- `protocolSourceObjectAddress` carries the `cpuAlarmId` of the alarm to acknowledge
- `protocolSourceConnectionNumber` carries the PLC connection number (same as tag write commands)
- The C# Change Stream handler in `MongoCommands.cs` branches on `protocolSourceASDU == "s7plus-alarm-ack"` before executing the ack PDU send
- The Express POST endpoint inserts the command following the existing `commandsQueue` insertOne pattern (see integration points below)

### Claude's Discretion
- Internal structure of the ack branch in `MongoCommands.cs` (new method vs inline branch)
- Exact field names for any additional alarm-ack-specific data in the queue document beyond cpuAlarmId and connectionNumber
- Pending state tracking mechanism in the Vue component (per-row Map, Set, or reactive field on alarm object)
- Express endpoint path for the ack POST request (follow existing `/Invoke/auth/` pattern)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements
- `.planning/REQUIREMENTS.md` — DRVR-03 and VIEW-03 definitions and acceptance criteria

### Spike gate
- `.planning/phases/04-ack-write-back/` — Spike decoded-fields doc (does not exist yet; must be created as first plan deliverable before implementation plans execute)

### Vue component to modify
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — existing 140-line component; `ackState` column template and `fetchAlarms` function are the integration points

### commandsQueue pattern to follow
- `json-scada/src/server_realtime_auth/index.js` lines ~1135–1155 — `db.collection(COLL_COMMANDS).insertOne(...)` pattern, field names, and BSON types to mirror for the new ack POST endpoint
- `json-scada/src/S7CommPlusClient/MongoCommands.cs` — existing Change Stream handler; ack branch added here, branching on `protocolSourceASDU == "s7plus-alarm-ack"`

### Prior phase context
- `.planning/phases/03-read-only-alarm-viewer/03-CONTEXT.md` — established patterns: inline endpoint in index.js AUTHENTICATION block, `projection: {_id: 0}`, `Array.isArray` guard, `setInterval` refresh lifecycle

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `S7PlusAlarmsViewerPage.vue` — `fetchAlarms()` and `setInterval` at 5000ms are already in place; ack confirmation piggybacks on this loop without modification
- `commandsQueue` / `MongoCommands.cs` Change Stream — the insert→watch→execute pattern is fully wired; add an `if (asdu == "s7plus-alarm-ack")` branch before the existing tag write logic
- `v-btn`, `v-progress-circular` (Vuetify) — both available in AdminUI; no new dependencies

### Established Patterns
- `setInterval` refresh owns the alarm list state — pending state is tracked per-row locally (e.g. a `Set<cpuAlarmId>` of pending IDs); the row's template renders button vs. spinner based on membership
- `fetch('/Invoke/auth/...')` POST pattern: body as JSON, JWT auth header — same as any other admin action in the SPA
- `protocolSourceASDU` as a string type discriminator: already used for `"bool"`, `"real"`, etc. — `"s7plus-alarm-ack"` follows the same convention

### Integration Points
- `S7PlusAlarmsViewerPage.vue`: `item.ackState` template slot — add `v-btn` / `v-progress-circular` for `!item.ackState` case; keep `v-icon mdi-check` for `item.ackState === true`
- `server_realtime_auth/index.js` AUTHENTICATION block: add `POST /Invoke/auth/ackS7PlusAlarm` endpoint — inserts into `commandsQueue` with `protocolSourceASDU: "s7plus-alarm-ack"` and `protocolSourceObjectAddress: cpuAlarmId`
- `MongoCommands.cs` `ProcessMongoCmd()`: add branch inside the Change Stream `ForEachAsync` lambda — detect `"s7plus-alarm-ack"` ASDU, extract `cpuAlarmId`, build and send the ack PDU using the prototype method from the spike

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches within the decisions above

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 04-ack-write-back*
*Context gathered: 2026-03-19*

# Project Research Summary

**Project:** S7CommPlus Alarm PoC — v1.1
**Domain:** SCADA alarm management — S7CommPlus protocol, C#/.NET driver, Vue 3 frontend
**Researched:** 2026-03-18
**Confidence:** HIGH (all findings grounded in direct codebase inspection and protocol analysis)

## Executive Summary

This project extends the validated v1.0 S7CommPlus alarm subscription driver with three targeted improvements: fixing a known `ackState` bug that makes every alarm appear acknowledged, resolving alarm class names from protocol-fixed numeric IDs, and delivering a TIA Portal-style alarm viewer in the AdminUI with per-row acknowledge capability. The entire stack — .NET 8, MongoDB.Driver 3.4.2, Node.js/Express, Vue 3 + Vuetify 3 — is already in place. No new NuGet packages and no new npm packages are required. The implementation is purely additive: new components are introduced alongside existing ones, and the existing tag-based alarm viewer (`AlarmsViewerPage.vue`) must not be touched.

The recommended build order follows a strict data-correctness-first principle. Driver-level fixes (`ackState` sentinel, alarm class name resolution) must land before the viewer is built, because the UI has nothing meaningful to show if `ackState` is always `true` and alarm class is a raw integer. The read-only viewer can be built and validated end-to-end as soon as driver fixes and two new REST endpoints are in place. The ack write-back to the PLC is the highest-risk item in the project and is explicitly gated on a Wireshark capture of TIA Portal performing an ack command — this is a hard prerequisite because the S7CommPlus ack PDU format is undocumented and a malformed PDU can crash the alarm subscription thread.

The single unresolved research gap is the exact `SetVariableRequest` payload for the ack command. Everything else — component boundaries, data flow, field mappings, Vue routing patterns, MongoDB update strategy — is confirmed from direct code inspection at HIGH confidence. The ack write-back phase must be planned with a Wireshark spike as its first deliverable before any implementation begins.

---

## Key Findings

### Recommended Stack

The existing stack covers all v1.1 requirements without additions. On the C# side, `AlarmThread.cs` and `Common.cs` receive targeted modifications, and `MongoCommands.cs` gets one new method and one new ASDU routing branch. The Node.js/Express `server_realtime_auth` server gets two new routes using the same `authJwt` middleware already applied to all protected endpoints. The Vue 3 frontend gets one new component (`S7PlusAlarmsViewerPage.vue`) and one new router entry; `v-data-table` and the routing infrastructure are already installed.

**Core technologies:**
- .NET 8 / C# — driver runtime — already in use; no version change
- MongoDB.Driver 3.4.2 — persistence — already pinned; `UpdateOne` (not `InsertOne`) for ack confirmation writes
- Node.js/Express (`server_realtime_auth`) — API layer — two new routes: `GET /api/s7plus-alarms` and `POST /api/s7plus-alarms/ack`
- Vue 3 ^3.4.31 + Vuetify 3.10 — frontend — already installed in AdminUI; `v-data-table` handles the alarm grid natively
- `fetch()` + `setInterval(5000)` — alarm polling — no WebSocket or SSE infrastructure needed for the viewer PoC

**What NOT to use:** Pinia/Vuex (overkill for one page), WebSockets/GraphQL subscriptions (not justified for 5s polling), new MongoDB collections (extend `commandsQueue` with `protocolSourceASDU: "ALARM_ACK"`).

### Expected Features

All v1.0 alarm data fields (`cpuAlarmId`, `alarmState`, `plcTimestamp`, `alarmText`, `additionalTexts`, `ackState`, `alarmClass`, etc.) are already stored in `s7plusAlarmEvents`. v1.1 adds one new field (`alarmClassName` string) and fixes one existing field (`ackState` boolean).

**Must have (table stakes — P1):**
- Fix `ackState` always-true bug — core data correctness; blocks every ack-related feature downstream
- Add `alarmClassName` string field — operators cannot interpret raw numeric alarm class IDs; static dictionary maps protocol-fixed IDs 1, 2, 256–272
- `S7PlusAlarmsViewerPage.vue` with TIA Portal-style columns — the primary deliverable of v1.1
- TIA Portal column set: Source, Date, Time, Status, Acknowledge, Alarm class name, Event text, ID, Additional text 1–3

**Should have (differentiators — P2):**
- Ack write-back to PLC via `commandsQueue` — closes the loop; HIGH complexity gated on Wireshark spike
- Acknowledge button in viewer — enabled only for `ackState: false`; depends on ack write-back API being functional
- Auto-refresh (setInterval 5s), pause during user interaction

**Defer (P3 / v2+):**
- Status and alarm class client-side filters (low cost but not essential for PoC)
- Vuetify built-in pagination
- "Receive alarms" PLC selector dropdown
- Multi-language alarm class names — protocol does not deliver custom TIA Portal display names; impossible without out-of-band configuration
- Alarm history export, alarm suppression/shelving, WebSocket real-time push

**ISA 18.2 display logic** — four states map to `alarmState` + `ackState` combinations: Unacknowledged Active / Acknowledged Active / Unacknowledged Cleared / Acknowledged Cleared.

**Feature dependency chain (critical ordering):**
- `ackState` bug fix is required by the ack button, the ISA 18.2 Acknowledge column, and ack write-back
- `alarmClassName` is required by the "Alarm class name" viewer column
- REST endpoints are required by the Vue component
- Ack write-back is required by the ack button — and is itself gated on a Wireshark spike

### Architecture Approach

The architecture is a layered SCADA pipeline: PLC → C# driver (alarm subscription) → MongoDB → Express REST → Vue viewer. All layers already exist; v1.1 adds targeted modifications at each layer without restructuring. The ack path is the reverse flow: Vue POST → Express → `commandsQueue` → C# `MongoCommands.cs` → `AlarmAckToPLC()` → PLC. Ack PDUs must be sent on a dedicated `alarmConn`, not the shared tag connection, to prevent PDU interleaving with the tag read/write cycle.

**Major components:**
1. `AlarmThread.cs` (MODIFIED) — fix ackState sentinel; add `alarmClassName` lookup; add ack-only notification handler that issues `UpdateOne` on `cpuAlarmId`
2. `Common.cs` (MODIFIED) — add static `AlarmClassNames` dictionary for protocol-fixed IDs 1, 2, 256–272
3. `MongoCommands.cs` (MODIFIED) — add `"ALARM_ACK"` routing branch and new `AlarmAckToPLC()` method
4. `server_realtime_auth/index.js` (MODIFIED) — two new Express routes with `authJwt` middleware
5. `S7PlusAlarmsViewerPage.vue` (NEW) — fully independent Vue 3 component; zero shared code with existing `AlarmsViewerPage.vue`
6. `AdminUI/src/router/index.js` (MODIFIED) — new `/s7plus-alarms` route (distinct from existing `/alarms-viewer`)

**Components that must not change:** `AlarmsViewerPage.vue` (tag-based viewer), existing `commandsQueue` routing for tag writes.

### Critical Pitfalls

1. **ackState always-true — wrong sentinel check** — Empirically verify via live PLC trace (or `AllStatesInfo` bit inspection) which field/value combination represents "not acknowledged." Verify `ackState: false` in MongoDB for a fresh unacknowledged alarm before proceeding to write-back. Fix before building any ack-related UI.

2. **Ack PDU format undocumented — crash risk** — Do not write a single line of ack send code before capturing a Wireshark trace of TIA Portal sending an ack command to a live PLC. A malformed PDU can terminate the alarm subscription connection. Implement and test ack send in isolation on `alarmConn` before integrating.

3. **Ack-only notifications silently dropped — MongoDB never updates** — The v1.0 handler processes only Coming/Going state changes. Ack-only PDUs have a distinct structure and require a separate handler branch that issues `UpdateOne` (not `InsertOne`) on `cpuAlarmId` to set `ackState: true`.

4. **Ack response consumed by notification receive loop** — Sending the ack PDU on the alarm connection risks the response being parsed as a notification. Use a dedicated `alarmConn` for ack sends (consistent with existing separate tag/alarm connection pattern) to prevent this race condition.

5. **Vue route collision and viewer isolation** — New route must be `/s7plus-alarms`, not `/alarms-viewer`. The new component must be fully independent — importing or extending `AlarmsViewerPage.vue` is prohibited by requirement. After adding the new route, verify the existing tag-based viewer still loads correctly.

---

## Implications for Roadmap

Based on combined research, a four-phase structure is recommended. The ordering is driven by two hard constraints: (a) data correctness must precede UI construction, and (b) the ack write-back phase is gated on a Wireshark spike that cannot be skipped.

### Phase 1: Foundation — Environment and Baseline Verification

**Rationale:** Confirm the v1.0 baseline is stable, development environment is functional, and MongoDB data confirms the `ackState` bug is reproducible. Establishes a working baseline before any modifications.
**Delivers:** Running v1.0 system verified; `s7plusAlarmEvents` data confirmed present; `ackState: true` bug reproduced on a known-unacknowledged alarm; PLCSIM or live PLC access confirmed.
**Avoids:** Discovering environment problems mid-implementation; allows verification of the ackState bug before attempting to fix it.

### Phase 2: Driver Fixes — ackState + Alarm Class Name Resolution

**Rationale:** Fix data correctness before building any UI. The viewer has nothing useful to show if `ackState` is always `true` and `alarmClass` is a raw integer. These are isolated C# changes with no frontend dependency and can be developed without a running frontend.
**Delivers:** `ackState` correctly reflects PLC acknowledgement state; `alarmClassName` string field present in all new alarm documents; MongoDB contains accurate, operator-readable alarm data.
**Uses:** `AlarmThread.cs`, `Common.cs`, MongoDB `UpdateOne` — no new dependencies.
**Implements:** Integration points 1 and 2 from ARCHITECTURE.md.
**Avoids:** Pitfall 1 (ackState always-true) — requires live trace to verify correct sentinel before merging; do not guess.
**Research flag:** REQUIRES SPIKE — exact ackState sentinel (`DateTime.MinValue` vs `AllStatesInfo` bit) must be verified against a live PLC or PLCSIM with a known-unacknowledged alarm. This spike is the first task of this phase.

### Phase 3: Read-Only Alarm Viewer (REST Endpoints + Vue Component)

**Rationale:** With correct data in MongoDB, the read-only viewer can be built and validated end-to-end. REST endpoints and Vue component can be developed in parallel. No ack functionality yet — validating the read-only viewer isolates the risk surface before adding write-back complexity.
**Delivers:** `GET /api/s7plus-alarms` endpoint; `S7PlusAlarmsViewerPage.vue` showing TIA Portal-style columns; `/s7plus-alarms` route in AdminUI; auto-refresh every 5s.
**Uses:** Node.js/Express with `authJwt`, Vue 3, Vuetify `v-data-table`, `fetch()` + `setInterval`.
**Implements:** Integration points 4 and 5 from ARCHITECTURE.md.
**Avoids:** Pitfall 5 (route collision) — new route `/s7plus-alarms`, existing tag-based viewer verified unchanged after new route is added.
**Research flag:** Standard patterns — well-documented Vue/Vuetify component and Express route patterns consistent with existing codebase; no additional research needed.

### Phase 4: Ack Write-Back + Acknowledge Button (GATED ON WIRESHARK SPIKE)

**Rationale:** Highest-risk phase. The S7CommPlus ack PDU format is undocumented. A Wireshark spike must complete and be reviewed before implementation begins. Once PDU format is confirmed, the full ack pipeline (Vue → Express → commandsQueue → C# → PLC → ack notification → MongoDB UpdateOne) can be implemented.
**Delivers:** `POST /api/s7plus-alarms/ack` endpoint; `commandsQueue` extended with `"ALARM_ACK"` ASDU; `AlarmAckToPLC()` method; ack-only notification handler in `AlarmThread.cs`; Acknowledge button in viewer enabled for unacknowledged alarms with pending state and duplicate-send prevention.
**Uses:** `MongoCommands.cs`, `AlarmThread.cs` (ack-only notification branch), `commandsQueue` collection, dedicated `alarmConn`.
**Implements:** Integration point 3 from ARCHITECTURE.md (ack write-back data flow).
**Avoids:** Pitfall 2 (undocumented PDU — crash risk), Pitfall 3 (ack-only notifications dropped), Pitfall 4 (ack response consumed by notification loop).
**Research flag:** REQUIRES WIRESHARK SPIKE — the first deliverable of this phase is a captured and decoded Wireshark trace of TIA Portal sending an ack command to a live PLC. Do not write ack send code until the `SetVariableRequest` payload (object ID, attribute ID, value type) is confirmed from the captured bytes.

### Phase Ordering Rationale

- Phase 2 before Phase 3: The viewer requires correct `ackState` and `alarmClassName` data to be meaningful. Building the UI on broken data validates nothing and creates rework.
- Phase 3 before Phase 4: The ack button depends on the viewer existing; validating the read-only path first isolates any bugs in data flow and API before adding write-back complexity.
- Phase 4 gated on spike: The Wireshark spike is the single largest uncertainty in the project. It must be the first deliverable of Phase 4, not a parallel workstream or an assumption.

### Research Flags

Phases requiring spikes or deeper investigation during planning:

- **Phase 2:** Wireshark capture or `AllStatesInfo` bit inspection required to confirm correct `ackState` sentinel for unacknowledged alarms. Cannot be resolved from static code analysis alone. Gate: merge only after `ackState: false` is confirmed in MongoDB for a live unacknowledged alarm.
- **Phase 4:** Wireshark capture of TIA Portal sending an ack command is a hard gate before any implementation. The `SetVariableRequest` payload for alarm ack is not in `Ids.cs` or any existing driver code.

Phases with standard patterns (additional research not needed):

- **Phase 3:** Vue 3 + Vuetify `v-data-table`, Express routes with `authJwt`, `fetch()` polling — well-documented and consistent with patterns already present in the codebase.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All technologies confirmed by direct `package.json`, `.csproj`, and source file inspection. No speculative dependencies. |
| Features | HIGH | Feature set derived from v1.0 codebase, TIA Portal screenshot, and ISA 18.2 four-state alarm model. All fields already in MongoDB. |
| Architecture | HIGH (one LOW exception) | Component boundaries, data flow, route patterns, and integration points confirmed from direct code inspection. Exception: ack PDU structure is LOW confidence — undocumented reverse-engineered protocol requiring Wireshark spike. |
| Pitfalls | HIGH | Derived from direct code inspection of known bugs, protocol constraints, and established SCADA patterns. All pitfalls have specific locations and prevention strategies. |

**Overall confidence:** HIGH — with the explicit caveat that the alarm ack PDU format is an open question that must be resolved via Wireshark spike before Phase 4 implementation begins.

### Gaps to Address

- **Alarm ack PDU structure (CRITICAL — gates Phase 4):** The exact `SetVariableRequest` payload to acknowledge an alarm over S7CommPlus is not documented in `Ids.cs` or any existing driver code. Resolution: Wireshark capture of TIA Portal performing an ack on a live PLC, then decode object ID, attribute ID, and value encoding from the captured bytes. This gap must be closed before Phase 4 implementation begins.

- **ackState correct sentinel (IMPORTANT — gates Phase 2 merge):** Whether the unacknowledged sentinel is `DateTime.MinValue`, zero epoch, or a specific `AllStatesInfo` bit is not confirmed from static code inspection alone. Resolution: Trigger an alarm on PLCSIM without acknowledging it, then inspect the raw `dai.AsCgs` fields in a debugger or from a Wireshark trace. This gap must be closed before Phase 2 is merged.

- **server_realtime_auth route registration pattern (MINOR):** Whether new routes are registered inline in `index.js` or in an `app/routes/` file, and whether `authJwt` is applied at the router or route level, needs a quick read of `server_realtime_auth/index.js`. Low risk — the pattern is consistent across the codebase and any deviation is easy to spot and correct.

---

## Sources

### Primary (HIGH confidence — direct codebase inspection)

- `S7CommPlusClient/AlarmThread.cs` — ackState computation, alarm document build logic
- `S7CommPlusClient/S7CommPlusDriver/Alarming/AlarmsMultipleStai.cs` — AlarmDomain/AlarmClass numeric ID encoding (1, 2, 256–272)
- `S7CommPlusClient/MongoCommands.cs` — commandsQueue consumer, ASDU routing pattern
- `json-scada/src/server_realtime_auth/index.js` — Express route and authJwt middleware pattern
- `AdminUI/src/router/index.js` — existing route structure
- `AdminUI/src/components/AlarmsViewerPage.vue` — existing viewer pattern (5-line iframe wrapper; must not be modified)
- `AdminUI/package.json` — Vue 3.4.31, Vuetify 3.10, vue-router 4.4.0 confirmed installed
- `S7CommPlusClient/S7CommPlusClient.csproj` — MongoDB.Driver 3.4.2, net8.0, x64 confirmed

### Secondary (MEDIUM confidence — standards and protocol corpus)

- ISA 18.2 alarm management standard — four-state alarm model (Unacked Active / Acked Active / Unacked Cleared / Acked Cleared)
- IEC 62682 — SCADA alarm management best practices
- S7CommPlus reverse-engineering corpus (open62541-s7commplus, s7commWireshark) — protocol behavior reference for ack PDU structure

### Tertiary (LOW confidence — requires validation)

- Alarm ack PDU structure — inferred from `SetVariableRequest` infrastructure present in the driver; exact payload requires Wireshark trace against live PLC before implementation
- TIA Portal user-defined alarm class display names — confirmed NOT transmitted in S7CommPlus notifications; English static mapping is the only viable approach without out-of-band TIA Portal configuration export

---

*Research completed: 2026-03-18*
*Ready for roadmap: yes*

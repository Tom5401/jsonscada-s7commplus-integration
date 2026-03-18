# Pitfalls Research

**Domain:** S7CommPlus alarm ack write-back, alarm class name resolution, Vue 3 SCADA alarm viewer
**Researched:** 2026-03-18
**Confidence:** HIGH

## Critical Pitfalls

### Pitfall 1: ackState Always-True — Wrong Field or Sentinel Check

**What goes wrong:**
The `ackState` field in MongoDB is always `true` because the code reads the wrong field or applies the wrong null/zero check. In S7CommPlus alarm notifications, the acknowledged timestamp (or equivalent sentinel) being `DateTime.MinValue` / zero-epoch signals "not acknowledged." If the code compares the wrong property, every alarm appears acknowledged regardless of PLC state.

**Why it happens:**
Alarm notification PDUs contain multiple timestamp fields. Without a Wireshark trace and verified field mapping, it is easy to read the wrong one or apply the wrong null check. The v1.0 code may have mapped `ackState` from a field that is always non-null.

**How to avoid:**
Empirically verify from a live PLC trace which field/value combination represents "not acknowledged." Trigger an alarm without acknowledging it, verify `ackState: false` in MongoDB before writing any ack logic.

**Warning signs:**
All alarms in MongoDB have `ackState: true` immediately on arrival, even fresh unacknowledged alarms.

**Phase to address:**
Ack fix phase (first phase of v1.1) — fix before implementing write-back.

---

### Pitfall 2: Ack Write-Back PDU Format — Undocumented Protocol Risk

**What goes wrong:**
The S7CommPlus ack command PDU is reverse-engineered and undocumented. Sending a malformed ack PDU can crash the alarm subscription thread, terminate the connection, or silently do nothing at the PLC.

**Why it happens:**
Without official documentation, PDU field order, sizes, and required identifiers (CpuAlarmId, alarm connection ID) must be inferred from Wireshark captures of TIA Portal sending ack commands.

**How to avoid:**
Capture a Wireshark trace of TIA Portal sending an ack command before writing any code. Verify the ack PDU structure against captured bytes. Test ack send in isolation on a spare connection before integrating into AlarmThread.

**Warning signs:**
PLC disconnects after ack attempt; alarm subscription stops receiving notifications; no ack confirmation visible in TIA Portal alarm viewer.

**Phase to address:**
Ack write-back phase — Wireshark trace required before implementation begins.

---

### Pitfall 3: Ack-Only Notifications Never Update MongoDB

**What goes wrong:**
When an alarm is acknowledged, the PLC sends an "ack-only" notification (no state change, just ack confirmation). If the handler only processes Coming/Going events, ack confirmations are silently dropped and `ackState` in MongoDB never updates.

**Why it happens:**
The v1.0 alarm notification handler was written for state changes. Ack-only notifications have a different structure and require parsing `CpuAlarmId` from the raw `PObject` and issuing a `MongoDB UpdateOne` (not `InsertOne`).

**How to avoid:**
Implement a separate handler branch for ack-only notifications. Issue `UpdateOne` with filter on `cpuAlarmId` (and optionally event timestamp) to set `ackState: true`.

**Warning signs:**
Ack command sent to PLC successfully (no error) but `ackState` in MongoDB remains `false`; ack confirmation visible in TIA Portal but not in mongodb.

**Phase to address:**
Ack write-back phase — implement ack notification handler alongside ack send.

---

### Pitfall 4: Ack PDU Response Consumed by Notification Receive Loop

**What goes wrong:**
Sending the ack PDU on the alarm connection races with the notification receive loop. The ack response PDU gets consumed by `WaitForAlarmNotification` as if it were a notification, causing parse errors or silent discard.

**Why it happens:**
The alarm connection has a single receive loop expecting only subscription notifications. Injecting an ack command on the same connection means the response arrives interleaved with notification PDUs.

**How to avoid:**
Use a separate connection for ack commands (safest, consistent with the existing separate tag/alarm connection pattern), OR implement a PDU type check at the top of the receive loop routing ack responses to a dedicated handler.

**Warning signs:**
Intermittent parse errors in alarm notification processing after ack commands are sent; notifications stop arriving after first ack.

**Phase to address:**
Ack write-back phase — connection architecture decision before implementation.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Polling alarm table in Vue (setInterval) | Simpler implementation | Stale reads, lost row selections, unnecessary DB queries | Never for SCADA alarm viewer — use subscriptions or SSE |
| Optimistic ack UI update (mark acked before driver confirms) | Responsive UX | Unsafe — PLC may reject ack, UI diverges from PLC state | Never for safety-relevant alarms |
| `ExploreASAlarms` called per alarm event | No caching logic | 3 PDU round-trips per alarm — kills performance | Never — run once at startup, cache result |
| Hardcoding alarm class names in frontend | Avoids lookup complexity | Breaks when PLC config changes; brittle | Only for single-PLC throwaway demos |
| Routing ack through existing `commandsQueue` (tag-write collection) | No new infrastructure | Alarm ack is a different ASDU — wrong routing silently drops ack | Never — use dedicated collection or explicit routing case |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| S7CommPlus `ExploreASAlarms` | Calling per-event or sharing the alarm receive loop | Run once at startup before `AlarmSubscriptionCreate`; cache alarm class map in memory |
| MongoDB ack update | Using `InsertOne` for ack confirmation | Use `UpdateOne` with filter on `cpuAlarmId`; only update `ackState` field |
| Vue Router new alarm viewer | Adding route at `/alarms-viewer` (same as existing tag-based viewer) | Use a distinct route e.g. `/s7plus-alarms`; verify no route collision in router index |
| AdminUI new page | Importing or extending existing `AlarmsViewerPage.vue` | New component is fully independent; zero shared code with existing viewer |
| Ack API layer | Bypassing json-scada GraphQL server with direct MongoDB write from frontend | All mutations go through the server layer; ack command routes to C# driver via MongoDB command collection |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Polling-based alarm table (setInterval) | Stale rows, flickering on ack, lost scroll position | GraphQL subscriptions or SSE for live updates | Immediately under any real alarm load |
| `ExploreASAlarms` per reconnect without cache | 3–5s delay on every reconnect | Cache alarm class map; refresh only on explicit reload or config change | Every reconnect after connection loss |
| `InsertOne` per ack update (creates duplicate document) | Duplicate alarm events in DB | `UpdateOne` with upsert=false for ack fields | First ack attempt |

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| No validation on ack command input (alarm ID from UI) | Malformed CpuAlarmId could crash alarm thread or send bad PDU to PLC | Validate alarm ID format server-side before forwarding to C# driver |
| Exposing alarm ack API without auth | Unauthenticated users can acknowledge safety alarms | Reuse json-scada's existing auth mechanism for the ack API endpoint |

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| No visual feedback during ack command in flight | Operator double-clicks, sends duplicate ack commands | Disable ack button while command in flight; re-enable after MongoDB confirmation |
| Mixing S7Plus alarms with tag-based alarms in same viewer | Confusing; different semantics and fields | Keep viewers fully separate; clear navigation labels |
| No "pending ack" state | Operator unsure if ack was received | Show "Acknowledging..." intermediate state; update to "Acknowledged" only after MongoDB confirms |
| Showing raw numerical alarmClass without name | Operators cannot interpret alarm class | Always display class name; number is secondary info |
| Auto-scrolling alarm table during live updates | Operator loses their place while reading | Pause live updates / scroll position when user is interacting with the table |

## "Looks Done But Isn't" Checklist

- [ ] **ackState fix:** Verify `ackState: false` in MongoDB for a fresh unacknowledged alarm (not just `true` for acknowledged)
- [ ] **Ack write-back:** Verify PLC-side ack reflected in TIA Portal alarm viewer after sending from AdminUI
- [ ] **Ack-only notification:** Verify MongoDB `ackState` updates to `true` after ack confirmation from PLC (not just after sending command)
- [ ] **Alarm class name:** Verify name appears for all alarm classes, including alarms received immediately after startup
- [ ] **New viewer route:** Verify existing tag-based `/alarms-viewer` still works unchanged after adding new S7Plus viewer route
- [ ] **Ack command connection:** Verify alarm subscription notifications continue uninterrupted after ack command sent (no thread crash)
- [ ] **Duplicate ack prevention:** Verify double-clicking Acknowledge does not send two PDUs to PLC

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| ackState always-true bug | LOW | Wireshark capture raw notification, identify correct field, update field mapping in code |
| Ack PDU crashes alarm thread | MEDIUM | Revert ack send, restart alarm subscription, capture Wireshark trace before retry |
| Route collision breaks existing viewer | LOW | Rename new viewer route, update nav links |
| Alarm class names missing after reconnect | LOW | Verify `ExploreASAlarms` called at startup; check cache populated before first notification |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| ackState always-true | Phase 2 (ack read fix) | MongoDB shows `false` for unacked alarms in PLCSIM |
| Ack PDU format | Phase 3 (write-back) | Wireshark trace matches expected ack PDU; PLC accepts without disconnect |
| Ack-only notification dropped | Phase 3 (write-back) | MongoDB `ackState` updates after PLC ack confirmation |
| Ack response consumed by loop | Phase 3 (write-back) | Notifications continue normally after ack sent |
| `ExploreASAlarms` per-event | Phase 2 (alarm class) | Startup log shows single call; alarm class map cached |
| Vue route collision | Phase 4 (viewer) | Existing tag-based viewer loads correctly after new route added |
| Optimistic ack UI | Phase 4 (viewer) | UI updates only after MongoDB confirmation, not on button click |

## Sources

- Direct code inspection: v1.0 AlarmThread.cs, AlarmsHandler.cs, Common.cs
- S7CommPlus reverse-engineering corpus (open62541-s7commplus, s7commWireshark)
- SCADA alarm management best practices (IEC 62682)
- Vue 3 + Vuetify 3 router and reactivity patterns
- json-scada AdminUI source: AlarmsViewerPage.vue (existing tag-based alarm viewer)

---
*Pitfalls research for: S7CommPlus alarm ack write-back, alarm class resolution, Vue 3 alarm viewer*
*Researched: 2026-03-18*

# Pitfalls Research

**Domain:** S7CommPlus alarm subscription integration into a running C#/.NET SCADA driver
**Researched:** 2026-03-17
**Confidence:** HIGH (all findings are grounded in direct code inspection of S7CommPlusDriver and S7CommPlusClient)

---

## Critical Pitfalls

### Pitfall 1: SubscriptionSetCreditLimit Uses the Wrong ObjectId in Alarm Context

**What goes wrong:**
`AlarmsHandler.cs` calls `SubscriptionSetCreditLimit(m_AlarmNextCreditLimit)` (line 158) when the credit tick threshold is reached. However, `SubscriptionSetCreditLimit` in `Subscription.cs` (line 129–138) writes to `m_SubscriptionObjectId` — the tag subscription's object ID — not `m_AlarmSubscriptionObjectId`. The alarm subscription is a separate PLC object with its own ID saved in `m_AlarmSubscriptionObjectId` (line 112). Sending the credit limit renewal to the wrong object ID means the alarm subscription stops delivering notifications silently when the 10-event initial credit is exhausted.

**Why it happens:**
Both the alarm and tag subscription code paths are marked as "test/experimental" and live in separate partial-class files. The alarm handler reuses a private method from the tag subscription partial class that was written to operate on tag subscription state. There is no alarm-specific `SetCreditLimit` method. The test code happened to work in short demos (firing 3 alarms) that never exhausted the credit.

**How to avoid:**
Implement a dedicated `AlarmSubscriptionSetCreditLimit(short limit)` method in the alarm partial class that sets the variable on `m_AlarmSubscriptionObjectId` instead of `m_SubscriptionObjectId`. This is a one-method addition mirroring the existing `SubscriptionSetCreditLimit`.

**Warning signs:**
- Alarm notifications stop arriving after exactly 10 events (the initial `m_AlarmNextCreditLimit = 10` value)
- No error is logged because `SubscriptionSetCreditLimit` fires-and-forgets (flag `0x74` = no response required)
- PLC subscription object shows zero free credit when queried via diagnostics

**Phase to address:**
Phase 1 (Alarm subscription wire-up). Must be corrected before any functional testing — a demo that fires 11+ alarms will silently stall.

---

### Pitfall 2: Blocking Receive Loop Cannot Coexist with the Read Cycle

**What goes wrong:**
`TestWaitForAlarmNotifications` blocks the calling thread inside `WaitForNewS7plusReceived` (spin-sleep loop on `m_ReceivedPDUs`). The existing `ConnectionThread` in `Program.cs` runs a polling read cycle (`PerformReadCycle`) in the same thread. Alarm subscription notifications arrive as unsolicited PDUs into `m_ReceivedPDUs`. If the alarm wait loop holds the thread, no tag reads happen. If the tag read cycle holds the thread, alarm PDUs queue up but are never consumed, and eventually the PLC-side subscription times out or credit is never renewed.

**Why it happens:**
`TestWaitForAlarmNotifications` is test scaffolding designed to be called from a dedicated test program that does nothing else. It is not an event-driven callback — it is a blocking for-loop. Integrating it into a production driver that also performs tag reads requires architectural change.

**How to avoid:**
Run alarm notification consumption in a dedicated thread separate from the tag read cycle. The `m_ReceivedPDUs` queue with its `m_Mutex` is safe for concurrent access between the reader callback and a consumer thread. A dedicated alarm thread calls `WaitForNewS7plusReceived` in a loop, checks if the dequeued PDU is an alarm notification (opcode check), and dispatches accordingly. Tag reads continue in `ConnectionThread` without competing for the queue.

**Warning signs:**
- Tag polling rate drops to near-zero when alarms are active
- Credit limit renewals are delayed because the alarm loop is blocked waiting on a tag read response
- Timeouts logged by `WaitForNewS7plusReceived` on the alarm consumer when a tag read response was dequeued by it instead

**Phase to address:**
Phase 1 (Alarm subscription wire-up / integration into S7CommPlusClient threading model). This is the most architecturally significant decision of the integration.

---

### Pitfall 3: AlarmSubscriptionDelete Ignores the Saved ObjectId

**What goes wrong:**
`AlarmSubscriptionDelete` (line 165–172) first zeroes `m_AlarmSubscriptionObjectId`, then calls `DeleteObject(SessionId2)`. It deletes the session object (the whole session), not the specific alarm subscription object. Compare with the description in the code comment: "Save the ObjectId, to modify the existing subscription." The ObjectId is saved but never used for deletion. In practice this tears down the entire connection rather than cleanly unsubscribing. On reconnect, the PLC may have a dangling subscription object consuming a subscription slot.

**Why it happens:**
`AlarmSubscriptionDelete` appears to have been written to mirror the session teardown approach before the object-ID-based delete was understood. The regular `SubscriptionDelete` has the same pattern — both use `SessionId2`. Whether deleting the session object is semantically equivalent to deleting the subscription object is undocumented (it is reverse-engineered protocol).

**How to avoid:**
After `AlarmSubscriptionCreate` succeeds, store the returned `ObjectId` and use it for deletion. Before reconnecting, call `CommRessources.ReadFree` to verify that `PlcSubscriptionsFree` is not zero. On reconnect, attempt to delete any orphaned subscription if the PLC reports subscriptions in use. Log the `PlcSubscriptionsFree` value at startup and after each connect/disconnect cycle.

**Warning signs:**
- `PlcSubscriptionsFree` decreases each time the driver restarts without recovering
- `AlarmSubscriptionCreate` returns a non-zero error code (subscription slot exhausted)
- PLC diagnostics show stale subscription objects from previous sessions

**Phase to address:**
Phase 1 (subscription lifecycle management). Pre-empt this by reading `CommRessources` on connect and comparing `PlcSubscriptionsFree` before and after subscription create/delete.

---

### Pitfall 4: RelationId Collision Between Tag and Alarm Subscriptions

**What goes wrong:**
Both `m_SubscriptionRelationId` (Subscription.cs line 33) and `m_AlarmSubscriptionRelationId` (AlarmsHandler.cs line 36) are initialized to `0x7fffc001`. The comment says "seems to be a startvalue, increases on next CreateObject" but neither value is actually incremented in the code. If both a tag subscription and an alarm subscription are created on the same connection instance, they both use the same RelationId. The PLC behavior for duplicate RelationIds is unknown (reverse-engineered protocol). The result may be a failed create, a silent collision, or one subscription overwriting the other on the PLC side.

**Why it happens:**
The test code was written for two separate use cases (test tag subscription only, or test alarm subscription only). No combined use case was ever tested, so the collision was never observed.

**How to avoid:**
Use different initial values for the alarm subscription RelationId (for example `0x7fffc002` or any value that does not conflict). Because the semantics of RelationId are not fully understood, log both values at subscription create time and observe PLC diagnostics to confirm both subscriptions exist simultaneously as distinct objects.

**Warning signs:**
- `AlarmSubscriptionCreate` returns success but no alarm notifications arrive
- `CommRessources.PlcSubscriptionsFree` only decreases by 1 after creating both subscriptions (expected: decrease by 2)
- PLC subscription object list shows only one entry

**Phase to address:**
Phase 1 (alarm subscription create). Assign a distinct RelationId before first test.

---

### Pitfall 5: NotImplementedException Crashes the Consumer Thread on Unknown PDU Return Codes

**What goes wrong:**
`Notification.Deserialize` (Notification.cs lines 126–129) throws `NotImplementedException` for return codes `0x83` and `default`. In the test code this is never reached because the PLC sends the expected codes. In production, unusual PDUs (version mismatches, PLC firmware updates, edge-case alarm types) can trigger these paths. An unhandled `NotImplementedException` in the alarm consumer thread kills that thread silently if it is not caught at the loop level. The existing `ConnectionThread` in `Program.cs` wraps the loop body in `catch (Exception e)` which would catch and log this, but any dedicated alarm thread must do the same.

**Why it happens:**
The test-grade notification parser was written to throw on anything unexpected rather than skip or log. This is appropriate for a test harness where you want to know about new protocol behavior. It is not appropriate for a production consumer that must stay running.

**How to avoid:**
Wrap `Notification.DeserializeFromPdu` in a try/catch in the alarm consumer loop. On `NotImplementedException` or any deserialization exception: log the raw PDU bytes at debug level (for future reverse-engineering), increment a dropped-event counter, and continue the loop. Do not re-throw. The alarm subscription must not terminate because of one unexpected PDU.

**Warning signs:**
- Alarm consumer thread stops processing but no disconnect is observed
- Log shows a single `NotImplementedException` followed by silence
- `PlcSubscriptionsFree` value does not recover (subscription is still live, client is just not consuming)

**Phase to address:**
Phase 1 (alarm consumer loop implementation). Standard exception hygiene, but easy to miss when porting test code.

---

### Pitfall 6: `FromNotificationObject` Returns Null Without Exception for Ack-Only Notifications

**What goes wrong:**
`AlarmsDai.FromNotificationObject` (AlarmsDai.cs lines 69–82) returns `null` when neither `DAI_Coming` nor `DAI_Going` is present in the notification object. The S7CommPlus protocol can send acknowledgement-only notifications (the alarm state did not change, but it was acknowledged). These notifications do not carry a Coming or Going subtype. If the caller does not null-check the result before accessing fields, a `NullReferenceException` is thrown. `TestWaitForAlarmNotifications` does not null-check `dai` before calling `Console.WriteLine(dai.ToString())` on line 152.

**Why it happens:**
The test code only exercised Coming and Going transitions. Ack-only notifications were not tested. The null return was added as a defensive measure but the caller was not updated.

**How to avoid:**
Always null-check the return of `FromNotificationObject`. When null is returned, log the raw `PObject` attributes for diagnostic purposes and skip the event without crashing. If ack state needs to be tracked, add a separate code path that inspects `DAI_Acknowledged` attributes directly on the `PObject` before calling `FromNotificationObject`.

**Warning signs:**
- `NullReferenceException` in the alarm consumer loop at the line that accesses `dai.CpuAlarmId` or similar
- Alarm events appear but acknowledgement state in MongoDB never updates
- Observed after an operator acknowledges an alarm that was already cleared

**Phase to address:**
Phase 1 (alarm event parsing). Add null-check as part of the initial integration, before testing with a real PLC.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Reuse `SubscriptionSetCreditLimit` for alarms | No new code | Alarm flow stops at 10 events — silent data loss | Never |
| Single `ConnectionThread` handles both reads and alarm receive | Simpler | Tag polling and alarm consumption starve each other | Never |
| Log `dai.ToString()` instead of structured MongoDB insert | Fast PoC | Alarm data is not persisted, PoC value is lost | Only if the goal is to verify protocol decode only, not end-to-end |
| Hardcode language ID (e.g. `1031` for de-DE) | Works on test PLC | Alarms return empty text on PLCs configured with different locale | Acceptable in PoC if PLC locale is known and fixed |
| Skip `CommRessources.ReadFree` on reconnect | Fewer round-trips on connect | Subscription slot exhaustion goes undetected | Never in a driver expected to restart |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| PLC alarm subscription + tag read on one connection | Calling `WaitForNewS7plusReceived` from two code paths; one consumes the other's PDU | Dedicate one thread to alarm PDU consumption; use opcode check to route PDUs if sharing a queue |
| MongoDB alarm collection | Inserting alarm events with `DateTime.Now` (local time) while PLC timestamps from `DtFromValueTimestamp` are UTC | Use `DateTime.UtcNow` for server-side timestamps; store PLC `AsCgs.Timestamp` (already UTC) as the authoritative event time |
| `AlarmSubscriptionCreate` return value check | Checking `res != 0` from `SendS7plusFunctionObject` but not checking `createObjRes.ReturnValue` | Both must be checked; `res==0` from send means the PDU was transmitted, not that the PLC accepted the subscription |
| S7-1200 vs S7-1500 firmware variation | `SubscriptionChangeCounter == 0` path in `Notification.Deserialize` is for newer 1500 firmware; ignoring it parses timestamps incorrectly on older CPUs | The existing parser handles this branching; do not simplify it |
| `AlarmSubscriptionRef_AlarmDomain` / `AlarmDomain2` | Setting both to all-zeros/65535 subscribes to all alarms; if the PLC has many alarm classes, the initial burst on connect can overwhelm the 10-event credit before credit renewal fires | Start with a higher initial credit limit (50–100) or set it to -1 (unlimited) during initial development to avoid false stalls |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Inserting one MongoDB document per alarm event with `InsertOneModel` | Write latency grows under rapid alarm bursts | Batch alarm inserts using the existing `BulkWriteAsync` pattern from `MongoUpdate.cs`, or a `ConcurrentQueue` analogous to `DataQueue` | Any alarm storm (>20 alarms/second) |
| Keeping full alarm text strings in memory for all active alarms | Memory grows unbounded on high-alarm PLCs | Alarm text is embedded in the notification (because `SendAlarmTexts = true`); no in-memory text cache is needed — write directly to MongoDB | Not a concern in this PoC scope |
| Calling `CommRessources.ReadFree` on every alarm notification | Extra PLC round-trip per event | Call `ReadFree` once on connect and after each subscription create/delete; not during normal notification processing | Immediately at >1 alarm/second |

---

## "Looks Done But Isn't" Checklist

- [ ] **Credit limit renewal:** Alarm notifications arrive during first 10 events — verify by triggering 15+ alarm transitions that all appear in MongoDB.
- [ ] **Subscription cleanup on disconnect:** After stopping the driver and restarting, verify `PlcSubscriptionsFree` is the same value as before the first run.
- [ ] **PLC-side timestamp in MongoDB:** Verify the stored timestamp comes from `AsCgs.Timestamp` (UTC, nanosecond precision) and not from `DateTime.Now` (local time, second precision).
- [ ] **AlarmsDai null guard:** Trigger an alarm acknowledgement without a state change (ack-only notification) and confirm no `NullReferenceException` appears.
- [ ] **Concurrent tag reads:** Verify tag polling continues at the configured `giInterval` rate while alarm notifications are arriving.
- [ ] **Non-zero `createObjRes.ReturnValue`:** On subscription create failure, the object is still created on the PLC (per code comment line 116). Verify that `AlarmSubscriptionDelete` is called even when `ReturnValue != 0`.

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Credit limit exhaustion (alarm flow stopped) | LOW | Reconnect to PLC; subscription is re-created and credit restarts. Fix the `SubscriptionSetCreditLimit` bug before next run. |
| Subscription slot exhaustion (PLC refuses new subscription) | MEDIUM | Use TIA Portal diagnostics to manually delete stale subscription objects, or power-cycle the PLC. Implement slot-check on reconnect to prevent recurrence. |
| Alarm consumer thread death from unhandled exception | LOW | Restart driver process; add top-level exception handling to the alarm loop. |
| RelationId collision (one subscription silently displaces the other) | LOW | Assign distinct RelationId values; verify both subscriptions exist via `CommRessources.ReadFree`. |
| MongoDB alarm collection missing timezone context | MEDIUM | Retroactively convert stored timestamps if UTC vs local is ambiguous. Prevent by storing timezone metadata in the collection schema from day one. |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| SubscriptionSetCreditLimit uses wrong ObjectId | Phase 1: alarm subscription wire-up | Trigger 15 alarms; confirm all 15 appear in MongoDB |
| Blocking receive loop conflicts with read cycle | Phase 1: threading model | Confirm tag poll rate is unchanged while alarms are firing |
| AlarmSubscriptionDelete ignores ObjectId / slot leak | Phase 1: lifecycle management | Check `PlcSubscriptionsFree` before and after 3 connect/disconnect cycles |
| RelationId collision | Phase 1: subscription create | Confirm `PlcSubscriptionsFree` decreases by exactly 1 per subscription created |
| NotImplementedException crashes consumer thread | Phase 1: alarm consumer loop | Send malformed or unexpected PDU (or simulate with unit test); confirm thread continues |
| AlarmsDai null on ack-only notification | Phase 1: event parsing | Acknowledge an alarm and confirm no exception; log entry shows ack event received |
| Server timestamp UTC inconsistency | Phase 2: MongoDB schema | Query stored documents and compare `plcTimestamp` with `serverTimestamp` fields; both should be UTC |

---

## Sources

- Direct code inspection: `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHandler.cs` — alarm subscription create/wait/delete, credit limit logic
- Direct code inspection: `S7CommPlusDriver/src/S7CommPlusDriver/Subscriptions/Subscription.cs` — `SubscriptionSetCreditLimit` uses `m_SubscriptionObjectId`
- Direct code inspection: `S7CommPlusDriver/src/S7CommPlusDriver/Core/Notification.cs` — `NotImplementedException` on unknown return codes, two-path timestamp parsing
- Direct code inspection: `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsDai.cs` — null return on missing Coming/Going subtype
- Direct code inspection: `S7CommPlusDriver/src/S7CommPlusDriver/Core/CommRessources.cs` — `PlcSubscriptionsFree`, `PlcSubscriptionsMax` readable from PLC
- Direct code inspection: `json-scada/src/S7CommPlusClient/Program.cs` — `ConnectionThread` structure, exception handling pattern
- Direct code inspection: `json-scada/src/S7CommPlusClient/Common.cs` — `DateTime.Now` used for `serverTimestamp` vs UTC alarm timestamps
- Direct code inspection: `json-scada/src/S7CommPlusClient/MongoUpdate.cs` — bulk write pattern, `DataQueue` as reference for alarm queue design
- Author comments in source: "IMPORTANT: This is basically a test for Alarming, how to use it, and how to later integrate this into the complete library!" (AlarmsHandler.cs line 28–30)

---
*Pitfalls research for: S7CommPlus alarm subscription integration (json-scada S7CommPlusClient)*
*Researched: 2026-03-17*

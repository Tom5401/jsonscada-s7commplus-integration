# Phase 4: Ack Write-Back - Research

**Researched:** 2026-03-19
**Domain:** S7CommPlus alarm acknowledgement PDU, Express POST endpoint, Vue 3 pending-state UX, MongoDB commandsQueue pattern
**Confidence:** HIGH (all findings verified against actual source code in the repo)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Spike deliverable (gates implementation)**
- Produce a decoded fields doc committed to `.planning/phases/04-ack-write-back/` — file must exist before any pipeline implementation begins
- Doc must contain: decoded `SetVariableRequest` fields (alarm object ID, attribute ID, value encoding) **plus** a prototype C# method that constructs and sends the PDU (not yet wired to commandsQueue)
- The prototype proves the PDU works against a live PLC before the full pipeline is wired up

**Acknowledge button UX**
- The `Acknowledge` column in `S7PlusAlarmsViewerPage.vue` replaces the `mdi-close` (red) icon with a small `Ack` button for unacknowledged rows (`ackState: false`)
- Button: `v-btn size="x-small" variant="tonal"` — compact, fits the table cell
- Acknowledged rows (`ackState: true`) keep their existing `mdi-check` (green) icon — no change
- **Pending state**: spinner (`v-progress-circular` small) replaces button text; button is disabled to prevent duplicate sends
- **On failure** (POST error or PLC rejects): re-enable the `Ack` button; log `console.warn` only — no toast, no extra infrastructure (PoC)

**Post-ack confirmation**
- Button stays in spinner pending state until the regular `setInterval` poll (5000ms) fires and `fetchAlarms()` returns a document with `ackState: true` for that alarm
- **If poll returns but `ackState` is still `false`**: exit pending state and re-enable the `Ack` button — operator can retry
- No targeted re-fetch or optimistic update — wait for the next poll cycle (simple, consistent with existing refresh pattern)

**commandsQueue document shape for alarm ack**
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

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| DRVR-03 | Acknowledgement command sent from json-scada is received and applied by the PLC | Spike spike must decode SetVariableRequest PDU format; C# branch in MongoCommands.cs sends the PDU via the established S7CommPlusConnection on the main connection |
| VIEW-03 | User can acknowledge an unacknowledged alarm from the viewer | Ack button in ackState column template; POST to Express endpoint; pending state via local Set; confirmation via existing 5000ms poll |
</phase_requirements>

---

## Summary

Phase 4 closes the ack loop: Vue button → Express POST → `commandsQueue` insert → C# Change Stream → S7CommPlusDriver `SetVariableRequest` → PLC → `ackState: true` in MongoDB → next poll cycle resolves pending state in the viewer.

The most critical unknown in this phase is the exact binary encoding of the alarm acknowledgement PDU. The S7CommPlusDriver library has no built-in `AlarmAcknowledge()` method — ack is a `SetVariableRequest` addressed to an object within the alarm subsystem, but the `InObjectId` and `Address` values are not exposed as named constants and must be reverse-engineered via Wireshark capture of TIA Portal sending an actual ack. This spike is the hard gate for all other implementation work; sending a malformed ack PDU can crash the alarm subscription thread.

All other pipeline pieces (commandsQueue insert pattern, Change Stream branching, Vue v-btn/v-progress-circular, fetchAlarms poll) are fully understood from existing code. The implementation risk is concentrated in the spike. Once the spike prototype proves the PDU is accepted by a live PLC, the three downstream tasks (Express endpoint, C# branch, Vue UX) are individually straightforward and can proceed in any order.

**Primary recommendation:** Execute the Wireshark spike first and commit the decoded fields doc before writing any pipeline code. The prototype C# method in that doc becomes the implementation template for the MongoCommands.cs branch.

---

## Standard Stack

### Core (all already installed — no new dependencies)
| Library/Component | Version | Purpose | Notes |
|-------------------|---------|---------|-------|
| S7CommPlusDriver | local (repo) | Send ack PDU to PLC via SetVariableRequest | No AlarmAcknowledge() API exists — must construct PDU manually; spike determines fields |
| MongoDB C# Driver | existing | Change Stream watches commandsQueue, updates alarm doc after ack | Already in S7CommPlusClient.csproj |
| Express (Node.js) | existing | POST endpoint `/Invoke/auth/ackS7PlusAlarm` | Inline in index.js AUTHENTICATION block, same pattern as listS7PlusAlarms |
| Vue 3 + Vuetify | existing | Ack button, pending spinner | v-btn, v-progress-circular already available in AdminUI |
| MongoDB Node driver | existing | commandsQueue insertOne, s7plusAlarmEvents update | db handle available in server_realtime_auth/index.js |

### No new packages required
All libraries needed are already present. No `npm install` or `dotnet add package` steps.

---

## Architecture Patterns

### Recommended Project Structure
No new files or directories. Changes are confined to three existing files:
```
json-scada/src/
├── AdminUI/src/components/S7PlusAlarmsViewerPage.vue    # Ack button + pending state
├── server_realtime_auth/index.js                        # POST endpoint (lines ~370 area)
└── S7CommPlusClient/MongoCommands.cs                    # Ack branch in ProcessMongoCmd()

.planning/phases/04-ack-write-back/
└── 04-ACK-SPIKE.md                                      # Spike deliverable (created first)
```

### Pattern 1: SetVariableRequest for Alarm Ack (Spike-gated)
**What:** A `SetVariableRequest` addressed to an alarm subsystem object ID, carrying the `cpuAlarmId` as the value (exact attribute ID TBD from spike). The `AlarmSubscriptionSetCreditLimit()` method in `AlarmsHandler.cs` is the closest existing example of SetVariable targeting the alarm subsystem — it sets `Ids.SubscriptionCreditLimit` on `m_AlarmSubscriptionObjectId`.

**When to use:** When `protocolSourceASDU == "s7plus-alarm-ack"` is detected in the Change Stream handler.

**Key insight from existing code (`AlarmsHandler.cs` lines 125–135):**
```csharp
// SOURCE: AlarmsHandler.cs — AlarmSubscriptionSetCreditLimit (confirmed SetVariable to alarm object)
var setVarReq = new SetVariableRequest(ProtocolVersion.V2);
setVarReq.TransportFlags = 0x74;                         // no-response flag
setVarReq.InObjectId = m_AlarmSubscriptionObjectId;      // object ID from CreateObjectResponse
setVarReq.Address = Ids.SubscriptionCreditLimit;         // LID within the object
setVarReq.Value = new ValueInt(limit);
res = SendS7plusFunctionObject(setVarReq);
```
The ack PDU will follow the same shape. The spike must determine: what `InObjectId` (alarm subsystem RID or a per-alarm object), what `Address` (attribute LID for ack), and what `Value` type/encoding carries the `cpuAlarmId`.

**Closest known object IDs from Ids.cs:**
- `Ids.NativeObjects_theAlarmSubsystem_Rid = 8` — the alarm subsystem root object
- `Ids.AlarmSubsystem_itsUpdateRelevantDAI = 2667` — used in GetActiveAlarms ExploreRequest
- `Ids.DAI_CPUAlarmID = 2670` — the field holding cpuAlarmId in a DAI object

### Pattern 2: commandsQueue insertOne for Alarm Ack
**What:** POST endpoint inserts a document with `protocolSourceASDU: "s7plus-alarm-ack"` (as a plain string — NOT `new Double(...)` like normal commands) and `protocolSourceObjectAddress` holding the `cpuAlarmId` string.

**Source: `index.js` lines 350–369 (listS7PlusAlarms) and lines 1135–1155 (tag write insertOne)**

```javascript
// SOURCE: index.js AUTHENTICATION block, inline endpoint pattern
app.use(
  OPCAPI_AP + 'auth/ackS7PlusAlarm',
  [authJwt.isAdmin],
  async (req, res) => {
    try {
      if (!db) return res.status(200).send({ error: 'DB not connected' })
      const { cpuAlarmId, connectionNumber } = req.body
      if (!cpuAlarmId || !connectionNumber) {
        return res.status(400).send({ error: 'Missing cpuAlarmId or connectionNumber' })
      }
      const result = await db.collection('commandsQueue').insertOne({
        protocolSourceConnectionNumber: new Double(connectionNumber),
        protocolSourceObjectAddress: cpuAlarmId.toString(),   // string — not Double
        protocolSourceASDU: 's7plus-alarm-ack',               // string discriminator
        protocolSourceCommandDuration: new Double(0),
        protocolSourceCommandUseSBO: false,
        pointKey: new Double(0),
        tag: '',
        timeTag: new Date(),
        value: new Double(0),
        valueString: '',
        originatorUserName: username,
        originatorIpAddress: req.headers['x-real-ip'] ||
                             req.headers['x-forwarded-for'] ||
                             req.socket.remoteAddress,
      })
      if (!result.acknowledged) return res.status(500).send({ error: 'Insert failed' })
      res.status(200).send({ ok: true })
    } catch (err) {
      Log.log(err)
      res.status(200).send({ error: err.message })
    }
  }
)
```

### Pattern 3: MongoCommands.cs Ack Branch
**What:** Inside `ProcessMongoCmd()` `ForEachAsync` lambda, add an early `if` branch before the existing `AddressCache.TryGetValue` block (which would fail for ack commands since `cpuAlarmId` is not a tag address). The branch detects `"s7plus-alarm-ack"`, extracts `cpuAlarmId`, and calls the prototype method from the spike doc.

**Source: `MongoCommands.cs` lines 91–160**

```csharp
// SOURCE: MongoCommands.cs — existing ASDU extraction at line 92
string asdu = change.FullDocument.protocolSourceASDU.ToString();

// Insert ack branch BEFORE the AddressCache.TryGetValue block (line 97)
if (asdu == "s7plus-alarm-ack")
{
    string cpuAlarmIdStr = change.FullDocument.protocolSourceObjectAddress.ToString();
    // ... parse cpuAlarmId, call SendAlarmAck(srv, cpuAlarmId) from spike prototype
    // ... update commandsQueue as delivered (same pattern as lines 150-159)
    break;
}
```

**Critical:** The ack must use the **main connection** (`srv.connection`) not the alarm-subscription connection (`alarmConn` in AlarmThread.cs). The alarm subscription connection is a separate `S7CommPlusConnection` instance managed entirely within the `AlarmThread` method — it is not accessible from `ProcessMongoCmd`. The main connection (`srv.connection`) is the one used for tag reads/writes and is accessible here.

### Pattern 4: Vue Pending State Tracking
**What:** A `ref(new Set())` of `cpuAlarmId` strings currently awaiting ack confirmation. The ackState column template renders spinner+disabled for IDs in the set, Ack button otherwise (when `!item.ackState`).

```javascript
// SOURCE: Vue 3 Composition API — ref + Set pattern
const pendingAcks = ref(new Set())

// In ackState column template slot:
// if item.ackState === true → mdi-check (unchanged)
// if item.ackState === false && pendingAcks.has(item.cpuAlarmId) → spinner, disabled
// if item.ackState === false && !pendingAcks.has(item.cpuAlarmId) → Ack button

async function ackAlarm(cpuAlarmId, connectionNumber) {
  pendingAcks.value = new Set([...pendingAcks.value, cpuAlarmId]) // trigger reactivity
  try {
    const response = await fetch('/Invoke/auth/ackS7PlusAlarm', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
      body: JSON.stringify({ cpuAlarmId, connectionNumber }),
    })
    if (!response.ok) throw new Error('POST failed')
    // Pending state stays until next poll confirms ackState: true
  } catch (err) {
    console.warn('Ack failed:', err)
    pendingAcks.value.delete(cpuAlarmId)
    pendingAcks.value = new Set(pendingAcks.value) // trigger reactivity
  }
}
```

**Pending resolution in `fetchAlarms`:** After `alarms.value = json`, scan for IDs in `pendingAcks` whose alarm now shows `ackState: true` and remove them. Also remove any pending ID where alarm now shows `ackState: false` (PLC rejected — allow retry).

### Pattern 5: Spike Document Structure
The spike doc at `.planning/phases/04-ack-write-back/04-ACK-SPIKE.md` must contain:
1. Wireshark capture summary — raw bytes of `SetVariableRequest` PDU (ack direction TIA Portal → PLC)
2. Decoded fields table: `InObjectId` value, `Address` (LID) value, `Value` type and encoding, `TransportFlags`
3. Prototype C# method — standalone, not wired to commandsQueue
4. Confirmation: "prototype tested against live PLC, result = 0"

### Anti-Patterns to Avoid
- **Don't call AlarmSubscriptionAck()** — no such method exists in S7CommPlusDriver; the ack is a SetVariableRequest, not a subscription operation
- **Don't use the alarm connection for ack** — `alarmConn` in AlarmThread is a separate `S7CommPlusConnection` instance; ack must go through `srv.connection` (the main connection) in `ProcessMongoCmd`
- **Don't insert ack into AddressCache path** — the existing `TryGetValue(address, out ItemAddress)` code is for tag writes only; jump over it with the `if (asdu == "s7plus-alarm-ack")` branch
- **Don't use `protocolSourceASDU: new Double(...)` for the ack command** — the tag write path converts ASDU to Double for numeric types; the ack discriminator must stay as a string; the C# `rtCommand` class reads it as `BsonString protocolSourceASDU`
- **Don't mutate a Vue reactive Set in-place without triggering reactivity** — `pendingAcks.value.add(id)` alone won't trigger template re-render; always reassign `pendingAcks.value = new Set(...)` or use a `reactive()` object

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| PLC communication | Custom socket/TPKT framing | S7CommPlusDriver `SetVariableRequest` + `SendS7plusFunctionObject` | TLS, sequence counters, integrity IDs, VLQ encoding all handled |
| Auth on Express endpoint | Custom JWT check | `[authJwt.isAdmin]` middleware array (already used on listS7PlusAlarms) | Consistent with all other auth endpoints |
| Pending state UI feedback | Custom loading overlay | `v-progress-circular` (Vuetify, already in AdminUI) | Zero dependency cost |
| Command queuing | Custom async channel | `commandsQueue` MongoDB collection + Change Stream (already fully wired) | Pattern proven across all protocol drivers |

**Key insight:** The alarm ack PDU payload format is the only unknown. Everything else is a combination of existing patterns already proven in this codebase.

---

## Common Pitfalls

### Pitfall 1: Using Wrong Connection for Ack
**What goes wrong:** Developer passes `alarmConn` (the alarm subscription connection) to the ack method instead of `srv.connection`.
**Why it happens:** AlarmThread creates its own `S7CommPlusConnection` for subscription receive loop; easy to conflate with the main connection.
**How to avoid:** The ack method receives `srv` (the `S7CP_connection` struct) and uses `srv.connection`. `alarmConn` is a local variable inside `AlarmThread` and is not accessible from `ProcessMongoCmd`.
**Warning signs:** `NullReferenceException` on `alarmConn` reference from outside `AlarmThread`.

### Pitfall 2: Malformed Ack PDU Crashing Alarm Subscription
**What goes wrong:** An incorrect `InObjectId` or `Address` in the `SetVariableRequest` causes the PLC to return a protocol error that corrupts the receive state machine, causing `AlarmThread` to exit and alarm events to stop arriving.
**Why it happens:** The S7CommPlus protocol state is shared; a bad response PDU can leave the parser in an inconsistent state.
**How to avoid:** Prototype the PDU in isolation (separate `S7CommPlusConnection`, not the live subscription connection) and confirm `res == 0` before wiring to the production pipeline. This is exactly what the spike prototype requirement enforces.
**Warning signs:** AlarmThread logs "receive error" and exits after an ack command is sent.

### Pitfall 3: cpuAlarmId Type Mismatch
**What goes wrong:** `cpuAlarmId` is a `ulong` (64-bit) in C# but MongoDB stores it as a string (see `BuildAlarmDocument`: `{ "cpuAlarmId", dai.CpuAlarmId.ToString() }`). The Vue component sees it as a string. The POST body should send it as a string. The C# branch reads `protocolSourceObjectAddress.ToString()` and parses to `ulong`.
**Why it happens:** MongoDB BSON does not have a native uint64; the string representation was chosen. Forgetting to parse it back to `ulong` in C# causes the PLC to receive the wrong alarm ID.
**How to avoid:** In the C# ack branch: `ulong cpuAlarmId = ulong.Parse(change.FullDocument.protocolSourceObjectAddress.ToString())`.
**Warning signs:** PLC returns error response with no matching alarm; ack appears to succeed (res == 0) but alarm remains unacknowledged in MongoDB.

### Pitfall 4: Vue Reactivity Not Triggered on Set Mutation
**What goes wrong:** Button does not switch to spinner after click; spinner does not clear after poll update.
**Why it happens:** Vue 3 `ref` wrapping a `Set` does not detect `.add()` / `.delete()` mutations.
**How to avoid:** Always replace `pendingAcks.value = new Set([...pendingAcks.value, id])` for add, and `new Set([...pendingAcks.value].filter(x => x !== id))` for delete.
**Warning signs:** Button stays enabled after click; no visual feedback.

### Pitfall 5: Pending State Never Cleared on Ack Timeout
**What goes wrong:** POST succeeds but PLC silently ignores the ack (no error returned); `ackState` never becomes `true`; spinner persists forever.
**Why it happens:** The "wait for poll to confirm" approach has no timeout.
**How to avoid:** The CONTEXT.md decision handles this: "If poll returns but `ackState` is still `false`: exit pending state and re-enable the Ack button." This timeout-by-next-poll design means: on every `fetchAlarms()` completion, remove from `pendingAcks` any ID whose alarm still shows `ackState: false` (retry is enabled). This is a PoC-appropriate approach.
**Warning signs:** After one failed ack cycle, the pending Set grows unboundedly.

### Pitfall 6: commandsQueue ASDU as Double vs String
**What goes wrong:** Inserting `protocolSourceASDU: new Double(0)` instead of `protocolSourceASDU: 's7plus-alarm-ack'` causes the C# branch to never match (it does `asdu.ToString() == "s7plus-alarm-ack"` which would see `"0"`).
**Why it happens:** The existing tag write path uses numeric ASDUs for most types; the ack discriminator is intentionally a string.
**How to avoid:** Insert `protocolSourceASDU: 's7plus-alarm-ack'` as a plain JS string. The `rtCommand.protocolSourceASDU` is a `BsonString` in C#, so it reads both string and numeric BSON values via the ToString() call.
**Warning signs:** C# logs "MongoDB CMD CS - Address not found in cache" for the ack command, meaning it fell through to the tag-write path.

---

## Code Examples

### SetVariableRequest Pattern (from existing AlarmSubscriptionSetCreditLimit)
```csharp
// Source: S7CommPlusDriver/Alarming/AlarmsHandler.cs lines 125-135
// This is the TEMPLATE for the alarm ack PDU — spike fills in the unknowns
var setVarReq = new SetVariableRequest(ProtocolVersion.V2);
setVarReq.TransportFlags = 0x74;             // 0x74 = no-response-required; 0x34 = response-required
setVarReq.InObjectId = /* TBD from spike */; // alarm subsystem object RID or per-alarm object ID
setVarReq.Address    = /* TBD from spike */; // LID of the "ack this alarm" attribute
setVarReq.Value      = /* TBD from spike */; // likely ValueLWord(cpuAlarmId) or similar
int res = SendS7plusFunctionObject(setVarReq);
```

### SetPlcOperatingState as Full SetVariable Example (Response-Required Path)
```csharp
// Source: S7CommPlusConnection.cs lines 714-745
// Shows the full send+receive+response-check pattern when TransportFlags = 0x34
var setVarReq = new SetVariableRequest(ProtocolVersion.V2);
setVarReq.InObjectId = Ids.NativeObjects_theCPUexecUnit_Rid;  // native object
setVarReq.Address    = Ids.CPUexecUnit_operatingStateReq;      // LID within object
setVarReq.Value      = new ValueDInt(state);
res = SendS7plusFunctionObject(setVarReq);
if (res != 0) { m_client.Disconnect(); return res; }
m_LastError = 0;
WaitForNewS7plusReceived(m_ReadTimeout);
var setVarRes = SetVariableResponse.DeserializeFromPdu(m_ReceivedPDU);
```

### listS7PlusAlarms Endpoint (Template for ackS7PlusAlarm Endpoint)
```javascript
// Source: index.js lines 350-369
app.use(
  OPCAPI_AP + 'auth/listS7PlusAlarms',
  [authJwt.isAdmin],
  async (req, res) => {
    try {
      if (!db) return res.status(200).send({ error: 'DB not connected' })
      const docs = await db
        .collection('s7plusAlarmEvents')
        .find({}, { projection: { _id: 0 } })
        .sort({ createdAt: -1 })
        .limit(200)
        .toArray()
      res.status(200).send(docs)
    } catch (err) {
      Log.log(err)
      res.status(200).send({ error: err.message })
    }
  }
)
```

### commandsQueue insertOne Pattern (From Tag Write Endpoint)
```javascript
// Source: index.js lines 1135-1155
let result = await db.collection(COLL_COMMANDS).insertOne({
  protocolSourceConnectionNumber: new Double(data.protocolSourceConnectionNumber),
  protocolSourceObjectAddress: data.protocolSourceObjectAddress,  // string for ack
  protocolSourceASDU: data.protocolSourceASDU,                    // 's7plus-alarm-ack' for ack
  protocolSourceCommandDuration: new Double(0),
  protocolSourceCommandUseSBO: false,
  pointKey: new Double(0),
  tag: '',
  timeTag: new Date(),
  value: new Double(0),
  valueString: '',
  originatorUserName: username,
  originatorIpAddress: req.headers['x-real-ip'] || req.headers['x-forwarded-for'] || req.socket.remoteAddress,
})
```

### Vue Pending State Reactivity (Set-based approach)
```javascript
// Source: Vue 3 Composition API pattern — reactivity-safe Set mutation
const pendingAcks = ref(new Set())

// Add:
pendingAcks.value = new Set([...pendingAcks.value, cpuAlarmId])

// Remove (on success or failure):
pendingAcks.value = new Set([...pendingAcks.value].filter(id => id !== cpuAlarmId))

// Template check:
// pendingAcks.value.has(item.cpuAlarmId)
```

---

## State of the Art

| Old Approach | Current Approach | Impact |
|--------------|------------------|--------|
| Custom alarm write logic | Reuse commandsQueue pattern + ASDU discriminator | Zero new infrastructure; consistent audit trail |
| Polling for ack confirmation | Piggyback on existing 5000ms setInterval | No extra fetch; PoC-appropriate simplicity |

---

## Open Questions

1. **Exact SetVariableRequest fields for alarm ack**
   - What we know: PDU is a SetVariableRequest targeting the alarm subsystem; `AlarmSubscriptionSetCreditLimit` in AlarmsHandler.cs uses SetVariable on `m_AlarmSubscriptionObjectId` with `Ids.SubscriptionCreditLimit`; `Ids.NativeObjects_theAlarmSubsystem_Rid = 8` is the root alarm subsystem object
   - What's unclear: Whether ack targets the subsystem root object (RID 8) or a per-alarm DAI object; which LID is the "acknowledge" attribute; what PValue type carries the cpuAlarmId
   - Recommendation: This is the spike deliverable. Do not guess. Capture Wireshark traffic of TIA Portal acknowledging an alarm, decode the SetVariableRequest fields, verify prototype returns res==0 against live PLC before wiring.

2. **JWT token availability in Vue SPA for ack POST**
   - What we know: Existing `fetchAlarms()` uses `fetch('/Invoke/auth/listS7PlusAlarms')` with no explicit Authorization header shown in the component code; AdminUI likely stores JWT in a Pinia store or localStorage
   - What's unclear: Whether the fetch pattern in AdminUI passes JWT automatically (cookie-based) or requires explicit header
   - Recommendation: Check how other authenticated fetches work in AdminUI (e.g., other `fetch('/Invoke/auth/...')` calls). If cookies carry the JWT, no header needed. If Bearer token required, get it from the same store used elsewhere.

3. **connectionId value for the ack command**
   - What we know: Each alarm document has `connectionId: srv.protocolConnectionNumber` (an integer); the Vue component has `item.connectionId` available in the row
   - What's unclear: The Vue component's `headers` array has `key: 'connectionId'` with title 'Source' — confirm the value passed to the POST body is the raw integer, not a display string
   - Recommendation: Pass `item.connectionId` directly from the row as the `connectionNumber` in the POST body.

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | None detected — PoC project has no automated test suite |
| Config file | None |
| Quick run command | Manual: build `dotnet build` + run + inspect logs |
| Full suite command | Manual: end-to-end with live PLC |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DRVR-03 | PLC receives and applies ack command | manual-only (requires live PLC + Wireshark) | n/a — spike Wireshark capture is the artifact | ❌ No automated test possible |
| VIEW-03 | Operator clicks Ack, UI shows pending, then resolves | manual-only (requires running AdminUI + PLC) | n/a | ❌ No automated test possible |

**Justification for manual-only:** Both requirements require a live S7-1200/S7-1500 PLC running PLCSIM or physical hardware. There is no test harness or mock PLC in this PoC project.

### Sampling Rate
- **Per task commit:** Manual smoke test — start driver, trigger alarm on PLC, click Ack in AdminUI, verify MongoDB `ackState` becomes `true` within one poll cycle
- **Per wave merge:** Full round-trip test as above; also verify Ack button state machine (pending → resolved / pending → retry)
- **Phase gate:** Wireshark capture doc committed + prototype confirmed + full round-trip before `/gsd:verify-work`

### Wave 0 Gaps
None for test infrastructure — there is none to set up. The spike doc is the first Wave 0 deliverable.

---

## Sources

### Primary (HIGH confidence)
- `json-scada/src/S7CommPlusClient/MongoCommands.cs` — Change Stream handler; ProcessMongoCmd full implementation; rtCommand class definition; ASDU string extraction pattern
- `json-scada/src/S7CommPlusClient/AlarmThread.cs` — alarm subscription connection lifecycle; AlarmThread receives alarms on separate S7CommPlusConnection
- `json-scada/src/S7CommPlusClient/Common.cs` — S7CP_connection struct; `srv.connection` vs alarm connection distinction
- `json-scada/src/server_realtime_auth/index.js` lines 350–369 — listS7PlusAlarms endpoint template; inline AUTHENTICATION block pattern; `authJwt.isAdmin` middleware
- `json-scada/src/server_realtime_auth/index.js` lines 1135–1155 — commandsQueue insertOne field names and BSON types
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` — existing component; ackState column slot; fetchAlarms with setInterval; alarms ref structure
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHandler.cs` — `AlarmSubscriptionSetCreditLimit`: confirmed SetVariableRequest pattern for alarm subsystem
- `S7CommPlusDriver/src/S7CommPlusDriver/S7CommPlusConnection.cs` lines 714–745 — `SetPlcOperatingState`: confirmed full SetVariableRequest send+receive pattern
- `S7CommPlusDriver/src/S7CommPlusDriver/Core/SetVariableRequest.cs` — complete SetVariableRequest field layout (InObjectId, Address, Value, TransportFlags)
- `S7CommPlusDriver/src/S7CommPlusDriver/Core/Ids.cs` — all named constant IDs; `NativeObjects_theAlarmSubsystem_Rid = 8`, `DAI_CPUAlarmID = 2670`
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsDai.cs` — `CpuAlarmId` is `ulong`; stored as string in MongoDB

### Secondary (MEDIUM confidence)
- S7CommPlus protocol community knowledge (Wireshark dissector, s7commplus Wireshark plugin): alarm ack likely uses SetVariable with DAI object as target — not independently verified; must be confirmed by spike

### Tertiary (LOW confidence)
- No web research performed — all findings from local codebase inspection; no external sources needed

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all libraries verified present in codebase; no new dependencies
- Architecture patterns (all except spike): HIGH — direct code references cited
- Spike PDU fields: LOW — unknown; this is the explicit open question; spike exists precisely because it cannot be determined from static analysis alone
- Common pitfalls: HIGH — derived from actual code structure (wrong connection, ASDU type mismatch, Vue reactivity)

**Research date:** 2026-03-19
**Valid until:** 2026-06-01 (stable codebase; S7CommPlusDriver protocol is not changing)

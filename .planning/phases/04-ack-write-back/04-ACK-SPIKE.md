# Alarm Acknowledgement PDU Spike

**Date:** 2026-03-18
**PLC:** S7-1500 (PLCSIM Advanced simulation)
**Status:** Prototype constructed from DLL analysis + AckJob notification decode. PLCSIM test run required to validate.

---

## Section 1: Wireshark Capture Summary

### Capture Setup

| Field | Value |
|-------|-------|
| Date | 2026-03-18 |
| PLC model | S7-1500 (PLCSIM Advanced) |
| Alarm acknowledged | CPUAlarmID = `0x8a0e000200010000` |
| TLS session key log | Available for S7CommPlusClient only (via `SSLKEYLOG` env var) |

### What Was Captured

**Frame 995 — TIA Portal to PLCSIM (encrypted)**
- PDU type: TLSv1.3 application data, 105 bytes payload
- Decoding: NOT POSSIBLE — no TIA Portal TLS session keys available
- Conclusion: The ack PDU sent by TIA Portal cannot be directly decoded from this capture.

**Frame 1000 — PLCSIM to S7CommPlusClient (decoded)**
- PDU type: S7CommPlus Notification (AckJob completion)
- Fully decodable because S7CommPlusClient's TLS session keys are available

Decoded AckJob notification from PLCSIM:

```
AckJob.Class_Rid / DynObjX7.0.28673
  AlarmingJob.AlarmJobState = 2 (completed)
  AlarmingJob.JobTimestamp   = 2026-03-18T14:27:48.734Z
  AckJob.AcknowledgementList = Addressarray Struct (AS_AlarmAcknfyElem.Class_Rid)
    [0] AS_AlarmAcknfyElem.CPUAlarmID    = 0x8a0e000200010000
        AS_AlarmAcknfyElem.AllStatesInfo = 2
        AS_AlarmAcknfyElem.AckResult     = 0 (success)
```

### Key Finding: CreateObject, NOT SetVariable

The initial plan hypothesis (`SetVariableRequest` targeting an alarm subsystem attribute) was **superseded** by DLL analysis of the Wireshark s7comm_plus dissector DLL combined with the AckJob notification decode.

**Why CreateObject, not SetVariableRequest:**

1. The AckJob notification response carries a **PLC-assigned dynamic RelId** (`DynObjX7.0.28673`).
2. Dynamic RelIds of the form `DynObjX7.x.y` are only assigned by `CreateObject` — the PLC creates a new transient object and returns its ID in the `CreateObjectResponse`.
3. `SetVariableRequest` writes to *existing* attributes on existing objects; it cannot cause the PLC to create a new object with a fresh dynamic ID.
4. The `AlarmSubscriptionCreate` pattern in `AlarmsHandler.cs` follows exactly this same flow: `CreateObjectRequest` → PLC assigns `DynObjX7.x.y` → `CreateObjectResponse` returns the new object ID.
5. The AckJob object mirrors the subscription object shape: `CreateObject` creates it, the PLC processes it, and the completion notification arrives asynchronously.

**InObjectId equivalent in CreateObject:** `CreateObjectRequest` does not have an `InObjectId` field. The equivalent routing parameter is `RequestId = 8` (`Ids.NativeObjects_theAlarmSubsystem_Rid`) — this is the parent object under which the AckJob is created.

---

## Section 2: Decoded Fields

### Class and Attribute IDs (from Wireshark s7comm_plus.dll value_string tables)

The following IDs were extracted by scanning the DLL's value_string tables and cross-validated against the AckJob notification decode (e.g., `AS_AlarmAcknfyElem.Class_Rid = 7604` matched the notification's struct type tag).

| Symbol | Hex ID | Decimal ID | Role |
|--------|--------|-----------|------|
| `AckJob.Class_Rid` | `0x0E34` | **3636** | ClassId of the AckJob object to create |
| `AlarmingJob.Application` | `0x0E35` | 3637 | Application name string (optional) |
| `AlarmingJob.Host` | `0x0E36` | 3638 | Hostname string (optional) |
| `AlarmingJob.User` | `0x0E37` | 3639 | Username string (optional) |
| `AlarmingJob.Class_Rid` | `0x0E38` | 3640 | Base class for AckJob |
| `AlarmingJob.AlarmJobState` | `0x0E39` | **3641** | Job state (0=pending, 2=completed) — read from response |
| `AlarmingJob.JobTimestamp` | `0x0E3A` | 3642 | Completion timestamp — read from response |
| `AlarmSubsystem.itsAckJob` | `0x0E3D` | **3645** | Composition AttributeId: places AckJob under AlarmSubsystem |
| `AckJob.AcknowledgementList` | `0x0E3F` | **3647** | The list of alarms to acknowledge |
| `AS_AlarmAcknfyElem.AllStatesInfo` | `0x1D9C` | **7596** | Ack state bitmask (value 2 = ack) |
| `AS_AlarmAcknfyElem.AckResult` | `0x1DAD` | 7597 | Result per element (0 = success) — read from response |
| `AS_AlarmAcknfyElem.CPUAlarmID` | `0x1DAE` | **7598** | The CPUAlarmID to acknowledge |
| `AS_AlarmAcknfyElem.Class_Rid` | `0x1DB4` | **7604** | ClassId of each acknowledgement list element |

### CreateObjectRequest Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| `TransportFlags` | `0x34` | Response required — needed to receive job completion |
| `RequestId` | `8` (`Ids.NativeObjects_theAlarmSubsystem_Rid`) | Parent object; InObjectId equivalent for CreateObject |
| `RequestValue` | `new ValueUDInt(0)` | Matches AlarmSubscriptionCreate pattern |
| `RequestObject.ClassId` | `3636` (`AckJob.Class_Rid`) | Type of the object being created |
| `RequestObject.RelationId` | `211` (`Ids.GetNewRIDOnServer`) | PLC assigns a fresh dynamic RelId |
| `RequestObject.AttributeId` | `3645` (`AlarmSubsystem.itsAckJob`) | Composition path: places object under AlarmSubsystem |

### AcknowledgementList Element (AS_AlarmAcknfyElem)

| Attribute | ID | Value | Notes |
|-----------|----|-------|-------|
| `CPUAlarmID` | `7598` | `new ValueLWord(cpuAlarmId)` | The ulong alarm ID to acknowledge |
| `AllStatesInfo` | `7596` | `new ValueUSInt(2)` | 2 = request acknowledgement |

**Encoding note:** The AcknowledgementList is typed as an Addressarray Struct in the notification response. For the request, the list element is modeled as a child `PObject` added to the AckJob object via `AddObject`. This mirrors how `AlarmSubscriptionRef` is a child PObject nested inside the Subscription PObject in `AlarmSubscriptionCreate`. The encoding requires PLCSIM verification — it is the single most uncertain step in the prototype.

---

## Section 3: Prototype C# Method

### Context: Why CreateObject, not SetVariableRequest

The original plan referenced `SetVariableRequest` as the expected PDU type based on the `AlarmSubscriptionSetCreditLimit` pattern. Static analysis and the Wireshark AckJob notification decode show the actual mechanism is `CreateObjectRequest`. The `InObjectId` concept from `SetVariableRequest` maps to `RequestId = 8` in `CreateObjectRequest` — both route the PDU to the alarm subsystem root object.

For reference, the `SetVariableRequest` shape (which is NOT used for alarm ack) is:
```csharp
// SetVariableRequest shape — used for credit limit, NOT for alarm ack
// InObjectId routes to an existing object; Address selects the attribute to write
var setVarReq = new SetVariableRequest(ProtocolVersion.V2);
setVarReq.InObjectId = m_AlarmSubscriptionObjectId;
setVarReq.Address    = Ids.SubscriptionCreditLimit;
setVarReq.Value      = new ValueInt(limit);
```

The alarm ack instead creates a transient `AckJob` object on the PLC using `CreateObjectRequest`, analogous to how `AlarmSubscriptionCreate` works.

### Prototype Method

```csharp
/// <summary>
/// Sends an alarm acknowledgement to the PLC by creating a transient AckJob object
/// under the AlarmSubsystem (RequestId = NativeObjects_theAlarmSubsystem_Rid = 8).
///
/// Based on:
///   - AlarmSubscriptionCreate() pattern in AlarmsHandler.cs (CreateObjectRequest shape)
///   - Wireshark DLL analysis (AckJob.Class_Rid=3636, AckJob.AcknowledgementList=3647,
///     AS_AlarmAcknfyElem.Class_Rid=7604, CPUAlarmID attr=7598, AllStatesInfo attr=7596)
///   - AckJob notification decode confirming DynObjX7 RelId assignment (CreateObject marker)
///
/// VERIFICATION STATUS: NOT YET TESTED against live PLC. Requires PLCSIM test run.
/// The AcknowledgementList encoding as a child PObject is the primary uncertainty.
/// </summary>
/// <param name="connection">The main S7CommPlusConnection (srv.connection — NOT the alarm subscription connection)</param>
/// <param name="cpuAlarmId">The CPUAlarmID of the alarm to acknowledge (from AlarmsDai.CpuAlarmId)</param>
/// <returns>0 on success, non-zero on failure</returns>
public static int SendAlarmAck(S7CommPlusConnection connection, ulong cpuAlarmId)
{
    int res;

    // --- IDs extracted from Wireshark s7comm_plus.dll DLL analysis ---
    const uint AckJob_Class_Rid              = 3636; // 0x0E34
    const uint AlarmSubsystem_itsAckJob      = 3645; // 0x0E3D — composition AttributeId
    const uint AckJob_AcknowledgementList    = 3647; // 0x0E3F — not used as attribute; child object is the list element
    const uint AS_AlarmAcknfyElem_Class_Rid  = 7604; // 0x1DB4
    const uint AS_AlarmAcknfyElem_CPUAlarmID = 7598; // 0x1DAE
    const uint AS_AlarmAcknfyElem_AllStatesInfo = 7596; // 0x1D9C

    // --- Build the acknowledgement list element ---
    // UNCERTAINTY: The list element is modeled as a child PObject nested inside the AckJob object.
    // This mirrors the AlarmSubscriptionRef child pattern in AlarmSubscriptionCreate().
    // An alternative encoding (Addressarray attribute) may be needed — must be verified with PLCSIM.
    var ackElem = new PObject();
    ackElem.ClassId     = AS_AlarmAcknfyElem_Class_Rid;   // 7604
    ackElem.RelationId  = Ids.GetNewRIDOnServer;           // 211 — PLC assigns dynamic RelId
    ackElem.AttributeId = AckJob_AcknowledgementList;      // 3647 — composition path
    ackElem.AddAttribute(AS_AlarmAcknfyElem_CPUAlarmID,    new ValueLWord(cpuAlarmId)); // 7598
    ackElem.AddAttribute(AS_AlarmAcknfyElem_AllStatesInfo, new ValueUSInt(2));           // 7596 — 2 = ack

    // --- Build the AckJob object ---
    var ackJobObj = new PObject();
    ackJobObj.ClassId     = AckJob_Class_Rid;           // 3636
    ackJobObj.RelationId  = Ids.GetNewRIDOnServer;      // 211 — PLC assigns new dynamic RelId (DynObjX7.x.y)
    ackJobObj.AttributeId = AlarmSubsystem_itsAckJob;   // 3645 — composition path under AlarmSubsystem
    ackJobObj.AddObject(ackElem);                        // Nest the element inside the job

    // --- Build the CreateObjectRequest ---
    // RequestId = 8 (NativeObjects_theAlarmSubsystem_Rid) is the InObjectId equivalent.
    // TransportFlags = 0x34 (response required) so we can read the CreateObjectResponse.
    var createObjReq = new CreateObjectRequest(ProtocolVersion.V2, 0, true);
    createObjReq.TransportFlags = 0x34;
    createObjReq.RequestId      = Ids.NativeObjects_theAlarmSubsystem_Rid; // 8
    createObjReq.RequestValue   = new ValueUDInt(0);
    createObjReq.SetRequestObject(ackJobObj);

    // --- Send ---
    res = connection.SendS7plusFunctionObject(createObjReq);
    if (res != 0)
    {
        Console.WriteLine($"SendAlarmAck: SendS7plusFunctionObject failed, res=0x{res:X8}");
        return res;
    }

    // --- Wait for response ---
    connection.m_LastError = 0;
    connection.WaitForNewS7plusReceived(connection.m_ReadTimeout);
    if (connection.m_LastError != 0)
    {
        Console.WriteLine($"SendAlarmAck: WaitForNewS7plusReceived failed, error=0x{connection.m_LastError:X8}");
        return connection.m_LastError;
    }

    // --- Deserialize CreateObjectResponse ---
    var createObjRes = CreateObjectResponse.DeserializeFromPdu(connection.m_ReceivedPDU);
    if (createObjRes == null)
    {
        Console.WriteLine("SendAlarmAck: CreateObjectResponse is null — invalid PDU");
        return S7Consts.errIsoInvalidPDU;
    }

    if (createObjRes.ReturnValue != 0)
    {
        Console.WriteLine($"SendAlarmAck: PLC returned error ReturnValue=0x{createObjRes.ReturnValue:X8} for cpuAlarmId=0x{cpuAlarmId:X16}");
        return S7Consts.errCliInvalidParams;
    }

    Console.WriteLine($"SendAlarmAck: cpuAlarmId=0x{cpuAlarmId:X16} acknowledged successfully (res == 0)");
    return 0;
}
```

### Integration Note

When wiring to `MongoCommands.cs` (Plan 02), the method will be called from inside the `ForEachAsync` Change Stream lambda:

```csharp
// In ProcessMongoCmd() — before the AddressCache.TryGetValue block
string asdu = change.FullDocument.protocolSourceASDU.ToString();
if (asdu == "s7plus-alarm-ack")
{
    ulong cpuAlarmId = ulong.Parse(
        change.FullDocument.protocolSourceObjectAddress.ToString());
    int res = SendAlarmAck(srv.connection, cpuAlarmId);
    // ... update commandsQueue as delivered ...
    break;
}
```

---

## Section 4: Verification

| Check | Status |
|-------|--------|
| Prototype tested against live PLC | **NOT YET — requires PLCSIM test run** |
| Alarm `cpuAlarmId` acknowledged successfully | **PENDING VERIFICATION** |
| `CreateObjectResponse.ReturnValue == 0` | PENDING |
| AckJob notification received confirming completion | PENDING |

### Next Steps for Verification

1. Start S7CommPlusClient with PLCSIM running and an active unacknowledged alarm.
2. Call `SendAlarmAck(srv.connection, cpuAlarmId)` from a test harness (or temporarily from `AlarmSubscriptionCreate`'s callback path).
3. Observe the `CreateObjectResponse.ReturnValue` — expected: `0`.
4. Check PLCSIM's alarm table — the alarm should move to acknowledged state.
5. If `ReturnValue != 0`: inspect the error code. Common failure modes:
   - Wrong `AcknowledgementList` encoding → try as a flat Addressarray attribute instead of child PObject
   - Wrong `AttributeId` on the `ackElem` PObject → try `AttributeId = 0` (no composition)
   - Wrong `ClassId` on `ackJobObj` → verify against live capture once TIA Portal TLS keys are available

### Fallback Encoding

If the child-PObject approach fails, the alternative is to encode the AcknowledgementList as a flat `ValueULWordArray` attribute directly on the AckJob object:

```csharp
// Fallback: flat array attribute instead of child PObject
ackJobObj.AddAttribute(AckJob_AcknowledgementList,
    new ValueULWordArray(new ulong[] { cpuAlarmId }, 0x20)); // 0x20 = Addressarray
```

This must be tested against PLCSIM. The child-PObject approach is tried first because it matches the decoded `AS_AlarmAcknfyElem.Class_Rid` struct type visible in the AckJob notification.

---

## Appendix: Protocol Comparison

| Mechanism | AlarmSubscriptionCreate | SendAlarmAck (this spike) |
|-----------|------------------------|--------------------------|
| PDU type | `CreateObjectRequest` | `CreateObjectRequest` |
| `RequestId` | `SessionId2` | `8` (AlarmSubsystem) |
| `RequestValue` | `ValueUDInt(0)` | `ValueUDInt(0)` |
| `TransportFlags` | `0x34` (response required) | `0x34` (response required) |
| Object `ClassId` | `Ids.ClassSubscription` (1001) | `AckJob.Class_Rid` (3636) |
| Object `RelationId` | `m_AlarmSubscriptionRelationId` | `Ids.GetNewRIDOnServer` (211) |
| Object `AttributeId` | — (0) | `AlarmSubsystem.itsAckJob` (3645) |
| Child object | `AlarmSubscriptionRef` | `AS_AlarmAcknfyElem` |
| Response check | `CreateObjectResponse` | `CreateObjectResponse` |

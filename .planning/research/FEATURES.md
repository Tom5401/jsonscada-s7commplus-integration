# Feature Research

**Domain:** S7CommPlus native alarm subscription — PLC alarm events in json-scada
**Researched:** 2026-03-17
**Confidence:** HIGH (derived directly from protocol source code, no external sources needed)

---

## Protocol Data Inventory

Before categorizing features, here is every field the S7CommPlus protocol actually delivers in an alarm notification. This is the raw material. Everything else is a feature decision about what to do with it.

### Fields per AlarmsDai (one notification = one DAI object)

| Source | Field | Type | Meaning |
|--------|-------|------|---------|
| AlarmsDai | `CpuAlarmId` | ulong | Unique alarm identity — combination of RelationId and Alid. Stable across reconnects. |
| AlarmsDai | `AllStatesInfo` | byte | Top-level state bitmask (exact semantics partially reverse-engineered) |
| AlarmsDai | `AlarmDomain` | ushort | 1=System, 2=Security, 256..272=UserClass_0..16 |
| AlarmsDai | `MessageType` | int | 1=Alarm AP, 2=Notify AP, 3=Info Report AP, 4=Event Ack AP |
| AlarmsDai | `SequenceCounter` | uint | Monotonic counter per alarm instance; deduplication and ordering |
| AlarmsDai | `SubtypeId` | uint | 2673 (DAI_Coming) = alarm became active; 2677 (DAI_Going) = alarm cleared |
| AsCgs | `Timestamp` | DateTime | PLC-side event timestamp (millisecond resolution) |
| AsCgs | `AckTimestamp` | DateTime | PLC-side acknowledgement timestamp (zero if not yet acked) |
| AsCgs | `AllStatesInfo` | byte | Per-transition state bitmask |
| AsCgs | `AssociatedValues` | SD_1..SD_10 | Up to 10 typed process values embedded in the alarm (Bool, Int, Real, String etc.) |
| AlarmsHmiInfo | `Priority` | byte | Alarm priority as configured in TIA Portal |
| AlarmsHmiInfo | `AlarmClass` | ushort | Alarm class identifier (SyntaxId >= 258 only) |
| AlarmsHmiInfo | `GroupId` | byte | Alarm group (SyntaxId >= 258) |
| AlarmsHmiInfo | `ClientAlarmId` | uint | HMI-side alarm ID |
| AlarmsHmiInfo | `Producer` | byte | Subsystem that produced the alarm (SyntaxId >= 258) |
| AlarmsHmiInfo | `Flags` | byte | Additional flags (SyntaxId >= 258) |
| AlarmTexts | `AlarmText` | string | Primary alarm message as configured in TIA Portal |
| AlarmTexts | `Infotext` | string | Optional informational text |
| AlarmTexts | `AdditionalText1..9` | string | Up to 9 supplementary text fields |
| Notification | `Add1Timestamp` | DateTime | Driver-receive-side wall clock timestamp |
| Notification | `NotificationSequenceNumber` | uint | Protocol-level sequence for credit accounting |

### Fields from BrowseAlarms / ExploreASAlarms (alarm catalog, not per-event)

These are available via a separate browse call before subscription, not delivered in notifications:

| Source | Field | Meaning |
|--------|-------|---------|
| AlarmData / MultipleStai | `Alid` | Alarm local ID within a block |
| AlarmData / MultipleStai | `AlarmEnabled` | Whether alarm is enabled in PLC config |
| AlarmData / MultipleStai | `Lids` | List of LID addresses associated |
| AlarmData / AlText | All text fields | Pre-fetched texts (alternative to inline subscription texts) |

---

## Feature Landscape

### Table Stakes (Must Have — PoC Has No Value Without These)

Features that directly satisfy the core project requirement: "alarms from PLC appear in json-scada via native subscription with full metadata."

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Alarm state field (`coming`/`going`) | Without active/cleared distinction there is no alarm — just noise | LOW | Map `SubtypeId` directly: 2673 = `"coming"`, 2677 = `"going"`. Single enum field in MongoDB document. |
| PLC-side timestamp | Operators need to know *when the PLC saw the event*, not when the driver received it | LOW | `AsCgs.Timestamp` (DateTime). Store as UTC ISO-8601 string or BSON Date. |
| Alarm identity (`CpuAlarmId`) | Required to correlate coming/going events for the same alarm instance and to dedup | LOW | Store as string (ulong won't fit cleanly into JSON number without precision loss). |
| Primary alarm text (`AlarmText`) | The human-readable description of what went wrong — core PoC deliverable | LOW | Delivered inline when `SendAlarmTexts=true` in subscription. No extra round-trip. |
| Subscription lifecycle management | Without create/maintain/delete, the driver leaks PLC resources and loses events after reconnect | MEDIUM | `AlarmSubscriptionCreate`, credit-limit replenishment loop, `AlarmSubscriptionDelete` on disconnect. Credit mechanism in `TestWaitForAlarmNotifications` must be extracted into production loop. |
| MongoDB write on each notification | Alarms must actually land in the database — the stated deliverable | LOW | Insert one document per DAI notification into a dedicated `s7alarms` collection. |
| Driver-receive timestamp | Audit trail: know if there is latency between PLC event and driver processing | LOW | `Notification.Add1Timestamp`. Cheap to store alongside PLC timestamp. |

### Differentiators (Nice-to-Have for a Complete Implementation)

Features that improve completeness or operator usability but are not required for the PoC to demonstrate value.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Acknowledgement state | Shows whether an operator has acknowledged the alarm on the PLC | LOW | `AsCgs.AckTimestamp` is always present; zero = not acked. Store as nullable datetime or bool derived from non-zero. No extra protocol work. |
| Alarm priority | Enables filtering/sorting by severity in any downstream consumer | LOW | `HmiInfo.Priority` is a single byte. Trivial to store. Requires SyntaxId check but HmiInfo is always deserialized. |
| Alarm class | Groups alarms by configured class (e.g. "Safety", "Process") | LOW | `HmiInfo.AlarmClass` — single ushort. Available when SyntaxId >= 258 (S7-1500 and most S7-1200 firmware). |
| Alarm domain | Distinguishes system alarms (domain 1-2) from user program alarms (256+) | LOW | `AlarmsDai.AlarmDomain` already parsed. Single ushort field. |
| Sequence counter | Enables detection of missed events and ordering of rapid-fire alarms | LOW | `AlarmsDai.SequenceCounter` already parsed. Store as integer. |
| Associated values SD_1..SD_10 | Process values at alarm time — critical for diagnosis ("temperature was 342 C when alarm fired") | MEDIUM | `AsCgs.AssociatedValues` is already decoded into typed AssociatedValue objects. Serializing as a BSON array of `{type, value}` pairs is straightforward. Complexity comes from deciding how many slots to store vs. compacting nulls. |
| Infotext field | TIA Portal informational annotation — supplements AlarmText | LOW | Already in `AlarmTexts.Infotext`. Store alongside AlarmText. Zero cost. |
| AlarmText language selection | Multi-language deployments need language-aware text | LOW | Subscription accepts an LCID; driver picks one language at subscription time. Hardcode to site language for PoC. |

### Anti-Features (Explicitly Out of Scope for This PoC)

Features to deliberately not build because they are out of scope per PROJECT.md, or because they add disproportionate complexity for a PoC.

| Feature | Why Requested | Why Out of Scope | Alternative |
|---------|---------------|-----------------|-------------|
| Acknowledgement write-back to PLC | Operators want to ack from SCADA UI | Protocol requires a separate `SetVariable` command with auth; adds round-trip error handling; PROJECT.md explicitly excludes this | Read ack state passively from `AckTimestamp`; send ack via TIA Portal or dedicated HMI for now |
| AdditionalText1..9 storage | Complete alarm context from TIA Portal | 9 extra string fields bloat every document; most alarms configure zero or one additional text; marginal value for PoC | Store only `AlarmText` + `Infotext`; add additional texts as a JSON sub-array when needed |
| Multi-language text storage (all LCIDs) | Internationalization | Subscription is parameterized to one LCID at creation time; storing all languages means multiple subscription objects or post-processing; unnecessary for PoC | Choose one LCID at driver startup; configurable in connection config |
| Alarm catalog browse pre-fetch | Pre-populating alarm metadata before events arrive | `BrowseAlarms`/`ExploreASAlarms` adds startup latency and complexity; inline texts (`SendAlarmTexts=true`) deliver texts with each notification already | Use inline text delivery; skip browse path entirely |
| Deduplication / state-machine tracking | Preventing duplicate coming/going in MongoDB | Adds stateful logic to the driver; MongoDB is an event log — consumers can deduplicate | Log all events; query-side dedup if needed |
| json-scada realtimeData integration | Surfacing alarm active state as a point in the existing data model | Requires mapping CpuAlarmId to a tag; no natural key exists; blurs the boundary between event log and live state | Separate `s7alarms` collection as designed; leave realtimeData untouched |
| AdminUI or frontend changes | Making alarms visible in json-scada UI | Out of scope per PROJECT.md; data in MongoDB is the deliverable | Inspect with MongoDB Compass or a simple script |
| Redundant / HA subscription | Surviving PLC failover or driver restart cleanly | Single-connection PoC per PROJECT.md | Single reconnect attempt on disconnect is sufficient for PoC |

---

## Feature Dependencies

```
[Subscription Lifecycle]
    └──required by──> [Alarm State Field]
    └──required by──> [PLC Timestamp]
    └──required by──> [Primary Alarm Text]
    └──required by──> [MongoDB Write]

[MongoDB Write]
    └──required by──> [All stored fields]

[Alarm State Field]
    └──enables──> [Acknowledgement State] (same notification, same AsCgs struct)
    └──enables──> [Associated Values]     (same AsCgs struct)

[Alarm Text (inline)]
    └──conflicts──> [Alarm Catalog Browse] (two ways to get texts; pick one)

[Ack Write-back]
    └──conflicts──> [Read-only PoC constraint]
```

### Dependency Notes

- **Subscription Lifecycle requires credit replenishment:** The PLC stops sending notifications when the credit limit is reached. The `TestWaitForAlarmNotifications` loop in `AlarmsHandler.cs` contains the credit-replenishment logic but is implemented as a finite test loop. Production code must convert this to a continuous async loop. This is a non-obvious dependency — the subscription will silently stop delivering events without it.
- **Alarm Text inline delivery is the right path:** The subscription object already sets `SendAlarmTexts=true` and `AlarmTextLanguages=[]` (empty = all languages). This means `AlarmTexts` arrives in each notification without needing a separate browse. The browse path (`ExploreASAlarms`, `GetTexts`) exists in the library but is unnecessary for this use case.
- **AllStatesInfo semantics are partially unknown:** Both `AlarmsDai.AllStatesInfo` and `AsCgs.AllStatesInfo` are parsed but their exact bitmask semantics are not fully documented in the source (marked with TODO). The `SubtypeId` (Coming/Going) is the reliable alarm-state signal; `AllStatesInfo` should be stored as raw bytes for future analysis but not used for logic.
- **HmiInfo fields require SyntaxId check:** `AlarmClass`, `GroupId`, `Producer`, `Flags` are only present when `SyntaxId >= 258`. The `AlarmsHmiInfo.Deserialize` and `FromValueBlob` already handle this correctly. MongoDB document should store these as nullable/absent rather than always-present.

---

## MVP Definition

### Launch With (PoC v1 — End-to-End Demo)

Minimum set needed to demonstrate "trigger alarm on PLC → event appears in MongoDB with all fields."

- [ ] **Subscription lifecycle** — create subscription on connect, replenish credit in loop, delete on disconnect
- [ ] **Alarm state (coming/going)** — `SubtypeId` mapped to a readable string field
- [ ] **PLC-side timestamp** — `AsCgs.Timestamp` stored as UTC BSON Date
- [ ] **Driver-receive timestamp** — `Notification.Add1Timestamp` stored as UTC BSON Date
- [ ] **Alarm identity** — `CpuAlarmId` stored as string
- [ ] **Primary alarm text** — `AlarmTexts.AlarmText` stored as string (inline, no browse)
- [ ] **MongoDB insert** — one document per notification into a `s7alarms` collection

### Add After Validation (v1.x — Completeness Pass)

Features to add once the core pipeline is confirmed working:

- [ ] **Acknowledgement state** — derive `isAcknowledged` bool from `AckTimestamp != DateTime.MinValue`; store `AckTimestamp`
- [ ] **Priority and AlarmClass** — `HmiInfo.Priority`, `HmiInfo.AlarmClass` as additional fields; trivial to add
- [ ] **AlarmDomain** — single ushort field; helps distinguish system vs user alarms
- [ ] **SequenceCounter** — store for ordering/gap detection
- [ ] **Infotext** — companion to AlarmText; store if non-empty
- [ ] **Associated values** — SD_1..SD_10 as a BSON array; add when a real alarm with associated values is available for testing

### Future Consideration (v2+)

- [ ] **Acknowledgement write-back** — requires explicit PoC scope expansion decision
- [ ] **Multi-language texts** — requires multiple subscription objects or language-keyed sub-documents
- [ ] **Additional texts 1-9** — when use cases are confirmed by operators
- [ ] **Alarm catalog pre-fetch** — if static alarm metadata (Lids, AlarmEnabled) proves useful

---

## Minimal MongoDB Alarm Event Document

The fields below represent the target document shape for the PoC launch (v1), with v1.x additions noted.

```json
{
  "_id": ObjectId(),
  "cpuAlarmId": "12345678901234",
  "alarmState": "coming",
  "plcTimestamp": ISODate("2026-03-17T10:23:45.123Z"),
  "driverTimestamp": ISODate("2026-03-17T10:23:45.145Z"),
  "alarmText": "Motor overtemperature",
  "connectionName": "PLC_PLANT_A",

  // v1.x additions
  "isAcknowledged": false,
  "ackTimestamp": null,
  "priority": 10,
  "alarmClass": 2,
  "alarmDomain": 256,
  "sequenceCounter": 42,
  "infotext": "Check cooling system",
  "associatedValues": [
    {"slot": 1, "type": "REAL", "value": 87.3},
    {"slot": 2, "type": "INT",  "value": 15}
  ]
}
```

**Index recommendation:** `{ connectionName: 1, plcTimestamp: -1 }` covers the primary query pattern (events for a given PLC in reverse chronological order).

---

## Feature Prioritization Matrix

| Feature | PoC Value | Implementation Cost | Priority |
|---------|-----------|---------------------|----------|
| Subscription lifecycle + credit loop | HIGH | MEDIUM | P1 |
| Alarm state (coming/going) | HIGH | LOW | P1 |
| PLC-side timestamp | HIGH | LOW | P1 |
| Primary alarm text (inline) | HIGH | LOW | P1 |
| MongoDB insert | HIGH | LOW | P1 |
| Acknowledgement state (passive) | MEDIUM | LOW | P2 |
| Priority + AlarmClass | MEDIUM | LOW | P2 |
| Alarm domain | MEDIUM | LOW | P2 |
| Sequence counter | LOW | LOW | P2 |
| Associated values SD_1..SD_10 | MEDIUM | MEDIUM | P2 |
| Acknowledgement write-back | HIGH (future) | HIGH | P3 |
| Multi-language texts | LOW (for PoC) | HIGH | P3 |
| Alarm catalog browse | LOW | MEDIUM | P3 |
| AdditionalText1..9 | LOW | LOW | P3 |

---

## Sources

- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHandler.cs` — subscription create/delete/wait loop, credit mechanism, TODO annotations on unknown values
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsDai.cs` — full notification data model, field list
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsAsCgs.cs` — timestamp, ack timestamp, associated values per transition
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsAlarmTexts.cs` — text model (11 slots per language, LCID-keyed sparse array)
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHmiInfo.cs` — priority, alarm class, group, flags; SyntaxId-gated fields
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsAssociatedValues.cs` — SD_1..SD_10 typed process values
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsMultipleStai.cs` — alarm domain enum values (1=System, 2=Security, 256-272=UserClass), message type enum
- `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/BrowseAlarms.cs` — ExploreASAlarms, GetActiveAlarms; confirms browse path is an alternative (not required) to inline texts
- `.planning/PROJECT.md` — PoC scope and out-of-scope constraints

---

*Feature research for: S7CommPlus alarm subscription — json-scada PoC*
*Researched: 2026-03-17*

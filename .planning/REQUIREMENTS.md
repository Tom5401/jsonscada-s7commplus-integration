# Requirements: S7CommPlus Alarm Subscriptions for json-scada

**Defined:** 2026-03-17
**Core Value:** Alarms from S7-1200/S7-1500 PLCs appear in json-scada via native protocol subscription — not polling — with full metadata (text, timestamp, ack state)

## v1 Requirements

### Driver Bug Fixes

Prerequisites — must be resolved before alarm subscription can function reliably.

- [x] **BUG-01**: Credit-limit renewal targets correct alarm subscription ObjectId (fix wrong ObjectId reference in `SubscriptionSetCreditLimit` call)
- [x] **BUG-02**: Alarm subscription slot on PLC is deleted cleanly on disconnect (fix `AlarmSubscriptionDelete` to delete subscription object, not session)
- [ ] **BUG-03**: Unknown PDU types are handled gracefully (catch `NotImplementedException` from `Notification.Deserialize` without crashing alarm thread)
- [ ] **BUG-04**: Ack-only notification events are handled without null dereference (null-check `AlarmsDai.FromNotificationObject` result before use)
- [x] **BUG-05**: Alarm subscription uses a distinct RelationId from tag subscription (assign separate value to prevent PLC-side collision)

### Alarm Subscription Lifecycle

- [ ] **LIFE-01**: Driver creates alarm subscription on successful PLC connection
- [x] **LIFE-02**: Driver maintains continuous credit-limit replenishment loop so subscription never goes silent
- [x] **LIFE-03**: Driver deletes alarm subscription cleanly when disconnecting or shutting down

### Alarm Event Data

- [ ] **DATA-01**: Each alarm event captures alarm state (Coming = active / Going = cleared)
- [ ] **DATA-02**: Each alarm event captures alarm text as configured in TIA Portal
- [ ] **DATA-03**: Each alarm event captures PLC-side event timestamp (UTC)
- [ ] **DATA-04**: Each alarm event captures acknowledgement state (passive receive, no write-back)

### Threading & Pipeline

- [ ] **THRD-01**: Alarm notifications are received on a dedicated thread separate from the tag read/write thread
- [ ] **THRD-02**: Thread-safe queue passes alarm events from notification receiver to MongoDB writer

### MongoDB Integration

- [ ] **MONGO-01**: Alarm events are written to a dedicated `s7plusAlarmEvents` collection (separate from `realtimeData`)
- [ ] **MONGO-02**: Alarm writes use acknowledged write concern (WriteConcern.W1 minimum)
- [ ] **MONGO-03**: Each alarm event document contains at minimum: `cpuAlarmId`, `alarmState`, `alarmText`, `timestamp`, `ackState`, `connectionId`, `createdAt`

## v2 Requirements

### Alarm Event Enrichment

- **ENRICH-01**: Alarm event captures priority field
- **ENRICH-02**: Alarm event captures alarm class identifier
- **ENRICH-03**: Alarm event captures alarm domain / group ID
- **ENRICH-04**: Alarm event captures associated values (up to 9 additional text fields)

### Lifecycle Enhancements

- **LIFE-04**: Driver automatically re-subscribes to alarms after connection loss and reconnect (reconnect handling is out of scope for v1 PoC)

## Out of Scope

| Feature | Reason |
|---------|--------|
| Ack write-back to PLC | PoC is read-only; write-back adds complexity without demonstrating core value |
| AdminUI / frontend changes | Data in MongoDB is the deliverable; UI integration is a separate concern |
| Polling-based alarm detection | Replaced entirely by native subscription — polling approach not extended |
| Upstream contribution to json-scada | Internal PoC only |
| Multi-instance redundancy for alarm driver | Single connection PoC; json-scada's existing Redundancy.js pattern not applied |
| AllStatesInfo bitmask interpretation | Semantics partially unknown; stored as raw value only |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| BUG-01 | Phase 1 | Complete |
| BUG-02 | Phase 1 | Complete |
| BUG-03 | Phase 1 | Pending |
| BUG-04 | Phase 1 | Pending |
| BUG-05 | Phase 1 | Complete |
| LIFE-01 | Phase 1 | Pending |
| LIFE-02 | Phase 1 | Complete |
| LIFE-03 | Phase 1 | Complete |
| DATA-01 | Phase 1 | Pending |
| DATA-02 | Phase 1 | Pending |
| DATA-03 | Phase 1 | Pending |
| DATA-04 | Phase 1 | Pending |
| THRD-01 | Phase 1 | Pending |
| THRD-02 | Phase 1 | Pending |
| MONGO-01 | Phase 1 | Pending |
| MONGO-02 | Phase 1 | Pending |
| MONGO-03 | Phase 1 | Pending |

**Coverage:**
- v1 requirements: 17 total
- Mapped to phases: 17
- Unmapped: 0

---
*Requirements defined: 2026-03-17*
*Last updated: 2026-03-17 — MONGO-01 collection name corrected from `alarmEvents` to `s7plusAlarmEvents` per CONTEXT.md locked decision*

# Requirements: S7CommPlus Alarm Subscriptions for json-scada

**Defined:** 2026-03-18
**Core Value:** Alarms from S7-1200/S7-1500 PLCs appear in json-scada via native protocol subscription — not polling — with full metadata (text, timestamp, ack state, associated values), so operators see real events the moment they occur.

## v1.0 Requirements (Validated)

### Alarm Subscription Pipeline

- [x] **PIPE-01**: S7CommPlus connection to PLC with symbolic (optimized block) access
- [x] **PIPE-02**: Read/write tag data via S7CommPlusClient integrated with json-scada MongoDB
- [x] **PIPE-03**: Subscribe to PLC alarm events via S7CommPlus alarm subscription protocol
- [x] **PIPE-04**: Receive alarm state (active/cleared) from PLC subscription notifications
- [x] **PIPE-05**: Receive alarm texts (as configured in TIA Portal) for each alarm event
- [x] **PIPE-06**: Receive PLC-side timestamps for each alarm event
- [x] **PIPE-07**: Receive acknowledgement state for each alarm (passive receive)
- [x] **PIPE-08**: Write alarm events to dedicated MongoDB collection `s7plusAlarmEvents`
- [x] **PIPE-09**: Handle alarm subscription lifecycle (create, maintain, tear down on disconnect)
- [x] **PIPE-10**: End-to-end: trigger alarm on PLC → event appears in MongoDB with all fields

## v1.1 Requirements

### Driver & Data

- [ ] **DRVR-01**: MongoDB `ackState` field correctly reflects PLC acknowledgement state — `false` for unacknowledged, `true` for acknowledged
- [ ] **DRVR-02**: MongoDB alarm documents include a resolved `alarmClassName` string derived from the numeric alarm class ID
- [ ] **DRVR-03**: Acknowledgement command sent from json-scada is received and applied by the PLC

### Alarm Viewer

- [ ] **VIEW-01**: User can navigate to a dedicated S7Plus Alarms Viewer page in AdminUI, separate from the existing tag-based alarm viewer
- [ ] **VIEW-02**: Alarm viewer displays TIA Portal-equivalent columns — Source, Date, Time, Status, Acknowledge, Alarm class name, Event text, ID, Additional texts 1–3
- [ ] **VIEW-03**: User can acknowledge an unacknowledged alarm from the viewer
- [ ] **VIEW-04**: Alarm viewer automatically refreshes to show new alarms without manual page reload
- [ ] **VIEW-05**: User can filter displayed alarms by status (Incoming / Outgoing / All)
- [ ] **VIEW-06**: User can filter displayed alarms by alarm class

## Future Requirements

### Driver & Data

- **DRVR-F01**: Multi-language alarm texts (LCID configurable per connection — currently hardcoded to 1033)
- **DRVR-F02**: Auto-resubscribe on alarm subscription failure mid-session (currently reconnect restarts thread correctly but no retry on mid-session failure)

### Alarm Viewer

- **VIEW-F01**: "Receive alarms" PLC selector dropdown for multi-PLC environments
- **VIEW-F02**: Alarm history export (CSV / PDF)
- **VIEW-F03**: Alarm suppression / shelving (ISA 18.2 advanced states)
- **VIEW-F04**: Real-time WebSocket push for instant alarm appearance

## Out of Scope

| Feature | Reason |
|---------|--------|
| Modifying existing `AlarmsViewerPage.vue` | Tag-based viewer must remain unchanged — new S7Plus viewer is fully independent |
| Custom TIA Portal alarm class display names | Not transmitted in S7CommPlus protocol — only numeric IDs available; static mapping for protocol-fixed IDs only |
| Upstream contribution to json-scada | Internal use only |
| Multi-PLC HA redundancy | Single connection PoC scope |
| AdminUI alarm integration into existing screen infrastructure | New dedicated viewer is the deliverable |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| DRVR-01 | — | Pending |
| DRVR-02 | — | Pending |
| DRVR-03 | — | Pending |
| VIEW-01 | — | Pending |
| VIEW-02 | — | Pending |
| VIEW-03 | — | Pending |
| VIEW-04 | — | Pending |
| VIEW-05 | — | Pending |
| VIEW-06 | — | Pending |

**Coverage:**
- v1.1 requirements: 9 total
- Mapped to phases: 0
- Unmapped: 9 ⚠️

---
*Requirements defined: 2026-03-18*
*Last updated: 2026-03-18 after initial v1.1 definition*

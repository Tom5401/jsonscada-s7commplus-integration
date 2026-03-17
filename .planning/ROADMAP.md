# Roadmap: S7CommPlus Alarm Subscriptions for json-scada

## Overview

The existing S7CommPlusClient driver already handles tag read/write with a live PLC. This roadmap extends it to receive native alarm subscription events and persist them to MongoDB. All 17 v1 requirements are tightly coupled — the bugs, lifecycle, threading, data capture, and MongoDB integration must all function together before a single reliable alarm event reaches the database. A single delivery phase reflects that reality. The PoC is complete when a real PLC alarm appears in `s7plusAlarmEvents` with correct metadata and the tag polling rate is unaffected.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: End-to-End Alarm Pipeline** - Fix all known protocol bugs, build the subscription lifecycle and producer/consumer threading model, write alarm events to MongoDB, and validate end-to-end with a live PLC

## Phase Details

### Phase 1: End-to-End Alarm Pipeline
**Goal**: A running driver that receives S7CommPlus alarm notifications from a live PLC, processes Coming and Going events without crashing or dropping events silently, and inserts structured documents into a `s7plusAlarmEvents` MongoDB collection — while tag polling continues unaffected
**Depends on**: Nothing (first phase)
**Requirements**: BUG-01, BUG-02, BUG-03, BUG-04, BUG-05, LIFE-01, LIFE-02, LIFE-03, DATA-01, DATA-02, DATA-03, DATA-04, THRD-01, THRD-02, MONGO-01, MONGO-02, MONGO-03
**Success Criteria** (what must be TRUE):
  1. Triggering a real alarm on the PLC causes a document to appear in the `s7plusAlarmEvents` MongoDB collection within a few seconds, containing alarm state, alarm text, PLC-side timestamp, and ack state
  2. Clearing the alarm on the PLC causes a corresponding Going event document to appear in `s7plusAlarmEvents`
  3. After 10 or more alarm events, alarm delivery continues without interruption (credit-limit replenishment is working)
  4. Disconnecting and reconnecting the driver leaves no leaked subscription slots on the PLC (PlcSubscriptionsFree recovers to its previous value)
  5. Tag read/write polling rate is not degraded while alarms are flowing (alarm thread is isolated from the tag read cycle)
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. End-to-End Alarm Pipeline | 0/? | Not started | - |

# json-scada S7CommPlus Full Integration

## What This Is
This project extends json-scada to provide full Siemens S7CommPlus support for S7-1200 and S7-1500 PLCs. The current implementation already supports secure connection, symbolic browse, and read/write using the included S7CommPlusDriver, and the next scope is end-to-end alarms and events integration. The target is an open-source SCADA stack where S7Comm and S7CommPlus capabilities are both production-ready.

## Core Value
Operators and integrators can reliably monitor and control modern Siemens PLCs through symbolic S7CommPlus access in json-scada, including alarms and events, without proprietary tooling.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Add complete alarms/events subscription and notification flow from S7CommPlusDriver into json-scada.
- [ ] Preserve and harden existing symbolic browse/read/write behavior for S7-1200/1500.
- [ ] Persist and expose alarm/event state, timestamps, text, and lifecycle in json-scada.
- [ ] Ensure configuration and operations are usable in real deployments (connection/auth/subscription/alarm settings).
- [ ] Add verification coverage for critical protocol paths and alarm/event scenarios.

### Out of Scope

- Porting S7CommPlus to plc4go/plc4x in this milestone — direct driver integration is already underway and is lower risk.
- Full cross-platform refactor of OpenSSL native integration — current milestone prioritizes functional completion in existing runtime context.

## Context
The workspace contains both json-scada and S7CommPlusDriver, and integration work is already in progress. Existing implementation includes symbolic read/write and tag creation mapping with alarm/event-related fields present in the S7CommPlus client tag model. S7CommPlusDriver includes protocol/session/authentication support and partial subscription/alarming code that must be completed and connected to json-scada alarm/event pipelines. Previous planning artifacts in .planning document architecture and cross-repo integration points.

## Constraints

- **Compatibility**: Must support Siemens S7-1200 and S7-1500 PLCs using S7CommPlus optimized access.
- **Architecture**: Build on current json-scada + S7CommPlusDriver integration path; avoid parallel protocol stack rewrites.
- **Reliability**: Alarm/event flow must be robust under continuous operation and communication interruptions.
- **Security**: Preserve secure communication/authentication behavior already implemented in S7CommPlusDriver.
- **Maintainability**: Document design and behavior clearly for future development and troubleshooting.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Use included S7CommPlusDriver as protocol foundation | Existing reverse-engineered implementation already covers core protocol complexity | ✓ Good |
| Prioritize json-scada integration over plc4go port | Faster path to production capability with lower rewrite risk | ✓ Good |
| Focus this milestone on alarms/events completion | Read/write is already implemented; alarms/events are highest remaining functional gap | — Pending |
| Keep planning and docs at workspace root | Supports multi-repo coordination between json-scada and S7CommPlusDriver | ✓ Good |

---
*Last updated: 2026-03-16 after initialization*

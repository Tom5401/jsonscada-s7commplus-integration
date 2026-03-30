# Requirements: S7CommPlus TagTreeBrowser Overhaul

**Defined:** 2026-03-30
**Core Value:** Alarms from S7-1200/S7-1500 PLCs appear in json-scada via native protocol subscription with full metadata, so operators see real events the moment they occur.

## v1.5 Requirements

Requirements for the TagTreeBrowser Overhaul milestone. Each maps to roadmap phases.

### Performance & Lazy Loading

- [ ] **PERF-01**: Backend endpoint `listS7PlusChildNodes` returns only direct children of a given parent path for a connection, not the entire tag subtree
- [ ] **PERF-02**: MongoDB compound index `{protocolSourceConnectionNumber, protocolSourceBrowsePath}` created at server startup for fast child-node queries
- [ ] **PERF-03**: TagTreeBrowser uses Vuetify `load-children` to fetch and display one tree level at a time on node expansion
- [ ] **PERF-04**: 5-second value refresh only polls values for currently expanded/visible leaf tags, not all tags in the datablock

### Value Display

- [ ] **DISP-01**: Digital (boolean) tags display TRUE/FALSE instead of 0/1 in the TagTreeBrowser value column, matching tabular view behavior

### Value Writing

- [ ] **WRITE-01**: User can push a new value to a PLC tag from a TagTreeBrowser leaf node via a write dialog
- [ ] **WRITE-02**: Write button only appears on leaf tags where `commandOfSupervised != 0` (tag has a paired command point)

### Non-Datablock Tags

- [ ] **AREA-01**: MArea, QArea, and other non-datablock area tags appear alongside datablocks in DatablockBrowser and can be browsed in TagTreeBrowser

## Future Requirements

Deferred to future release. Tracked but not in current roadmap.

### Write Enhancements

- **WRITE-03**: Write dialog shows PLC acknowledgement feedback (poll commandsQueue for delivered: true)
- **WRITE-04**: Configurable value refresh interval (currently hardcoded 5s)

### Advanced Browsing

- **BROWSE-01**: Tag search/filter within tree by name
- **BROWSE-02**: WebSocket/SSE real-time value push instead of 5s polling
- **BROWSE-03**: FB type name column (requires GetTypeInformation browse per DB)

## Out of Scope

| Feature | Reason |
|---------|--------|
| Eager full-tree loading (keep v1.4 behavior) | Defeats 100k+ tag goal; causes browser freezes |
| Reuse dlgcomando.html popup for writes | Requires window.opener.WebSAGE context; incompatible with Vue SPA |
| Real-time WebSocket value push | Significant infrastructure complexity for PoC; 5s polling acceptable |
| Write ack feedback | Adds round-trip complexity; "command sent" sufficient for PoC |
| FB type name column | Requires GetTypeInformation browse per DB; high complexity (carried from v1.2) |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| PERF-01 | 16 | Pending |
| PERF-02 | 16 | Pending |
| PERF-03 | 17 | Pending |
| PERF-04 | 17 | Pending |
| DISP-01 | 17 | Pending |
| WRITE-01 | 18 | Pending |
| WRITE-02 | 18 | Pending |
| AREA-01 | 19 | Pending |

**Coverage:**
- v1.5 requirements: 8 total
- Mapped to phases: 8
- Unmapped: 0

---
*Requirements defined: 2026-03-30*
*Last updated: 2026-03-30 after initial definition*

# Requirements: S7CommPlus Tag Tree Browser for json-scada

**Defined:** 2026-03-26
**Milestone:** v1.4 — Tag Tree Browser
**Core Value:** Operators can navigate all PLC datablocks and their tag hierarchies from AdminUI, with live values for configured tags — turning a flat unmanageable tag list into a TIA Portal-style tree browser.

## v1.4 Requirements

### Driver

- [x] **DRIVER-03**: Driver stores the full datablock list from the PLC in a MongoDB `s7plusDatablocks` collection at startup, using upsert keyed on `{connectionNumber, db_name}`; each document contains `db_name`, `db_number`, `db_block_relid`, `db_block_ti_relid`, and `connectionNumber`

### Backend API

- [ ] **API-03**: `GET /Invoke/auth/listS7PlusDatablocks?connectionNumber=N` returns all datablock documents for the specified connection, sorted by `db_name`; admin-guarded consistent with other S7Plus endpoints
- [ ] **API-04**: `GET /Invoke/auth/listS7PlusTagsForDb?connectionNumber=N&dbName=X` returns all `realtimeData` tag documents for that connection where `protocolSourceObjectAddress` starts with `"X"` (quoted DB name prefix); used by TagTreeBrowser to build the tree client-side
- [ ] **API-05**: `POST /Invoke/auth/touchS7PlusActiveTagRequests` accepts an array of `{connectionNumber, protocolSourceObjectAddress}` pairs and upserts them into the `activeTagRequests` MongoDB collection with a refreshed TTL — enabling active OPC read polling for leaf tags that are currently visible in the tree

### DatablockBrowser

- [ ] **DBBROWSER-01**: Operator can see all datablocks on a connected PLC (showing `db_name` and `db_number`) via a new `DatablockBrowserPage` added to the AdminUI navigation menu
- [ ] **DBBROWSER-02**: Operator can filter the datablock list to a specific PLC using a connection selector dropdown
- [ ] **DBBROWSER-03**: Operator can open `TagTreeBrowserPage` for a specific datablock by clicking on a row; the tag browser opens in a new browser tab

### TagTreeBrowser

- [ ] **TAGTREE-01**: Operator sees configured tags for a datablock displayed as a lazy-expanding tree, where the tree structure is derived by parsing `protocolSourceObjectAddress` strings from `realtimeData` (e.g. `"DBName".SubStruct.Tag` → tree depth matching TIA Portal's symbol hierarchy)
- [ ] **TAGTREE-02**: Operator sees live tag values for expanded leaf tags, auto-refreshing every 5 seconds; expanding a node triggers a `touchS7PlusActiveTagRequests` call so the OPC read service actively polls the visible tags
- [ ] **TAGTREE-03**: Operator sees the data type of each leaf tag in the tree (from the `type` or equivalent field in `realtimeData` documents)
- [ ] **TAGTREE-04**: `TagTreeBrowserPage` accepts `?db=<name>&connectionNumber=<N>` query parameters on load, enabling navigation from both `DatablockBrowserPage` and `S7PlusAlarmsViewerPage`

### Integration

- [ ] **INTEGRATION-01**: In `S7PlusAlarmsViewerPage`, the `originDbName` cell is rendered as a clickable link; clicking opens `TagTreeBrowserPage` in a new browser tab pre-filtered to that datablock and connection

## Future Requirements

- Client-side name filter in DatablockBrowserPage (low effort, not prioritised)
- Node type icons per Softdatatype category (MDI icons matching TIA Portal visual language)
- Last-updated timestamp next to live value in tag tree

## Out of Scope

| Feature | Reason |
|---------|--------|
| On-demand PLC browse for unconfigured tags | Requires dedicated 3rd S7CommPlusConnection + commandsQueue ASDU; PoC scope; realtimeData sufficient |
| Write-back / force tag values from browser | Requires SBO logic, access control, command queue extension |
| Multi-dimensional array expansion in tree | 1-dim covers majority of real PLC programs; defer |
| Export tag list to CSV | Not prioritised for PoC |
| Auto-resubscribe after alarm subscription failure | Pre-existing tech debt, not v1.4 scope |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| DRIVER-03 | Phase 12 | Complete |
| API-03 | Phase 13 | Pending |
| API-04 | Phase 13 | Pending |
| API-05 | Phase 13 | Pending |
| DBBROWSER-01 | Phase 14 | Pending |
| DBBROWSER-02 | Phase 14 | Pending |
| DBBROWSER-03 | Phase 14 | Pending |
| TAGTREE-01 | Phase 15 | Pending |
| TAGTREE-02 | Phase 15 | Pending |
| TAGTREE-03 | Phase 15 | Pending |
| TAGTREE-04 | Phase 15 | Pending |
| INTEGRATION-01 | Phase 15 | Pending |

**Coverage:**
- v1.4 requirements: 12 total
- Mapped to phases: 12
- Unmapped: 0

---
*Requirements defined: 2026-03-26*
*Traceability updated: 2026-03-26 — roadmap created*

# Phase 18: Value Write Dialog - Context

**Gathered:** 2026-03-30
**Status:** Ready for planning

<domain>
## Phase Boundary

Add a "Modify" write button to writable leaf nodes in `TagTreeBrowserPage.vue` (only where `commandOfSupervised != 0`), opening a new `PushValueDialog.vue` modal that POSTs an OPC WriteRequest (ServiceId 671) to `/Invoke/`. No backend changes required — `commandOfSupervised` is already returned in the `listS7PlusChildNodes` projection (D-02 from Phase 16) and present on every leaf node from Phase 17's `mapDocToNode()`.

</domain>

<decisions>
## Implementation Decisions

### Write Button (D-01)
- **D-01:** A small `variant="outlined"` `size="x-small"` button labelled **"Modify"** appears in the `append` slot of writable leaf nodes (where `commandOfSupervised != 0`). Placed after the address span. No button is rendered for non-writable leaf nodes or folder nodes.

### Success Feedback (D-02)
- **D-02:** On successful write, show an **inline green success message** inside the dialog (e.g., `v-alert` with `type="success"`), then **auto-close the dialog after ~2 seconds**. On failure, show an inline error message without closing (roadmap SC5).

### Agent's Discretion
- **D-03 (discretion):** Digital tags (type `'digital'`) write as `Type: OpcValueTypes.Double, Body: 1.0` (TRUE) or `0.0` (FALSE) — consistent with the `directCommandExec` pattern in websage.js. Analog/string tags write as `Type: OpcValueTypes.Double` (parsed float) or fall back to `Type: OpcValueTypes.String` if `isNaN`.
- **NodeId lookup:** The write targets `commandOfSupervised` as the numeric `NodeId.Id`, not the supervised tag's `_id`. This is the established pattern — the command point is what gets written to, not the display tag.
- **RequestHandle:** Random integer `Math.floor(Math.random() * 100000000)` — matches existing pattern.
- **ServiceId typo in roadmap:** ROADMAP SC4 says "ServiceId 676" — this is a typo. The correct value is `671` (WriteRequest), confirmed in `opc-codes.js` L104 and `opc_codes.js` L82.
- **Dialog is a separate `.vue` file** (`PushValueDialog.vue`) imported and used by `TagTreeBrowserPage.vue` via props/emit — not inlined in the page component.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Frontend — write dialog target file
- `json-scada/src/AdminUI/src/components/TagTreeBrowserPage.vue` — Phase 17 output; contains `mapDocToNode()` (already maps `commandOfSupervised`), the `append` slot template where the Modify button goes, and `openedNodes`/`treeItems` structure

### Frontend — OPC write pattern (legacy reference)
- `json-scada/src/AdminUI/public/websage.js` §`directCommandExec` (~L1411) — canonical example of how to construct an OPC WriteRequest body (`ServiceId`, `Body.NodesToWrite`, `NodeId`, `AttributeId.Value`, `Value.Type`, `Value.Body`) and POST to `/Invoke/`
- `json-scada/src/AdminUI/public/opc-codes.js` — `OpcServiceCode.WriteRequest = 671`, `OpcValueTypes.Double`, `OpcAttributeId.Value` constants (Vue SPA must replicate these inline since `opc-codes.js` is a legacy global-var file, not an ES module)

### Backend — write handler
- `json-scada/src/server_realtime_auth/index.js` §`case opc.ServiceCode.WriteRequest` (~L878) — the handler that processes writes; enforces `canSendCommands`, looks up the command point by `NodeId.Id`, inserts to `COLL_COMMANDS`, and enqueues `UserActionsQueue`

### Dialog UI pattern
- `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` ~L137–178 — existing `v-dialog → v-card → v-card-title/text/actions` pattern to follow

### Requirements
- `.planning/REQUIREMENTS.md` — WRITE-01 and WRITE-02 define acceptance criteria for this phase

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `mapDocToNode(doc)` in TagTreeBrowserPage.vue (L~54–65): already maps `commandOfSupervised: doc.commandOfSupervised || 0` onto every leaf node — the Modify button visibility check is simply `item.commandOfSupervised !== 0`
- `formatLeafValue(item)` (L~68–72): `item.type === 'digital'` check already established — reuse same check in dialog for `v-select` vs `v-text-field` rendering
- `v-dialog v-card v-card-title/text/actions` pattern — used in S7PlusAlarmsViewerPage.vue and UserManagementTab.vue; follow the same structure

### Established Patterns
- All `/Invoke/auth/*` calls use bare `fetch()` with `Content-Type: application/json` — no axios or custom http client
- POST to `/Invoke/` (no `/auth/` segment) for OPC protocol requests — confirmed in websage.js
- `v-chip size="x-small"` used for type label on leaf nodes — use same size for Modify button for visual consistency

### Integration Points
- `PushValueDialog.vue` receives: `item` (the leaf node object from treeItems), `modelValue` (boolean open/close) — emits `update:modelValue` to close
- The write POSTs to `/Invoke/` not `/Invoke/auth/` — the existing `/Invoke/` route handles OPC protocol with its own auth layer (`canSendCommands` check inside the handler)

</code_context>

<specifics>
## Specific Ideas

- Button label is **"Modify"** (not "Write" or "Push") — user's explicit choice
- Success state: inline `v-alert type="success"` inside dialog → auto-close after ~2s
- Error state: inline `v-alert type="error"` without closing — user can retry or cancel

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 18-value-write-dialog*
*Context gathered: 2026-03-30*

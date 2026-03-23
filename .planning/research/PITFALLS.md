# Pitfalls Research

**Domain:** S7CommPlus alarm origin (PLC browse + relationID lookup) and alarm delete (per-row + bulk filtered) in json-scada
**Researched:** 2026-03-23
**Confidence:** HIGH (codebase analysis) / MEDIUM (protocol-level claims — reverse-engineered protocol, no public spec)

---

## Critical Pitfalls

### Pitfall 1: relationID is NOT Stable Across TIA Portal Downloads

**What goes wrong:**
The in-memory lookup map built at driver startup (`relationID → DB/FB name`) becomes silently stale. After a TIA Portal compile-and-download, PLCs in the S7-1200/S7-1500 family reassign internal object IDs — including the `RelationId` that the alarm subscription notifications carry in each `AlarmsDai`. The driver continues running with the old map, and the `dbName` field written to MongoDB is wrong for every alarm fired after the download, with no error logged.

**Why it happens:**
The S7CommPlus `RelationId` is an internal PLC object-tree identifier generated at compile time, not a stable user-visible name. `BrowseAlarms.cs` shows that the `cpuAlarmId` itself is constructed as `(ulong)relationId << 32 | (ulong)alid << 16` — both halves can change after a re-download. The driver never re-browses after startup unless it reconnects. A TIA Portal download causes a warm restart but keeps the TCP connection alive, so the driver does not detect the change.

**How to avoid:**
- Accept the map as startup-time best-effort, not a long-lived guarantee. Store the resolved name in the alarm event document at write time (not as a live join). Once written, the name is immutable even if the PLC is re-downloaded.
- Do not try to back-fill old documents after a download. The old names were correct at the time of the events.
- Log startup browse results with the PLC firmware version (readable via `ExploreRequest`) so post-incident analysis can identify when a re-download occurred.
- Keep the browse isolated to the tag connection (`srv.connection`) at startup. Do not re-browse during the alarm subscription loop.

**Warning signs:**
- Alarm events after a known TIA Portal download show DB names that do not match any DB visible in the current TIA Portal project.
- `cpuAlarmId` values in new events do not match any key in the startup-built map (missing lookup → empty/unknown origin).

**Phase to address:** Phase that implements the browse call and map construction (v1.2 Phase 1 / driver side).

---

### Pitfall 2: Browse on the Tag Connection Races With the Alarm Thread Startup

**What goes wrong:**
The alarm subscription browse (`ExploreASAlarms` or a custom ExploreRequest on the tag connection at startup) fires on `srv.connection` while `AlarmThread` is starting up on its own `alarmConn`. If the browse is done on the wrong connection — or if the two connections share a mutex — the alarm subscription creation and the browse response can interleave PDUs on the same socket, producing `CreateObjectResponse.DeserializeFromPdu` failures or wrong session IDs parsed into alarm objects.

**Why it happens:**
`ConnectionThread` in `Program.cs` starts the alarm thread (`AlarmThread`) and then calls `BrowseAndCreateTags` in immediate sequence (lines 286–296). The existing tag browse (`srv.connection.Browse(...)`) runs on the tag connection. A startup alarm-origin browse must also use `srv.connection` (tag conn), not `alarmConn` — but `alarmConn` connects immediately at the start of `AlarmThread` and sends `AlarmSubscriptionCreate` almost simultaneously. The connections are separate TCP sockets, so there is no PDU interleaving between them; however, the browse on `srv.connection` must complete before the first read cycle starts, and the alarm thread does not depend on browse results.

**How to avoid:**
- Run the origin browse on `srv.connection` (tag connection) in the same block as `BrowseAndCreateTags`, before the first read cycle. Never call ExploreRequest on `alarmConn`.
- Start `AlarmThread` after the tag browse completes, not before it (reverse the order in `ConnectionThread`). This is a safe sequence change; the alarm thread is independent of browse results.
- Store the lookup result in `srv` as a `Dictionary<uint, string>` (`RelationIdToDbName`) accessible from `AlarmThread` via the shared `srv` reference.

**Warning signs:**
- `CreateObjectResponse` deserialization errors logged at startup when alarm subscription is created.
- `Notification.DeserializeFromPdu` returns null for the first alarm notification received.

**Phase to address:** Phase 1 (driver — startup browse implementation).

---

### Pitfall 3: ExploreASAlarms Returns Empty or Throws on PLCs With No User Alarms Configured

**What goes wrong:**
`ExploreASAlarms` in `BrowseAlarms.cs` calls `exploreRes.Objects.First(...)` — which throws `InvalidOperationException` if the result set is empty. On a real S7-1200 or S7-1500 PLC with no Program Alarms configured (only System/Diagnostic alarms, or a PLC with an empty user program), the explore response may return zero objects matching `PLCProgram_Class_Rid`. The driver crashes at startup and exits `ConnectionThread`, which then kills the alarm thread too.

**Why it happens:**
The PLCSIM Advanced V8 environment used for v1.1 validation had Program Alarms configured. Real hardware may not. The `First()` LINQ call has no guard. Additionally, the `0x8a7e0000` system alarm area (AS Alarms) is only valid on S7-1500 firmware >= 2.0; an S7-1200 may return an error PDU with a non-zero ReturnValue, which is not checked before the `First()` call on the result.

**How to avoid:**
- Replace all `.First(o => ...)` calls in the browse path with `.FirstOrDefault(o => ...)` + null checks. Log "no Program Alarms found" and return an empty map rather than throwing.
- Validate `exploreRes.ReturnValue == 0` before accessing `exploreRes.Objects`. On non-zero return, log a warning, populate an empty map, and continue — alarm subscription still works, just without origin enrichment.
- Wrap the entire browse-for-origin in a try-catch so a browse failure is non-fatal and does not prevent alarm subscription.

**Warning signs:**
- `InvalidOperationException: Sequence contains no elements` in the log at startup.
- Driver exits immediately after the first connection attempt against a PLC with no user alarms.

**Phase to address:** Phase 1 (driver — startup browse implementation).

---

### Pitfall 4: Delete Race With the 5-Second Auto-Refresh

**What goes wrong:**
The operator clicks "Delete" for row X. The HTTP DELETE request reaches the backend at t=0. The frontend's 5-second refresh fires at t=0.1, fetches the list (including row X), and re-renders the table with row X present. The DELETE completes at t=0.5 and removes the MongoDB document. The refresh at t=5.1 fetches the list without row X. The net visible effect is: row X flickers back for up to 5 seconds after clicking Delete. If the operator double-clicks, a second DELETE is sent for a document that no longer exists — `DeleteOne` returns `DeletedCount=0`, which must be treated as success, not an error.

**Why it happens:**
The auto-refresh has no coordination with in-flight mutation operations. The Vue component does not optimistically remove the row on click, waiting instead for the next poll to confirm the deletion.

**How to avoid:**
- Optimistically remove the row from `alarms.ref` immediately on successful DELETE response (same pattern as the pending ack set already used for `ackState`). This hides the flicker without requiring coordination with the refresh timer.
- Treat `DeletedCount=0` as a non-error in the backend (the document was already gone — idempotent).
- Do NOT cancel the refresh timer during a delete; keep the 5s cadence independent.

**Warning signs:**
- Row reappears for one refresh cycle after clicking Delete.
- Backend returns an error on second click of the same row (if error is thrown for `DeletedCount=0`).

**Phase to address:** Phase 2 (AdminUI — delete button implementation).

---

### Pitfall 5: Bulk "Delete Filtered" Filter Drift Between UI State and Backend Execution

**What goes wrong:**
The operator sets Status=Incoming and clicks "Delete Filtered." The Vue component sends the current filter state to the backend. Between the time the user clicked the button and the backend executes the MongoDB `deleteMany`, the alarm subscription thread has inserted new alarms that match the filter. Those new alarms are deleted even though the operator did not see them and may not have intended to delete them.

The reverse is also possible: the operator sees 10 rows, clicks Delete Filtered, the backend receives the request, but the filter parameters sent are slightly broader than the visible rows (e.g., "All" for alarm class while the operator expected to delete only the 10 visible "Incoming" rows from one specific class).

**Why it happens:**
There is no atomic snapshot between "what the operator sees" and "what gets deleted." The filter is re-applied against the live collection at execution time. The collection is continuously written to by `AlarmThread`.

**How to avoid:**
- The backend must execute `deleteMany` using exactly the filter parameters sent in the request body — no more, no less. Serialize the full filter (statusFilter + alarmClassFilter + connectionId if applicable) and pass all fields.
- Add a `dryRunCount` step: the backend first counts matching documents, and the response includes how many will be deleted. The frontend confirms. This is an optional UX improvement, not a correctness requirement for PoC.
- For PoC scope: accept that new alarms arriving in the ~100ms between click and execution may be included in the delete. Document this as known behavior. Do NOT attempt transactional isolation — this is a PoC.
- Use the MongoDB document `_id` array for precise deletion if the operator selects specific rows (per-row delete case). `deleteMany` with filter is acceptable for bulk delete with explicit operator acknowledgment.

**Warning signs:**
- Operator reports "I only deleted 10 alarms but 15 were removed."
- Backend logs show a high `DeletedCount` that does not match the visible row count in the UI.

**Phase to address:** Phase 2 (AdminUI delete + Phase 3 backend delete endpoint).

---

### Pitfall 6: Active / Unacked Alarms Deleted From History While Still Active on PLC

**What goes wrong:**
The operator deletes an alarm event document from MongoDB while the corresponding alarm is still active (Coming, unacked) on the PLC. The alarm subscription thread does not re-insert the document. The alarm remains active on the PLC but is invisible in the viewer. The operator believes the alarm was resolved. When the PLC sends the "Going" (cleared) notification, the alarm thread writes a new Going document — but there is no corresponding Coming document to pair with it, breaking the event history.

**Why it happens:**
The S7CommPlus alarm subscription is event-driven (push), not poll-and-sync. There is no reconciliation loop that re-queries the PLC for currently active alarms and ensures they are present in MongoDB. The viewer is purely a MongoDB projection.

**How to avoid:**
- For PoC scope: do not prevent deletion of active/unacked alarms but warn the operator in the UI. Show a confirmation dialog: "This alarm is currently active on the PLC. Deleting it will remove it from history but the PLC alarm state is unchanged. Continue?"
- Check `alarmState === 'Coming'` and `ackState === false` before allowing deletion; surface a warning chip or confirmation dialog in Vue.
- Do NOT add a server-side guard that blocks deletion of active alarms — this would require a PLC query in the delete path, which is out of scope.

**Warning signs:**
- Operator deletes a Coming+unacked alarm, then sees an orphan Going event with no matching Coming event in the viewer.
- Alarm count on PLC HMI does not match alarm count in the viewer.

**Phase to address:** Phase 2 (AdminUI — delete button UX) and Phase 3 (backend delete endpoint design notes).

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Build origin map once at startup, never refresh | Simple; no background thread needed | Map is stale after TIA Portal re-download until driver restart | Acceptable for PoC — document in VALIDATION.md |
| Store DB name in alarm event document at write time (denormalized) | No join needed at query time | Old documents have wrong name after re-download | Always acceptable — events are immutable log entries |
| No index on `s7plusAlarmEvents.cpuAlarmId` | No migration needed | DeleteOne by cpuAlarmId does a full collection scan | Add index in Phase 3 before delete operations; negligible effort |
| No index on `s7plusAlarmEvents.alarmState` + `ackState` | No migration needed | Bulk delete filter on unindexed fields scans full collection | Acceptable at PoC scale (hundreds of documents) |
| Bulk delete uses filter, not document ID list | Simpler frontend implementation | May delete concurrent inserts matching filter | Acceptable for PoC — document as known behavior |
| No undo / soft-delete for alarm history | Simpler delete path | Deleted alarms are gone; no recovery without MongoDB backup | Acceptable for PoC |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| S7CommPlus `ExploreASAlarms` on tag connection | Calling it on `alarmConn` instead of `srv.connection` | Browse must use the tag connection; `alarmConn` is exclusively for alarm subscription PDUs |
| MongoDB `deleteMany` from Node.js backend | Throwing an error when `deletedCount === 0` | `deletedCount === 0` is success (idempotent delete) — return `{ deleted: 0 }` not an error |
| Vue `pendingAcks` Set pattern reused for `pendingDeletes` | Forgetting to remove the ID from the set on delete success | On successful delete, remove the row from `alarms.ref` AND clear the pending ID; otherwise spinner never stops |
| `ExploreASAlarms` + `.First()` LINQ | Throws on PLCs with no user alarms (S7-1200 with only diagnostic alarms) | Use `.FirstOrDefault()` + null guard; treat empty result as empty map, not error |
| Storing `RelationId` as `uint` in MongoDB | BSON Int32 cannot hold values > 2^31; `uint` max = 4,294,967,295 | Store as `BsonInt64` (Int64) or as string to preserve full uint range without sign-extension |
| server_realtime_auth delete endpoint authentication | Adding the endpoint without `[authJwt.isAdmin]` middleware (copy-paste from unprotected routes) | Mirror the existing `listS7PlusAlarms` pattern exactly — `[authJwt.isAdmin]` must be present |
| `deleteMany` filter construction in Node.js | Passing Vue filter values directly without sanitizing (e.g., 'All' → no filter, not `alarmState: 'All'`) | Map 'All' → omit the field from the MongoDB filter; never pass 'All' as a literal field value |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| No index on `s7plusAlarmEvents` for delete queries | Delete operations scan the full collection; noticeably slow when collection grows | Add `{ cpuAlarmId: 1 }` and `{ alarmState: 1, ackState: 1 }` compound index before Phase 3 | Unnoticeable at <1000 documents; visible at 10,000+ |
| `listS7PlusAlarms` returns `.limit(200)` but bulk delete ignores limit | Operator sees 200 rows, clicks Delete All, but deleteMany removes all 50,000 documents matching the filter | Backend must apply the same filter for both list and delete — and the operator must be shown the actual delete count | At PoC scale with <1000 events, this is rarely a practical problem but a UX surprise |
| Browse blocks the tag read cycle | `ExploreASAlarms` is synchronous on `srv.connection`; if it takes 5s on a large PLC program, the tag polling loop is blocked for that duration | Perform browse before first read cycle in `ConnectionThread`, not inside the loop. The existing `BrowseAndCreateTags` already does this correctly. | Only an issue at startup; no runtime impact |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Delete endpoint missing `authJwt.isAdmin` | Any authenticated user (read-only role) can delete alarm history | Copy the `[authJwt.isAdmin]` guard from `ackS7PlusAlarm` exactly |
| Accepting arbitrary MongoDB filter JSON from the client body | Server-side injection or unintended collection scans | Whitelist allowed filter fields (alarmState, alarmClassName, connectionId); reject unknown keys |
| No rate limiting on delete endpoint | Operator (or script) can delete entire collection in a loop | Acceptable for PoC (internal only); flag for production |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| No confirmation dialog for Delete row | Operator accidentally deletes an alarm that is still active on PLC; no recovery | Show confirmation dialog for Coming+unacked alarms; allow direct delete for Going/acked |
| No confirmation dialog for "Delete Filtered" | Bulk delete of hundreds of records with one click | Always show "Delete N matching records?" confirmation before executing |
| Row reappears after delete due to 5s refresh | Operator believes delete failed and clicks again | Optimistically remove the row from `alarms.ref` on successful DELETE response |
| "Delete Filtered" button active when filter shows zero rows | Operator clicks an apparently-functional button that does nothing | Disable "Delete Filtered" when `filteredAlarms.length === 0` |
| No origin column when browse failed at startup | DB Name column is always empty; operator thinks feature is broken | Show "(unavailable)" instead of empty string; add tooltip "Origin lookup failed at startup" |

---

## "Looks Done But Isn't" Checklist

- [ ] **RelationID stored in MongoDB**: Verify `s7plusAlarmEvents` documents have a `relationId` field (not just the resolved `dbName`) — the raw ID is needed if the name lookup failed at startup.
- [ ] **Browse failure is non-fatal**: Confirm driver still subscribes to alarms and writes events if `ExploreASAlarms` returns an error or empty map — alarm subscription must not depend on browse success.
- [ ] **Delete is idempotent**: Second click on already-deleted row returns success (not 404 or error) — verify with `deletedCount === 0` handling in backend.
- [ ] **Bulk delete filter matches visible rows**: Confirm that the filter object sent from Vue matches exactly the filter applied by MongoDB query — test with Status=Incoming and a specific Alarm Class selected.
- [ ] **Origin column shows "(unknown)"**: Confirm that alarms whose `relationId` was not in the startup map show a graceful fallback, not an exception.
- [ ] **RelationId uint range**: Confirm that `relationId` values > 2,147,483,647 (e.g., `0x8a7e0000 = 2,323,103,744`) are stored as Int64 or string in BSON — not silently truncated to Int32.
- [ ] **No index missing**: Confirm `cpuAlarmId` index exists before running delete operations against more than a few hundred documents.
- [ ] **Delete endpoint authenticated**: Confirm `authJwt.isAdmin` middleware is present on both the per-row and bulk delete routes in `server_realtime_auth/index.js`.
- [ ] **AlarmThread sees the origin map**: Confirm the lookup dictionary built during startup browse is stored on `srv` and is accessible from `AlarmThread` without a separate connection or lock.

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Stale origin map after TIA Portal download | LOW | Restart the S7CommPlusClient driver — it re-browses at the next `ConnectionThread` connect |
| Browse failure at startup (empty map) | LOW | Log shows the failure; restart driver; if PLC has no user alarms the map will always be empty — expected behavior |
| Active alarm deleted by operator | MEDIUM | Wait for the PLC to re-send a new alarm event (e.g., transition Going then Coming again); or manually inspect PLC via TIA Portal |
| Bulk delete deleted more than intended | HIGH (data loss) | Restore from MongoDB backup (if taken); no in-app undo. For PoC, accept this risk and document it. |
| `RelationId` stored as Int32 with truncation | MEDIUM | Re-run the browse at next restart; update `BuildAlarmDocument` to use Int64; existing wrong documents remain but are harmless (display issue only) |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| RelationID instability after TIA Portal download | Phase 1 (driver browse) | Document as known behavior in VALIDATION.md; confirm graceful fallback when map miss occurs |
| Browse on wrong connection / race with alarm thread startup | Phase 1 (driver browse) | Confirm browse is on `srv.connection` and completes before first read cycle; AlarmThread still starts and receives events |
| ExploreASAlarms throws on PLC with no user alarms | Phase 1 (driver browse) | Test with a connection config pointing to a PLC or PLCSIM instance with zero Program Alarms configured |
| Delete race with 5s refresh / flicker | Phase 2 (Vue delete button) | Click Delete, observe row disappears immediately without waiting for next 5s poll |
| Bulk delete filter drift | Phase 2 (Vue) + Phase 3 (backend endpoint) | Set Status=Incoming, click Delete Filtered, verify only Incoming events are removed |
| Active alarm deleted silently | Phase 2 (Vue UX) | Trigger Coming alarm, click Delete, confirm dialog appears for active/unacked rows |
| `RelationId` uint stored as Int32 truncation | Phase 1 (driver browse + BuildAlarmDocument) | Inspect MongoDB document for `relationId: NumberLong(...)` — must not be negative (sign-extended) for IDs > 0x7FFFFFFF |
| Delete endpoint missing auth | Phase 3 (backend delete endpoint) | Attempt DELETE without admin token — verify 401 is returned |
| No index before delete operations | Phase 3 (backend delete endpoint) | Confirm index creation runs as part of driver startup or migration step |

---

## Sources

- Codebase analysis: `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/BrowseAlarms.cs` (ExploreASAlarms, GetTexts, AlarmData, cpuAlarmId construction from RelationId + alid)
- Codebase analysis: `S7CommPlusDriver/src/S7CommPlusDriver/Alarming/AlarmsHandler.cs` (AlarmsHandler, WaitForAlarmNotification, credit limit)
- Codebase analysis: `json-scada/src/S7CommPlusClient/AlarmThread.cs` (BuildAlarmDocument, alarm subscription loop)
- Codebase analysis: `json-scada/src/S7CommPlusClient/Program.cs` (ConnectionThread ordering: AlarmThread start then BrowseAndCreateTags)
- Codebase analysis: `json-scada/src/S7CommPlusClient/MongoCommands.cs` (existing ack pipeline — pattern for delete pipeline)
- Codebase analysis: `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue` (pendingAcks pattern, fetchAlarms loop, filter state)
- Codebase analysis: `json-scada/src/server_realtime_auth/index.js` (listS7PlusAlarms, ackS7PlusAlarm — patterns for delete endpoints)
- Protocol knowledge: reverse-engineered S7CommPlus driver (Thomas Wiens, th.wiens@gmx.de) — RelationId is a compile-time object-tree ID, not a stable user-visible name
- General ICS engineering: TIA Portal compile-and-download causes PLC internal object ID reassignment (standard behavior, MEDIUM confidence — not in public spec)
- MongoDB docs: [deleteMany](https://www.mongodb.com/docs/drivers/node/current/usage-examples/deleteMany/) — `deletedCount === 0` is valid success, not an error
- MongoDB docs: [Race conditions in concurrent operations](https://medium.com/@codersauthority/handling-race-conditions-and-concurrent-resource-updates-in-node-and-mongodb-by-performing-atomic-9f1a902bd5fa)

---
*Pitfalls research for: S7CommPlus alarm origin + alarm delete features (v1.2)*
*Researched: 2026-03-23*

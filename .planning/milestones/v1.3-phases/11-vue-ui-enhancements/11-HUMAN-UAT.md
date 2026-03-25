---
status: partial
phase: 11-vue-ui-enhancements
source: [11-VERIFICATION.md]
started: 2026-03-25T00:00:00Z
updated: 2026-03-25T00:00:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Timestamp column display
expected: Single Timestamp column visible with format `2026-03-24_12:57:10.758`; no separate Date or Time columns
result: [pending]

### 2. Priority sort
expected: Clicking Priority column header sorts alarms ascending then descending by numeric priority
result: [pending]

### 3. Ack indicator
expected: Alarms with isAcknowledgeable === false show a dash (`-`) in Acknowledge column; others show Ack button or green checkmark
result: [pending]

### 4. Page preservation
expected: Navigating to page 2+, waiting 5+ seconds for auto-refresh, page number stays unchanged
result: [pending]

### 5. Source filter
expected: Source dropdown in filter row populates with distinct PLC connection names; selecting one filters the table; combines with Status and Alarm Class filters
result: [pending]

### 6. Ack All full flow
expected: Ack All button shows count; clicking opens confirmation dialog; Cancel closes it; Acknowledge processes acks sequentially; one failure does not block remaining acks
result: [pending]

## Summary

total: 6
passed: 0
issues: 0
pending: 6
skipped: 0
blocked: 0

## Gaps

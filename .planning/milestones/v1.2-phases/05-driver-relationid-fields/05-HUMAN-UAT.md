---
status: partial
phase: 05-driver-relationid-fields
source: [05-VERIFICATION.md]
started: 2026-03-23T15:00:00Z
updated: 2026-03-23T15:00:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Live document verification
expected: With a connected PLCSIM Advanced instance, trigger an alarm event and inspect the resulting MongoDB document in the `s7plusAlarmEvents` collection. Document must contain `relationId` field (BSON Int64, non-zero) and `dbNumber` field (BSON Int32, non-zero, ≤ 65535).
result: [pending]

## Summary

total: 1
passed: 0
issues: 0
pending: 1
skipped: 0
blocked: 0

## Gaps

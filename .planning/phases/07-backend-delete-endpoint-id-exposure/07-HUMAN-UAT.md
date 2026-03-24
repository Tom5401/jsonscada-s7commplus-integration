---
status: partial
phase: 07-backend-delete-endpoint-id-exposure
source: [07-VERIFICATION.md]
started: 2026-03-24T00:00:00Z
updated: 2026-03-24T00:00:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Auth guard runtime HTTP response
expected: POST /Invoke/auth/deleteS7PlusAlarms without a JWT token returns 401/403 and no document is deleted. POST with a non-admin JWT also returns 401/403 with no deletion. (Middleware wiring confirmed in source; RESEARCH.md Pitfall 1 documents a pre-existing ambiguity in the no-token path that cannot be resolved by static analysis.)
result: [pending]

## Summary

total: 1
passed: 0
issues: 0
pending: 1
skipped: 0
blocked: 0

## Gaps

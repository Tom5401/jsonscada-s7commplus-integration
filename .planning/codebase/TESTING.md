# Testing

**Analysis Date:** 2026-03-17

## Summary

Testing coverage is **minimal**. Only one test file exists in the entire codebase. The vast majority of modules have no tests.

---

## Test Framework

**Where tests exist:** `src/log-io/ui/` only

| Tool | Version | Usage |
|------|---------|-------|
| Jest | (in log-io/ui) | Test runner |
| React Testing Library | (in log-io/ui) | Component testing |

---

## Test File

**Location:** `src/log-io/ui/src/components/index.test.tsx`

**What it tests:** The `log-io` React UI component

**Test cases:**
- Renders default single screen
- Renders inputs received from server via socket
- Adding additional screens
- Multiple screens with multiple inputs
- Socket emit on input activation/deactivation
- Socket emit on stream activation/deactivation
- Renders messages to correct screen

**Pattern — Socket Mock:**
```ts
const mockSocket = () => {
  const callbacks: any = {}
  const mockEmit = jest.fn()
  return {
    on: (eventName, callback) => { /* store callbacks */ },
    trigger: (eventName, data) => { /* invoke callbacks */ },
    emit: mockEmit,
  }
}
```

**Pattern — State Update Wrapping:**
```ts
act(() => {
  socket.trigger('+input', testInput)
})
```

**Pattern — Assertions:**
```ts
const element = getByTestId('screen-0')
expect(element).toBeDefined()
expect(socket.emit).toBeCalledWith('+activate', 'app1|server1')
```

---

## Coverage

| Module | Tests |
|--------|-------|
| `log-io/ui` | 7 test cases |
| `server_realtime_auth` | None (`"test": "echo \"Error: no test specified\" && exit 1"`) |
| `cs_data_processor` | None |
| `AdminUI` (Vue) | None |
| `calculations` (Go) | None |
| `i104m` (Go) | None |
| All C# modules | None |
| All other Node.js modules | None |

**Coverage threshold:** None enforced

---

## Running Tests

```bash
# Only works for log-io/ui
cd src/log-io/ui
npm test
```

All other modules return:
```
Error: no test specified
exit code 1
```

---

## Gaps & Observations

- No integration tests
- No API-level tests for `server_realtime_auth` or `graphql-server`
- No tests for protocol driver logic
- No tests for MongoDB change stream processing
- No tests for calculation engine
- No frontend component tests for AdminUI (Vue 3 / Vitest would be the natural choice)
- No CI pipeline enforcing test runs detected

---

## Recommended Test Strategy (if adding tests)

**AdminUI (Vue 3):** Vitest + Vue Test Utils
```bash
npm install -D vitest @vue/test-utils
```

**Node.js services:** Jest or Node built-in `node:test`

**Go services:** Built-in `go test` with table-driven tests

**C# services:** xUnit or NUnit

**Integration:** Test against a local MongoDB replica set (`rs1`)

---

*Testing analysis: 2026-03-17*

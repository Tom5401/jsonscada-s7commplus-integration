# Coding Conventions

**Analysis Date:** 2026-03-17

## Code Formatting

**Tool:** Prettier (enforced in AdminUI)

**Config** (`src/AdminUI/.prettierrc`):
```json
{
  "tabWidth": 2,
  "semi": false,
  "singleQuote": true,
  "trailingComma": "es5",
  "printWidth": 80,
  "bracketSpacing": true,
  "endOfLine": "lf",
  "vueIndentScriptAndStyle": true,
  "arrowParens": "always"
}
```

**Linting:** ESLint 8.57 with `plugin:vue/vue3-essential` rules

---

## Naming Conventions

### TypeScript / Node.js
| Entity | Convention | Example |
|--------|-----------|---------|
| Files | `kebab-case` | `connection-manager.ts`, `load-config.js` |
| Interfaces | `PascalCase` with `I` prefix | `ILog`, `IConfig`, `IProcessInstance` |
| Functions | `camelCase` | `bindInputToScreen`, `handleNewMessage` |
| Constants | `UPPER_SNAKE_CASE` | `STORAGE_KEY`, `ENV_PREFIX` |
| Classes | `PascalCase` | `ConnectionManager` |
| Variables | `camelCase` | `connectionString`, `tagList` |

### Vue Components
- Files: `PascalCase.vue` (`AdminPage.vue`, `TagsTab.vue`)
- Component names match file names exactly
- Pages are in `src/pages/`, reusable in `src/components/`

### Go
- Packages: single lowercase word matching directory
- Exported types/functions: `PascalCase`
- Private: `camelCase`

### C#
- Classes/Methods: `PascalCase`
- Private fields: `camelCase` or `_camelCase`

---

## TypeScript Patterns

**Strict Mode:** TypeScript strict mode enabled in AdminUI

**Import Order:**
1. Node.js stdlib
2. Third-party packages
3. Application modules (relative)
4. ES modules use `.js` extensions even for `.ts` sources

**Type Safety:**
- Interfaces over `any` where possible
- Custom `ILog` interface for logger: `{ levelNow: number; log(level: number, msg: string): void }`
- Config objects typed via interfaces

---

## Error Handling

**Pattern — Critical Failures:**
```js
process.on('uncaughtException', (err) => {
  console.error('Uncaught Exception:', err)
  process.exit(1)
})
```

**Pattern — Async:**
- `try/catch` around MongoDB operations
- Reconnect loops with `setTimeout` for database/connection failures

**Pattern — Protocol Drivers:**
- Log error and continue (don't crash on single bad datapoint)
- Process-level exit only on unrecoverable config errors

---

## Logging

**Custom ILog interface** (used across Node.js services):
```js
// simple-logger.js pattern (per service)
const log = {
  levelNow: 2,
  log(level, msg) {
    if (level <= this.levelNow)
      console.log(new Date().toISOString() + ' - ' + msg)
  }
}
```

**Log levels:** 0 (critical), 1 (error), 2 (info, default), 3 (debug)

**Format:** ISO 8601 timestamps — `2026-03-17T10:30:00.000Z - message`

**Comments:** JSDoc for public functions; inline comments explain *why*, not *what*

---

## State Management (Vue / Frontend)

**Pattern:** `useReducer` with `immer` for immutable updates

**Action pattern:** Type-safe action unions
```ts
type Action =
  | { type: 'SET_TAG'; payload: Tag }
  | { type: 'CLEAR_ALARMS' }
```

**Store location:** Pinia or local component state (no centralized Vuex)

---

## Module Structure (Node.js Services)

Each service follows a consistent internal pattern:
```
service-name/
├── index.js          # Entry point, main loop
├── load-config.js    # Load conf/json-scada.json + env vars
├── app-defs.js       # Constants and type definitions
├── simple-logger.js  # Lightweight logger
├── redundancy.js     # Active/passive redundancy handling
└── package.json
```

---

## Configuration Loading Pattern

```js
// Standard pattern across all Node.js services
const configFile = process.env.JS_CONFIG_FILE || '../conf/json-scada.json'
const config = JSON.parse(fs.readFileSync(configFile))
// Override with env vars: JS_<SERVICE>_<PARAM>
```

Environment variable prefix: `JS_` (e.g., `JS_CSDATAPROC_LOG_LEVEL`, `JS_MCPJSDB_PORT`)

---

## MongoDB Interaction Pattern

**Change Streams (primary read pattern):**
```js
const changeStream = collection.watch(pipeline, { fullDocument: 'updateLookup' })
changeStream.on('change', (change) => { /* handle update */ })
```

**Write pattern:** `replaceOne` with `upsert: true` for idempotent writes

**Connection:** Shared connection string from config, reconnect on error

---

*Conventions analysis: 2026-03-17*

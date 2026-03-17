# Directory Structure

**Analysis Date:** 2026-03-17

## Repository Layout

```
dev/
├── json-scada/               # Main project (git submodule)
│   ├── src/                  # All source code - 40+ modules
│   ├── conf/                 # Active runtime configuration
│   ├── conf-templates/       # Configuration templates
│   ├── docs/                 # Documentation
│   ├── platform-*/           # Platform-specific setup scripts
│   ├── demo-docker/          # Docker demo environment
│   ├── compile-docker/       # Docker build definitions
│   ├── mongo_seed/           # MongoDB initial seed data
│   ├── sql/                  # PostgreSQL schema and migrations
│   ├── svg/                  # Process display SVG files
│   ├── bin/                  # Compiled binaries (runtime)
│   ├── log/                  # Runtime logs
│   ├── tmp/                  # Temporary files
│   ├── supervisord.conf      # Supervisor process manager config
│   └── Dockerfile            # Main Docker image
└── .planning/                # GSD planning artifacts
    └── codebase/             # This codebase map
```

---

## Source Modules (`json-scada/src/`)

Each module is an independently deployable service with its own `package.json` or `go.mod`:

### Frontend
| Module | Language | Description |
|--------|----------|-------------|
| `AdminUI/` | Vue 3 / TypeScript | Main web UI (SCADA HMI, admin, alarms, config) |
| `svg-display-editor/` | JavaScript | Browser-based SVG process display editor |
| `svgedit/` | JavaScript | SVG editor library (forked) |

### Backend Core (Node.js)
| Module | Language | Description |
|--------|----------|-------------|
| `server_realtime_auth/` | Node.js / TypeScript | Main API gateway, auth, Apollo GraphQL, Socket.IO |
| `cs_data_processor/` | Node.js | Core data processor — writes realtime values to MongoDB |
| `cs_custom_processor/` | Node.js | Custom data processing hooks |
| `graphql-server/` | Node.js | PostGraphile auto-GraphQL over PostgreSQL |
| `config_server_for_excel/` | Node.js | Excel-based configuration importer |
| `carbone-reports/` | Node.js | PDF/document report generation |
| `calculations/` (alt) | Node.js | JS-based calculation engine (secondary) |
| `shell-api/` | Node.js | Shell command execution API |
| `log-io/` | Node.js | Log streaming server + React UI |

### Backend Core (Go)
| Module | Language | Description |
|--------|----------|-------------|
| `calculations/` | Go | Real-time formula/calculation engine |
| `i104m/` | Go | IEC 60870-5-104 protocol driver |
| `plc4x-client/` | Go | Apache PLC4X multi-protocol driver |

### Protocol Drivers (.NET/C#)
| Module | Language | Description |
|--------|----------|-------------|
| `OPC-UA-Client/` | C# | OPC-UA client |
| `OPC-UA-Server/` | Node.js | OPC-UA server (node-opcua) |
| `OPC-DA-Client/` | C# | OPC Classic DA/AE client |
| `opcdaaehda-client-solution-net/` | C# | Extended OPC DA/AE/HDA .NET client |
| `S7CommPlusClient/` | C# | Siemens S7 PLC driver |
| `iec61850_client/` | C# | IEC 61850 substation client |
| `lib60870.netcore/` | C# | IEC 60870-5-101/104 library |
| `dnp3/` | C++ | DNP3 protocol driver |
| `alarm_beep/` | C# | Alarm audio notification |

### IoT / Messaging
| Module | Language | Description |
|--------|----------|-------------|
| `mqtt-sparkplug/` | Node.js | MQTT Sparkplug B host/device |
| `telegraf-listener/` | Node.js | Telegraf metrics ingestion |

### Utilities & Tools
| Module | Language | Description |
|--------|----------|-------------|
| `mcp-json-scada-db/` | Node.js | MCP server for AI tool access to DB |
| `backup-mongo/` | Shell | MongoDB backup utility |
| `certificate-creator/` | Node.js | TLS certificate generation |
| `camera-onvif/` | Node.js | ONVIF camera integration |
| `demo_simul/` | Node.js | Data simulator for demos |
| `grafana_alert2event/` | Node.js | Grafana alert → SCADA event bridge |
| `inkscape-extension/` | Python | Inkscape SVG tag-binding extension |
| `updateUser/` | Node.js | CLI user management tool |
| `oshmi2json/` | Node.js | OSHMI config converter |
| `oshmi_sync/` | Node.js | OSHMI sync utility |
| `custom-developments/` | Mixed | User customization directory |

### C/C++ Libraries
| Module | Language | Description |
|--------|----------|-------------|
| `libiec61850/` | C | IEC 61850 C library (libiec61850) |
| `libplctag/` | C | libplctag PLC communication library |
| `mongo-cxx-driver/` | C++ | MongoDB C++ driver (for DNP3) |

---

## AdminUI Structure (`src/AdminUI/src/`)

```
src/
├── main.js               # App entry point
├── App.vue               # Root component
├── i18n.js               # Internationalization setup
├── pages/                # Page-level components (route targets)
│   ├── index.vue
│   ├── LoginPage.vue
│   ├── DashboardPage.vue
│   ├── AdminPage.vue
│   ├── AlarmsViewerPage.vue
│   ├── EventsViewerPage.vue
│   ├── TabularViewerPage.vue
│   ├── DisplayViewerPage.vue
│   ├── TagsTab.vue
│   ├── ProtocolConnectionsTab.vue
│   ├── ProtocolDriverInstancesTab.vue
│   ├── UserManagementTab.vue
│   ├── RolesManagementTab.vue
│   ├── SystemSettingsTab.vue
│   └── ... (more)
├── components/           # Shared components (mirrors pages)
├── router/               # Vue Router configuration
├── plugins/              # Vuetify, i18n plugin registration
├── assets/               # Static assets
├── styles/               # Global CSS/SCSS
└── locales/              # i18n translation files
```

---

## Configuration Files

| Path | Purpose |
|------|---------|
| `conf/json-scada.json` | Active runtime config (MongoDB connection, node name) |
| `conf-templates/json-scada.json` | Config template |
| `conf-templates/config_viewers.js` | Viewer/display configuration |
| `supervisord.conf` | Process manager — starts all services |
| `platform-ubuntu-2404/supervisor/` | Per-service supervisor .conf files |
| `platform-ubuntu-2404/nginx/` | Nginx reverse proxy config |
| `platform-ubuntu-2404/telegraf/` | Telegraf collection config |
| `platform-ubuntu-2404/postgresql/` | PostgreSQL init scripts |

---

## Naming Conventions

**Files:**
- Node.js services: `kebab-case` (e.g., `cs-data-processor.js`, `load-config.js`)
- Vue components: `PascalCase.vue` (e.g., `AdminPage.vue`, `TagsTab.vue`)
- TypeScript: `kebab-case.ts` (e.g., `connection-manager.ts`)
- Go: `snake_case` or single-word packages

**Modules:**
- Each protocol driver = one directory in `src/`
- Each Node.js service has: `index.js` (entry), `load-config.js`, `app-defs.js`, `simple-logger.js`

**Branches:**
- `master` is the main development branch

---

*Structure analysis: 2026-03-17*

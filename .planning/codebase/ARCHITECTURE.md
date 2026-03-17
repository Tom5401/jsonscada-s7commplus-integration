# Architecture

**Analysis Date:** 2026-03-17

## Pattern Overview

**Overall:** Modular Distributed Microservices with MongoDB-Centric Event Processing

**Key Characteristics:**
- MongoDB as the primary real-time database and single source of truth
- Event-driven architecture using MongoDB Change Streams for real-time data propagation
- Loosely coupled independent service modules communicating through MongoDB collections
- Multi-language support (Node.js, C#/.NET, Go, C/C++) allowing best-fit technology choices per module
- Horizontal scalability through independent module deployment and MongoDB sharding
- Protocol-specific driver modules that bridge external industrial protocols (IEC60870, IEC61850, OPC-UA, DNP3, MQTT, Modbus, etc.) to the core database
- Stateless, redundancy-capable services enabling failover and load balancing

## Layers

**Presentation Layer:**
- Purpose: Web-based user interface for visualization, configuration, and monitoring
- Location: `src/AdminUI`
- Contains: Vue 3 SPA, Vuetify components, Inkscape-based SVG editor integration
- Depends on: Realtime Data Server API, Server Realtime Auth API
- Used by: End users via HTTP/HTTPS browsers

**API & Real-Time Data Layer:**
- Purpose: Provide HTTP/WebSocket APIs for external consumers and internal frontends; serve real-time point data with authentication
- Location: `src/server_realtime_auth` (primary HTTP/WebSocket server), `src/graphql-server` (GraphQL interface), `src/shell-api` (shell command API)
- Contains: Express.js servers, WebSocket connections, JWT authentication, GridFS file serving, GraphQL endpoint via PostGraphile
- Depends on: MongoDB (realtimeData, soeData, config collections), PostgreSQL (historian data)
- Used by: AdminUI, custom applications, external integrations

**Protocol Driver Layer:**
- Purpose: Acquire data from external devices/systems using industry standard protocols; translate protocol values to core data model
- Location: `src/{protocol}-client` and `src/{protocol}-server` directories (e.g., `src/OPC-UA-Client`, `src/IEC61850_client`, `src/dnp3`, `src/mqtt-sparkplug`, etc.)
- Contains: Protocol-specific client/server implementations (IEC60870-5-104, IEC60870-5-101, IEC61850 MMS, DNP3, OPC-UA, OPC-DA, MQTT/Sparkplug-B, PLC4X Modbus, libplctag Ethernet/IP, Telegraf listener)
- Depends on: MongoDB for config (protocolConnections, protocolDriverInstances) and real-time updates, external network services
- Used by: Core system for point acquisition and control commands

**Data Processing & Transformation Layer:**
- Purpose: Process raw protocol data into analogs/statuses; perform calculations; handle custom business logic
- Location: `src/cs_data_processor` (MongoDB Change Stream monitor), `src/calculations` (cyclic calculation engine), `src/cs_custom_processor` (user-defined scripts)
- Contains: Change Stream listeners, data conversion logic, calculation expressions, custom processors
- Depends on: MongoDB (realtimeData collection for input, reads/writes via Change Streams), PostgreSQL for historian writes
- Used by: Core system for enriched data availability

**Configuration & Management Layer:**
- Purpose: Store and manage system configuration; handle user roles, permissions, and system settings
- Location: `src/shell-api` (shell commands), `src/server_realtime_auth` (admin endpoints), `src/config_server_for_excel` (Excel-based config)
- Contains: Configuration API endpoints, Excel export/import, RBAC enforcement
- Depends on: MongoDB (config collections: devices, protocols, roles, users), LDAP/AD for directory integration
- Used by: Administrators via AdminUI or external tools

**Data Persistence & Historian Layer:**
- Purpose: Archive historical time-series data for analysis and reporting; integrate with visualization tools
- Location: PostgreSQL/TimescaleDB databases, `src/backup-mongo` (MongoDB backup utility)
- Contains: Time-series historian schema, indexes, archival logic
- Depends on: MongoDB for source data, PostgreSQL drivers from cs_data_processor
- Used by: Grafana dashboards, custom reporting, data analytics

**Support & Utility Services:**
- Purpose: Specialized integrations and auxiliary functions
- Location: `src/alarm_beep`, `src/camera-onvif`, `src/grafana_alert2event`, `src/carbone-reports`, `src/log-io`, `src/custom-developments`, `src/mcp-json-scada-db`
- Contains: Audio alarms, camera/video integration, Grafana alert ingestion, report generation, logging aggregation, MCP server for AI integration
- Depends on: MongoDB, external services (cameras, report engines, log aggregators)
- Used by: Operations teams, integrations

## Data Flow

**Real-Time Data Acquisition & Processing:**

1. Protocol driver reads data from external device (e.g., IEC60870-5-104 server)
2. Driver authenticates with MongoDB, writes raw value to `realtimeData` collection
3. MongoDB Change Stream in `cs_data_processor` detects the write
4. Data processor applies conversion formulas (raw→analog/status) based on point config
5. Processor updates same `realtimeData` document with converted values
6. Calculation engine (Go-based) performs cyclic calculations on dependent points
7. Updated values propagate to AdminUI via WebSocket connections from `server_realtime_auth`
8. Historical archiver writes to PostgreSQL/TimescaleDB for persistence
9. Grafana queries PostgreSQL for dashboard visualization

**Command Execution (Control):**

1. AdminUI user issues command (e.g., switch open/close) via HTTP POST to `server_realtime_auth`
2. Server validates user permissions against MongoDB roles collection
3. Command enqueued in `commandsQueue` collection with timestamp and user context
4. Protocol driver Change Stream listener detects new command
5. Driver formats command per protocol rules and transmits to remote device
6. Driver updates command status in queue document (acknowledged/completed/failed)
7. AdminUI polls/WebSocket receives feedback and updates UI

**Configuration Update:**

1. Administrator modifies config in AdminUI (e.g., adds new point)
2. Configuration persisted to MongoDB `pointConfig` collection
3. All driver processes monitor `protocolConnections` and `pointConfig` collections
4. Drivers detect changes and hot-reload configuration without restart
5. New point immediately available for acquisition

**State Management:**

- All state resides in MongoDB collections; no in-process state persists across service restarts
- Each service instance is stateless and can be restarted independently
- Redundancy achieved through multiple instances monitoring same collections
- Redundancy.js modules in drivers handle election logic (primary/standby patterns)
- MongoDB's replica set provides consistency and failover at database layer

## Key Abstractions

**Point (Tag):**
- Purpose: Represents a single data point (sensor, setpoint, status signal)
- Examples: `src/cs_data_processor/cs_data_processor.js` processes point documents; `src/server_realtime_auth/index.js` serves point data via REST/WebSocket
- Pattern: MongoDB document with fields: `_id`, `name`, `tag`, `deviceId`, `protocolDriverInstanceId`, `rawValue`, `value`, `status`, `timeTag`, `sourceConnectionId`, `alarmState`, `formula`, `kStatus`, `kConv`, etc.

**Protocol Connection:**
- Purpose: Represents a single connection to a remote device/system (e.g., one IEC60870-5-104 TCP connection)
- Examples: `src/OPC-UA-Client`, `src/iec61850_client`, `src/dnp3/Dnp3Client`
- Pattern: MongoDB document in `protocolConnections` collection with: connectionId, protocol type, hostname, port, credentials, redundancy settings

**Protocol Driver Instance:**
- Purpose: Represents a running process instance of a protocol driver
- Examples: Could run 3 instances of DNP3Client for redundancy/load distribution
- Pattern: MongoDB document tracking instance ID, process status, points assigned, last heartbeat timestamp

**Command:**
- Purpose: Represents a control action to be sent to a remote device
- Examples: `src/server_realtime_auth` enqueues; protocol drivers dequeue and execute
- Pattern: MongoDB document in `commandsQueue` with: commandId, pointId, value, timestamp, status (new/sent/acknowledged/completed/failed), userId

**Calculation Formula:**
- Purpose: Encodes derivation logic for computed points (e.g., "temperature_avg = (temp1 + temp2) / 2")
- Examples: `src/calculations/calculations.go` evaluates formulas; formula stored as string in point document `formula` field
- Pattern: String expression or code snippet evaluated by calculation engine against other point values in real-time

## Entry Points

**Web UI Server:**
- Location: `src/AdminUI/src/main.js`
- Triggers: `npm run dev` (development) or `npm run build && serve` (production)
- Responsibilities: Initialize Vue 3 app, configure Vuetify theme, set up router, mount i18n, bootstrap component tree

**Real-Time Data API Server:**
- Location: `src/server_realtime_auth/index.js`
- Triggers: `node index.js` (CLI launch, typically managed by supervisor/systemd)
- Responsibilities: Listen on HTTP port (default 8080), expose REST API for point queries, serve WebSocket for real-time updates, enforce JWT authentication, proxy to GraphQL server

**GraphQL Server:**
- Location: `src/graphql-server/index.js`
- Triggers: `node index.js`
- Responsibilities: Expose GraphQL endpoint connected to PostgreSQL historian database via PostGraphile, enable complex queries on historical data

**Change Stream Data Processor:**
- Location: `src/cs_data_processor/cs_data_processor.js`
- Triggers: `node cs_data_processor.js [instance] [logLevel] [configFile]`
- Responsibilities: Monitor MongoDB Change Stream on realtimeData collection, apply conversion formulas, update processed values, write history to PostgreSQL

**Calculation Engine:**
- Location: `src/calculations/calculations.go`
- Triggers: `go run calculations.go` or compiled binary
- Responsibilities: Perform cyclic evaluations of formula-based points, update derived point values at configured intervals

**Protocol Drivers:**
- Example locations: `src/OPC-UA-Client`, `src/dnp3/Dnp3Client`, `src/iec61850_client`
- Triggers: `node client.js` or `dotnet run` or compiled binaries, managed by supervisor
- Responsibilities: Connect to remote device per protocol, poll/subscribe for data, write to MongoDB realtimeData collection, listen for commands in commandsQueue collection

## Error Handling

**Strategy:** Graceful degradation with logging and optional alerting

**Patterns:**
- Try-catch blocks wrap MongoDB operations with fallback logging (see `src/server_realtime_auth/index.js` exception handlers)
- Change Stream listeners handle connection drops and reconnect with exponential backoff
- Protocol drivers catch communication exceptions and log; point status marked as BAD/TIMEOUT; redundant instances continue operation
- Async operations use Promise.catch() or async-await with try-catch
- Uncaught exceptions logged to console/file; process may exit if critical (e.g., database unreachable)
- Custom logger (`src/server_realtime_auth/simple-logger.js`, used across modules) writes timestamped messages to file and stdout

## Cross-Cutting Concerns

**Logging:** Distributed across modules; each uses `simple-logger.js` pattern (see `src/cs_data_processor/simple-logger.js`, `src/server_realtime_auth/simple-logger.js`) writing to local files and console; aggregated by log-io or external aggregators

**Validation:** MongoDB schema validation enforced at collection level; point config validated on insert/update; user input sanitized in AdminUI before API calls

**Authentication:** JWT tokens issued by `server_realtime_auth/app/controllers/auth.controller.js`; validated on API endpoints; LDAP/AD integration via LDAP client library in auth controller

**Authorization:** RBAC enforced in API layer; roles and permissions stored in MongoDB `roles` collection; API routes check user role against required permissions

**Redundancy:** Multi-instance design; primary/standby election via Redundancy.js modules; MongoDB replica sets provide backend HA

---

*Architecture analysis: 2026-03-17*

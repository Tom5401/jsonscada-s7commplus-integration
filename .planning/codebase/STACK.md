# Technology Stack

**Analysis Date:** 2026-03-17

## Languages

**Primary:**
- Node.js (JavaScript/TypeScript) 24.x - Backend services, data processors, authentication, config servers
- Go 1.24.0 - Protocol implementations (IEC 104, calculations, PLC drivers)
- C# (.NET 8.0) - IEC 60870, IEC 61850, OPC-UA clients and servers, DNP3
- C - IEC 61850 library (libiec61850) via CMake/C++

**Secondary:**
- Bash/Shell - Docker build scripts, initialization, platform-specific setup
- SQL - PostgreSQL database schemas
- HTML/CSS/Vue 3 - Frontend UI (AdminUI)

## Runtime

**Environment:**
- Docker container (Ubuntu 24.04 base image)
- Process management via Supervisor

**Package Manager:**
- npm (Node.js packages)
- Go modules (Go packages)
- NuGet (via dotnet for .NET packages)

**Lockfile:** Present in Node.js projects (package.json), Go projects (go.mod)

## Frameworks

**Core Infrastructure:**
- Node.js 24.x (runtime)
- .NET 8.0 SDK (C# services)
- Go 1.24.0 (CLI tools and servers)

**Frontend:**
- Vue 3.4.31 - Component framework for AdminUI
- Vuetify 3.10.8 - Vue component library
- Vite 7.0.6 - Build tool and dev server
- Vue Router 4.4.0 - Frontend routing

**Backend Frameworks:**
- Express 4.17.1 - HTTP server framework (used in config_server_for_excel, graphql-server, server_realtime_auth)
- Apollo Server 5.0.0 - GraphQL server (server_realtime_auth)
- PostGraphile 4.12.4 - GraphQL API over PostgreSQL (graphql-server)
- Node-OPCUA 2.124.0 - OPC-UA server implementation

**Testing:**
- No testing framework detected in production code

**Build/Dev:**
- TypeScript 5.5.0 - Type checking for Node.js services
- ESLint 8.57.1 - Code linting (AdminUI)
- Vite - Frontend build tool
- tsx - TypeScript execution for development

## Key Dependencies

**Critical - Database Drivers:**
- MongoDB driver (Node.js) 7.0.0 - Primary data store driver, used across all data processors
- MongoDB driver (Go) go.mongodb.org/mongo-driver v1.17.4 - For Go services
- PostgreSQL driver (Node.js) pg 8.11.5 - For carbone-reports, server_realtime_auth
- MongoDB.Driver (.NET) 3.4.2 - For .NET services

**Critical - Protocols & Integration:**
- node-opcua 2.124.0 - OPC-UA server
- OPCFoundation.NetStandard.Opc.Ua.Client 1.5.376.244 - .NET OPC-UA client
- OPCFoundation.NetStandard.Opc.Ua.Configuration 1.5.376.244 - OPC-UA configuration
- Apache PLC4X (plc4go, plc4x-client) - Multi-protocol industrial driver framework
- Sparkplug Client 3.2.2 - MQTT Sparkplug protocol support
- MQTT 5.7.0 - MQTT broker communication

**Authentication & Security:**
- jsonwebtoken 9.0.0 - JWT token handling
- bcryptjs 3.0.3 - Password hashing
- ldapts 8.0.36 - LDAP authentication support
- Mongoose 9.1.1 - MongoDB ODM (server_realtime_auth)

**Monitoring & Utilities:**
- Winston 3.13.0 - Logging (mqtt-sparkplug)
- Carbone 3.2.3 - Report generation
- Queue-FIFO 0.2.6 - In-memory queue (data processors)
- Grafana (3000) - Metrics visualization
- Telegraf - Time-series data collection
- Metabase - Business analytics and dashboards

**UI & Components:**
- Material Design Icons (MDI) 7.4.47 - Icon library
- Lucide Vue Next 0.441.0 - Vue icon components
- Vue I18n 11.1.2 - Internationalization
- Core-js 3.37.1 - JavaScript polyfills
- Sass 1.77.6 - CSS preprocessing

**Development Utilities:**
- unplugin-vue-components 0.27.2 - Auto-import Vue components
- unplugin-vue-router 0.10.0 - Vue router generation
- vite-plugin-vuetify 2.0.3 - Vuetify integration

## Infrastructure & External Services

**Container Runtime:**
- Docker (building and running containerized services)
- Supervisor - Process management for multiple services

**Databases:**
- MongoDB 8.0 - NoSQL document database (primary)
- PostgreSQL 18 + TimescaleDB - Time-series and relational data
- SQLite 3 - Lightweight relational database support

**Message Brokers & Protocols:**
- MQTT 5.x - Message queuing for industrial protocols
- Grafana alerts via webhook integration

**Web Server:**
- Nginx - Reverse proxy, static file serving

**Monitoring & Collection:**
- Telegraf - Data collection agent
- Grafana - Metrics visualization
- Metabase - BI/Analytics

**C++ Libraries:**
- mongo-cxx-driver 4.0.0 - C++ MongoDB driver (for DNP3 Server)
- OpenDNP3 - DNP3 protocol implementation

**Utilities:**
- ffmpeg - Video processing
- OpenJava 21 JDK - Java support for third-party integrations

## Configuration

**Environment:**
Configuration loaded from:
- JSON file: `conf/json-scada.json` (default configuration)
- Environment variables with prefix pattern: `JS_*` (service-specific: `JS_CSDATAPROC_*`, `JS_MCPJSDB_*`, etc.)
- Supervisor config files in `/etc/supervisor/conf.d/`
- Database connection strings from environment or config file

**Key Configuration Files:**
- `conf-templates/json-scada.json` - Main configuration template
- `conf-templates/config_viewers.js` - Viewer configuration
- `supervisord.conf` - Supervisor daemon configuration
- `platform-ubuntu-2404/*` - Platform-specific configs (Telegraf, PostgreSQL, Nginx, Supervisor services)

**MongoDB Configuration:**
- Connection string: `mongodb://127.0.0.1:27017/json_scada?replicaSet=rs1&readPreference=primary&directConnection=true&ssl=false`
- Database name: `json_scada`
- Replica set name: `rs1`

**PostgreSQL Configuration:**
- Default database: `json_scada`
- Default user: `postgres`
- Port: 5432
- Additional databases: `grafanaappdb`, `metabaseappdb`

## Platform Requirements

**Development:**
- Linux/Ubuntu 24.04 recommended (Windows/macOS via Docker)
- Node.js 24.x
- Go 1.24.0
- .NET 8.0 SDK
- CMake 3.x
- Git
- Build tools (gcc, make)

**Production:**
- Docker container deployment
- Minimum: 2GB RAM, 10GB storage
- Ports exposed:
  - 80 (HTTP)
  - 443 (HTTPS)
  - 3000 (Grafana)
  - 4840 (OPC-UA)
  - 5432 (PostgreSQL)
  - 8080 (Node.js apps)
  - 9000 (Supervisor)
  - 20000 (DNP3)
  - 2404 (IEC 104)
  - 27017 (MongoDB)

**Build Environment Variables:**
- `DOTNET_CLI_TELEMETRY_OPTOUT=1` - Disable .NET telemetry
- `NODE_OPTIONS=--max-old-space-size=10000` - Increase Node heap
- `GO111MODULE=auto` - Go module mode
- `CGO_ENABLED=1` - Enable CGO for C bindings

---

*Stack analysis: 2026-03-17*

# External Integrations

**Analysis Date:** 2026-03-17

## Databases

### MongoDB (Primary Data Store)
- **Version:** 8.0
- **Connection:** `mongodb://127.0.0.1:27017/json_scada?replicaSet=rs1&readPreference=primary&directConnection=true&ssl=false`
- **Database:** `json_scada`
- **Replica set:** `rs1`
- **Usage:** Central data hub for all real-time process values, alarms, protocol config
- **Change Streams:** All services subscribe via MongoDB Change Streams for real-time updates
- **Key collections:** `realtimeData`, `protocolDriverInstances`, `protocolConnections`, `users`, `roles`, `commandsQueue`
- **Drivers used:** Node.js `mongodb` v7.0.0, Go `go.mongodb.org/mongo-driver` v1.17.4, .NET `MongoDB.Driver` v3.4.2, C++ `mongo-cxx-driver` v4.0.0

### PostgreSQL + TimescaleDB (Historical Data)
- **Version:** PostgreSQL 18 + TimescaleDB extension
- **Default DB:** `json_scada`
- **Port:** 5432
- **Additional databases:** `grafanaappdb`, `metabaseappdb`
- **Usage:** Time-series historical data for process values, Grafana datasource, Metabase analytics

### SQLite
- **Usage:** Lightweight relational data for specific drivers (e.g., carbone-reports)

---

## Industrial Protocols

### IEC 60870-5-104 (IEC 104)
- **Module:** `src/i104m/` (Go)
- **Library:** Apache PLC4X (`plc4go`)
- **Port:** 2404
- **Role:** Client/server for electric utility SCADA communication

### IEC 60870-5-101 (Serial)
- **Module:** `src/lib60870.netcore/` (C#)
- **Library:** `lib60870` (C/C#)
- **Role:** Serial-based legacy utility protocol

### IEC 61850
- **Module:** `src/iec61850_client/` (C#)
- **Library:** `libiec61850`
- **Role:** Substation automation client

### DNP3
- **Module:** `src/dnp3/` (C++)
- **Library:** `OpenDNP3`, `mongo-cxx-driver`
- **Port:** 20000
- **Role:** Distributed Network Protocol for SCADA

### OPC-UA
- **Server module:** `src/OPC-UA-Server/` (Node.js, `node-opcua` v2.124.0)
- **Client module:** `src/OPC-UA-Client/` (C#, `OPCFoundation.NetStandard.Opc.Ua`)
- **Port:** 4840
- **Role:** OPC Unified Architecture server and client

### OPC-DA / OPC-DA/AE (Legacy)
- **Module:** `src/OPC-DA-Client/`, `src/opcdaaehda-client-solution-net/` (C#)
- **Role:** Legacy OPC Classic client for Windows DCS/SCADA

### MQTT / Sparkplug B
- **Module:** `src/mqtt-sparkplug/` (Node.js)
- **Library:** `mqtt` v5.7.0, `sparkplug-client` v3.2.2
- **Role:** IoT message broker; Sparkplug B device/host node support
- **Auto-tag discovery:** `auto-tag.js` for dynamic tag registration

### Modbus / PLC4X Multi-Protocol
- **Module:** `src/plc4x-client/` (Go)
- **Library:** Apache PLC4X `plc4go`
- **Role:** Unified driver for Modbus, EtherNet/IP, S7, BACnet, and more

### S7 (Siemens)
- **Module:** `src/S7CommPlusClient/` (C#)
- **Role:** Siemens PLC S7 protocol client

### Telegraf Listener
- **Module:** `src/telegraf-listener/` (Node.js)
- **Role:** Receives Telegraf time-series data; maps to JSON-SCADA realtime values

---

## Authentication & Authorization

### JWT Tokens
- **Library:** `jsonwebtoken` v9.0.0
- **Usage:** Session tokens issued by `server_realtime_auth`; verified on API calls

### LDAP
- **Library:** `ldapts` v8.0.36
- **Usage:** LDAP/AD authentication in `server_realtime_auth`; users can authenticate against enterprise directory

### bcrypt Password Hashing
- **Library:** `bcryptjs` v3.0.3
- **Usage:** Local user password storage

---

## External Monitoring & Analytics

### Grafana
- **Port:** 3000
- **Role:** Metrics dashboards; consumes PostgreSQL/TimescaleDB
- **Integration:** `src/grafana_alert2event/` converts Grafana alerts → JSON-SCADA events

### Metabase
- **Database:** `metabaseappdb` (PostgreSQL)
- **Role:** Business analytics and self-service dashboards

### Telegraf
- **Role:** Time-series data collection agent; feeds PostgreSQL/TimescaleDB
- **Config:** `platform-ubuntu-2404/telegraf/`

---

## Web / HTTP Services

### Nginx (Reverse Proxy)
- **Ports:** 80 (HTTP), 443 (HTTPS)
- **Role:** Routes external requests to internal services (AdminUI, GraphQL, Grafana, Metabase)
- **Config:** `platform-ubuntu-2404/nginx/`

### GraphQL API
- **Module:** `src/graphql-server/` (Node.js, PostGraphile v4.12.4)
- **Role:** Auto-generated GraphQL over PostgreSQL

### Apollo GraphQL Server
- **Module:** `src/server_realtime_auth/` (Apollo Server v5.0.0 + Express v5)
- **Role:** Authenticated real-time data API with WebSocket subscriptions via Socket.IO

---

## Report Generation

### Carbone
- **Module:** `src/carbone-reports/` (Node.js)
- **Library:** `carbone` v3.2.3
- **Role:** Document/PDF report generation from JSON templates

---

## Video / Camera

### ONVIF Camera
- **Module:** `src/camera-onvif/`
- **Role:** IP camera integration for process video in SCADA displays

### FFmpeg
- **Role:** Video processing for ONVIF streams

---

## SVG Display System

### Inkscape Extension
- **Module:** `src/inkscape-extension/`
- **Role:** Design process display SVGs with data-binding tags

### SVG Editor
- **Module:** `src/svg-display-editor/`, `src/svgedit/`
- **Role:** Browser-based SVG editor for process graphics

---

## MCP Server (AI Integration)

### json-scada MCP Server
- **Module:** `src/mcp-json-scada-db/`
- **Role:** Model Context Protocol server exposing JSON-SCADA database to AI/LLM tooling

---

*Integrations analysis: 2026-03-17*

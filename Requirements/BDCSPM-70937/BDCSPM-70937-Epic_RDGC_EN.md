# BDCSPM-70937 — Epic: Port MQTT Protocol Monitoring from Probe Server to RDGC

## Capability Reference

| Field | Value |
|---|---|
| **Capability** | BDCSPM-70937: RDGC has the ability to monitor devices using MQTT |
| **Parent** | BDCSPM-6550: RDGC monitoring protocol parity with BLSS |
| **Fix Version** | BLSS 8.2.0 (2027-01-15) |
| **Priority** | High |
| **Dependency** | BDCSPM-6541: RDGC device multi-protocol auto discovery |
| **Labels** | Customer-SustainingEngineering, DIT, RDG |
| **Reference Implementation** | Probe Server — MQTT Protocol Handler (production) |

---

### EPIC: BDCSPM-70937-RDGC — Port MQTT Protocol Monitoring from Probe Server to RDGC

**BLSS Product(s)**: Distributed IT Performance Management (RDGC component)

**Summary**
Port the existing MQTT protocol monitoring capability from Probe Server to RDGC (Remote Data Gateway Client), leveraging the proven Protocol Handler architecture, Monitor Configuration model, and Monitoring Templates mechanism already in production on Probe Server.

**Description**

- **Problem statement**: Probe Server already supports MQTT protocol monitoring (MQTT is listed in the Monitor Configuration protocol list alongside SNMP, Modbus, BACnet, IPMI, OPC UA, etc.). However, RDGC — which serves edge and remote DIT (Distributed IT) sites — does not yet support MQTT. Customers deploying MQTT-enabled devices at edge locations managed by RDGC cannot monitor these devices through Brightlayer.
- **Proposed solution**: Extract the Probe Server MQTT Protocol Handler into a shared module and integrate it into RDGC's existing protocol plugin architecture. Reuse the following proven technologies:
  - Protocol Handler plugin architecture (coexisting with SNMP, Modbus, BACnet, OPC UA, etc.)
  - Monitor Configuration model (IP Address, Probe/RDG association, protocol selection)
  - Monitoring Templates mechanism for defining collection points
  - Protocol configuration panel UI pattern (right-side parameter panel, as seen for SNMP)
  - Device binding model (Device → Monitor Config → RDG Client association)
- **User personas**: IT/OT Administrators, Operations Teams, System Integrators
- **Business value**:
  - Achieves RDGC-to-Probe Server protocol parity for MQTT
  - Leverages battle-tested implementation to significantly reduce development risk and effort
  - Enables MQTT device monitoring for DIT edge-site customers
  - Increases device density per license, driving revenue
- **Key workflows** (aligned with existing Probe Server flow):
  1. Administrator selects MQTT protocol in Device Monitor Configuration
  2. Configures MQTT-specific parameters (Broker Host/Port, Topic, QoS, Auth, TLS)
  3. Associates a Monitoring Template to define collection points
  4. RDGC establishes MQTT connection and subscribes to topics
  5. Ingested data flows into existing monitoring pipeline (Graphs, Alarms, Traps)
  6. Device Dashboard displays connection status and communication health
- **Cross-product touchpoints**: Shares Protocol Handler codebase with Probe Server; Monitor Configuration model consistency across products
- **UX/UI references**:
  - Existing Probe Server Monitor Config page (see attached screenshot)
  - MQTT appears in protocol list (already exists on Probe Server)
  - MQTT protocol parameter panel follows the same layout as SNMP panel (right-side configuration)
- **On-prem operational impact**:
  - Install/upgrade impact: RDGC package includes new MQTT Protocol Handler module; configuration schema extension
  - Sizing impact: Equivalent resource profile to Probe Server under same load; validate on RDGC edge hardware
  - Network impact: Outbound port 8883 (MQTTS) / 1883 (MQTT); firewall documentation update required
  - Backup impact: MQTT configuration data included in standard RDGC backup

---

## Existing Implementation Reference (Probe Server)

The following capabilities exist on Probe Server and should be directly reused for RDGC:

| Existing Capability | Probe Server Implementation | RDGC Porting Strategy |
|---|---|---|
| Protocol list | Monitor Config → Protocol List includes MQTT | Reuse protocol registration mechanism; register MQTT Handler in RDGC |
| Protocol config panel | Right-side parameter panel (reference: SNMP panel with Port, Protocol, Version, Community, Security) | Implement MQTT-specific panel: Broker Host/Port, ClientID, QoS, Topic, Auth, TLS |
| Monitoring Templates | Supports per-protocol collection templates | Reuse Template engine; add MQTT template type |
| Device binding | Device → Monitor Config → Probe/RDG Server association | Replace Probe with RDGC; maintain Device → Monitor Config model |
| Connection management | Probe Server manages protocol connections | RDGC manages MQTT connections locally (no Probe relay needed) |
| JSON payload parsing | Probe Server parses MQTT message payloads | Extract as shared module; reuse in RDGC |

---

## Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Security | MQTT connection encryption | TLS 1.2+ mandatory; aligned with Probe Server | P0 |
| Security | Credential storage | Reuse RDGC existing encrypted storage mechanism | P0 |
| Reliability | Auto-reconnect on broker disconnect | First retry within 30s; exponential backoff to max 5 min | P0 |
| Scalability | MQTT devices per RDGC instance | ≥ 50 MQTT devices (RDGC hardware constraint) | P1 |
| Scalability | Topic subscriptions | ≥ 200 topics per RDGC instance | P1 |
| Performance | MQTT module additional CPU overhead | < 10% on RDGC edge hardware baseline | P1 |
| Performance | Additional memory footprint | < 128 MB | P1 |
| Compatibility | MQTT protocol versions | v3.1.1 and v5.0 (aligned with Probe Server) | P1 |
| Compatibility | Coexistence with existing protocols | SNMP/Modbus/BACnet/OPC UA unaffected | P0 |
| Observability | Connection status | Visible in device Dashboard | P1 |

---

## Acceptance Criteria (Epic-level)

- [ ] AC1: MQTT appears in RDGC Monitor Configuration protocol list (consistent with Probe Server)
- [ ] AC2: Administrator can configure MQTT protocol parameters via UI (Broker, Port, Topic, Auth, TLS)
- [ ] AC3: RDGC establishes TLS-encrypted MQTT connection and subscribes to topics to receive device telemetry
- [ ] AC4: Ingested MQTT data flows into existing monitoring pipeline (Graphs, Alarm Panel, Traps)
- [ ] AC5: Monitoring Templates support MQTT device type templates
- [ ] AC6: Device Dashboard displays MQTT connection status (Connected/Disconnected/Error)
- [ ] AC7: Communication failures are clearly indicated with error details for troubleshooting
- [ ] AC8: MQTT monitoring does not degrade existing SNMP/Modbus/BACnet device monitoring performance
- [ ] AC9: Feature parity with Probe Server MQTT implementation achieved

---

## Scope

**In scope**:
- MQTT Protocol Handler porting to RDGC (code reuse/adaptation from Probe Server)
- MQTT protocol configuration panel UI (following SNMP panel layout pattern)
- MQTT Monitoring Templates support
- TLS encrypted connections (TLS 1.2+)
- Authentication: Username/Password, Client Certificate (mTLS)
- Topic subscription and message reception
- JSON payload parsing (reuse Probe Server parsing logic)
- Device connection status display
- Integration with existing RDGC monitoring pipeline (Graphs, Alarms, Traps)
- Communication timeout detection and alerting
- Device discovery integration (BDCSPM-6541 multi-protocol auto discovery)

**Out of scope**:
- MQTT message publishing (device control)
- MQTT Broker hosting or lifecycle management
- Non-JSON payload formats (Protobuf, MessagePack — future iteration)
- Sub-30-second update guarantees
- Modifications to Probe Server MQTT functionality
- Advanced analytics or ML processing
- Polling Groups adaptation (MQTT is event-driven/push; traditional polling semantics do not apply)

**Assumptions**:
1. ⚠️ ASSUMPTION: Probe Server MQTT Protocol Handler code can be extracted as a standalone shared module for RDGC reuse
2. ⚠️ ASSUMPTION: RDGC and Probe Server share a compatible Protocol Handler Plugin Interface
3. ⚠️ ASSUMPTION: RDGC edge hardware (ARM/x86 compact devices) can run the MQTT client library
4. ⚠️ ASSUMPTION: Existing Monitor Configuration data model can be extended for MQTT parameters without breaking changes
5. ⚠️ ASSUMPTION: Monitoring Templates engine on RDGC is consistent with Probe Server implementation — [TBD — confirm with architecture team]

---

## Dependencies & Risks

| Item | Type | Owner | Mitigation | Status |
|---|---|---|---|---|
| BDCSPM-6541: Multi-protocol auto discovery | Dep | TBD | Coordinate MQTT device discovery mechanism | Ready for Development |
| Probe Server MQTT code modularity | Risk | Architecture | Assess coupling; refactor into shared library if needed | ⚠️ NEEDS VALIDATION |
| RDGC edge hardware resource constraints | Risk | QA | Performance testing on target hardware | Open |
| Protocol Handler interface differences between Probe Server and RDGC | Risk | Dev team | Unified interface abstraction; adapter pattern | Open |
| MQTT client library cross-platform compatibility (ARM/x86) | Risk | Dev team | Confirm Paho C library supports target platforms | Open |
| Monitor Configuration data model compatibility | Risk | Architecture | Schema migration ensuring backward compatibility | Open |

---

## Operational Deliverables (required for on-prem)

- [ ] Installation/upgrade runbook updated (RDGC MQTT module deployment steps)
- [ ] Sizing guide updated (RDGC edge hardware + MQTT load baseline)
- [ ] Firewall/port documentation updated (8883/TLS, 1883/unencrypted)
- [ ] Backup/restore procedure updated (MQTT configuration data)
- [ ] Release notes entry drafted
- [ ] Lifecycle Services training updated (MQTT configuration workflow)
- [ ] User guide updated (MQTT device setup and monitoring configuration)

---

## Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| S1 | Extract Probe Server MQTT Protocol Handler as Shared Module | 8 | Sprint 1 |
| S2 | Register MQTT Protocol Handler in RDGC Plugin Architecture | 5 | Sprint 1 |
| S3 | MQTT Protocol Configuration Panel UI (Monitor Config) | 8 | Sprint 2 |
| S4 | MQTT Connection Management (TLS + Auth + Reconnect) | 8 | Sprint 2 |
| S5 | Topic Subscription and Message Reception Pipeline | 5 | Sprint 3 |
| S6 | JSON Payload Parsing and Parameter Mapping (Reuse Probe Logic) | 5 | Sprint 3 |
| S7 | Monitoring Templates Support for MQTT Device Type | 5 | Sprint 3 |
| S8 | Communication Timeout Detection and Alerting | 5 | Sprint 4 |
| S9 | Device Connection Status and Communication Health Display | 5 | Sprint 4 |
| S10 | Integration with Existing Monitoring Pipeline (Graph/Alarm/Trap) | 8 | Sprint 5 |
| S11 | Performance Validation and Protocol Coexistence Testing (RDGC Hardware) | 5 | Sprint 5 |
| S12 | End-to-End Integration Test and Probe Server Feature Parity Verification | 5 | Sprint 6 |

**Total Estimate**: 72 Story Points / 6 Sprints

---

## Story Details

---

### STORY: S1 — Extract Probe Server MQTT Protocol Handler as Shared Module

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (Shared Component)

**Description**
As a Developer,
I want the Probe Server's MQTT Protocol Handler to be extracted into a reusable shared module,
So that RDGC can leverage the same battle-tested implementation without code duplication.

**Context & notes**:
- Analyze Probe Server MQTT Handler code structure and dependencies
- Identify coupling points with Probe Server infrastructure (logging, config, connection pool, etc.)
- Extract into standalone library/module with a clean Protocol Handler Interface
- Ensure Probe Server continues to function identically (regression testing)

**Acceptance Criteria**

AC1:
  Given: Probe Server MQTT Protocol Handler code is extracted into a standalone module
  When: Probe Server references the new module and runs
  Then: Probe Server MQTT functionality is identical to pre-extraction (all regression tests pass)

AC2:
  Given: The standalone MQTT Handler module
  When: Inspecting its dependencies
  Then: No direct dependency on Probe Server-specific infrastructure (abstracted via interfaces)

AC3:
  Given: Module interface documentation
  When: RDGC team reviews
  Then: Interface satisfies RDGC integration requirements with no blocking issues

**Tasks**
- [ ] Task 1: Analyze Probe Server MQTT Handler code structure and dependency graph
- [ ] Task 2: Define Protocol Handler Interface (abstraction layer)
- [ ] Task 3: Refactor MQTT Handler into standalone module/library
- [ ] Task 4: Replace original Probe Server code with module reference
- [ ] Task 5: Execute Probe Server regression tests
- [ ] Task 6: Write module API documentation and integration guide

**Definition of Ready checklist**
- [ ] Probe Server MQTT source code accessible
- [ ] Architecture team approves modularization approach
- [ ] Regression test suite ready
- [ ] Estimated by team

---

### STORY: S2 — Register MQTT Protocol Handler in RDGC Plugin Architecture

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As a Developer,
I want to register the MQTT Protocol Handler in RDGC's plugin architecture,
So that MQTT appears as an available protocol option in the Monitor Configuration.

**Context & notes**:
- RDGC already has a Protocol Handler registration mechanism (used for SNMP, Modbus, etc.)
- Register the S1 shared MQTT module as an RDGC Protocol Plugin
- After registration, MQTT should appear in Monitor Config → Protocol List
- Implement RDGC-specific adapter connecting the shared module to RDGC runtime

**Acceptance Criteria**

AC1:
  Given: MQTT Protocol Handler module is integrated into the RDGC build
  When: RDGC service starts
  Then: MQTT appears in the available protocol list (alongside SNMP, Modbus, BACnet, etc.)

AC2:
  Given: RDGC loads the MQTT Protocol Handler
  When: System logs are inspected
  Then: Logs show "MQTT Protocol Handler registered successfully"

AC3:
  Given: MQTT Handler is registered in RDGC
  When: User selects MQTT in Monitor Configuration UI
  Then: The protocol-specific configuration panel is activated (even if initially a placeholder)

**Tasks**
- [ ] Task 1: Register MQTT Handler in RDGC Protocol Plugin Registry
- [ ] Task 2: Implement RDGC → Shared Module adapter layer
- [ ] Task 3: Configure MQTT Handler start/stop lifecycle management
- [ ] Task 4: Verify protocol list UI shows MQTT option
- [ ] Task 5: Write unit tests

**Definition of Ready checklist**
- [ ] S1 modularization complete
- [ ] RDGC Plugin Interface documentation confirmed
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: S3 — MQTT Protocol Configuration Panel UI (Monitor Config)

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC — Web UI)

**Description**
As an IT/OT Administrator,
I want an MQTT protocol configuration panel in the Monitor Configuration page,
So that I can configure MQTT connection parameters for a device (following the same pattern as the existing SNMP panel).

**Context & notes**:
- Reference the existing SNMP panel layout (Port, Protocol, Version, Community, Security Level, Auth Protocol, etc.)
- MQTT panel parameters:
  - Broker Host / Port (default 8883)
  - Client ID
  - Protocol Version (v3.1.1 / v5.0)
  - QoS Level (0 / 1)
  - Topic Pattern(s) — support multiple topics
  - Auth Type (None / Username+Password / Client Certificate)
  - Username / Password (masked after save)
  - TLS Enabled (toggle)
  - CA Certificate (file upload or path)
  - Keep Alive Interval (seconds)
- Panel appears on the right side when MQTT protocol is selected
- "Verify" button tests connection (consistent with existing Verify button behavior)

**Acceptance Criteria**

AC1:
  Given: Administrator selects MQTT protocol in Monitor Configuration
  When: Protocol is selected
  Then: Right-side panel displays MQTT configuration with all required parameter fields

AC2:
  Given: MQTT configuration panel is filled with valid parameters
  When: "Verify" button is clicked
  Then: System tests MQTT connection and displays success/failure result within 15 seconds

AC3:
  Given: MQTT configuration is completed
  When: "Submit" is clicked to save
  Then: Configuration is persisted; Password field is encrypted at rest; UI shows "•••••••"

AC4:
  Given: A saved MQTT configuration exists
  When: Administrator reopens Monitor Configuration for the device
  Then: All parameters are correctly populated (password field shows mask)

**Tasks**
- [ ] Task 1: Implement MQTT configuration panel UI component (following SNMP panel layout)
- [ ] Task 2: Implement fields: Broker Host, Port, Client ID, Protocol Version
- [ ] Task 3: Implement fields: QoS, Topic Pattern(s) (support multiple topics)
- [ ] Task 4: Implement auth configuration: Auth Type selector + Username/Password + Certificate
- [ ] Task 5: Implement TLS toggle + CA Certificate configuration
- [ ] Task 6: Implement "Verify" button for connection testing
- [ ] Task 7: Implement configuration persistence API call
- [ ] Task 8: Write UI component tests
- [ ] Task 9: Write E2E tests

**Definition of Ready checklist**
- [ ] UX confirms panel layout (reference SNMP panel)
- [ ] Configuration API endpoint defined
- [ ] RBAC roles confirmed (admin can edit)
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: S4 — MQTT Connection Management (TLS + Auth + Reconnect)

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want RDGC to establish and maintain secure MQTT connections with auto-reconnect,
So that device monitoring is reliable and secure.

**Context & notes**:
- Reuse Probe Server's proven connection management logic
- Adapt to RDGC runtime environment (potentially more resource-constrained)
- TLS 1.2+, Username/Password, mTLS
- Auto-reconnect with exponential backoff
- Connection state published to internal state bus for UI consumption

**Acceptance Criteria**

AC1:
  Given: MQTT configuration is saved and device "Monitored" toggle is ON
  When: RDGC initiates connection
  Then: TLS-encrypted connection is established; CONNACK received

AC2:
  Given: An active MQTT connection
  When: Broker restarts or network interruption occurs
  Then: RDGC attempts first reconnect within 30s; uses exponential backoff (max 5 min)

AC3:
  Given: Successful reconnection after outage
  When: Connection is restored
  Then: All previously configured topic subscriptions are automatically re-established

AC4:
  Given: Invalid credentials configured
  When: RDGC attempts to connect
  Then: Connection rejected; error logged; no retry with same credentials

**Tasks**
- [ ] Task 1: Integrate shared MQTT module's connection manager into RDGC
- [ ] Task 2: Adapt to RDGC TLS/certificate store
- [ ] Task 3: Implement reconnect strategy (exponential backoff, configurable)
- [ ] Task 4: Implement connection state publishing to RDGC internal message bus
- [ ] Task 5: Implement linkage with "Monitored" toggle (ON = connect, OFF = disconnect)
- [ ] Task 6: Write integration tests (normal connect, reconnect, invalid credentials)

**Definition of Ready checklist**
- [ ] S1 + S2 complete
- [ ] RDGC certificate store API documented
- [ ] Test broker environment ready
- [ ] Estimated by team

---

### STORY: S5 — Topic Subscription and Message Reception Pipeline

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As a System Integrator,
I want RDGC to subscribe to configured MQTT topics and receive messages through the data pipeline,
So that device telemetry flows into the monitoring system.

**Context & notes**:
- Topic configuration sourced from S3 Monitor Configuration
- Support wildcards (+, #)
- Messages enter RDGC internal data pipeline
- QoS 0 and QoS 1 support

**Acceptance Criteria**

AC1:
  Given: MQTT connection is established
  When: Configured topic subscriptions are executed
  Then: Messages on matching topics are received and passed into the data pipeline

AC2:
  Given: Wildcard topic (e.g., `site/+/telemetry`) is configured
  When: Messages published on matching topics
  Then: All matching messages are received

AC3:
  Given: Topic subscription fails (e.g., ACL denial)
  When: SUBACK returns error code
  Then: Error logged with topic name and reason; device status reflects subscription failure

**Tasks**
- [ ] Task 1: Implement topic subscription manager (reads topic list from Monitor Config)
- [ ] Task 2: Implement message reception and internal queue routing
- [ ] Task 3: Support wildcard topics (+ and #)
- [ ] Task 4: Implement SUBACK error handling
- [ ] Task 5: Write integration tests

**Definition of Ready checklist**
- [ ] S4 connection management complete
- [ ] Data pipeline interface definition confirmed
- [ ] Estimated by team

---

### STORY: S6 — JSON Payload Parsing and Parameter Mapping (Reuse Probe Logic)

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want RDGC to parse MQTT JSON payloads and map them to device monitoring parameters,
So that device telemetry data is stored in the standard parameter model.

**Context & notes**:
- Reuse Probe Server's existing JSON parsing logic
- Support JSONPath expressions for nested value extraction
- Map to RDGC parameter model (name, value, unit, timestamp, quality)
- Malformed payloads must not affect other devices

**Acceptance Criteria**

AC1:
  Given: MQTT message contains JSON payload `{"temperature": 23.5, "humidity": 45}`
  When: Parser processes the message
  Then: Parameters "temperature" = 23.5 and "humidity" = 45 are correctly extracted and stored

AC2:
  Given: JSONPath mapping rules are configured
  When: Nested JSON payload arrives
  Then: Target values are correctly extracted per mapping rules

AC3:
  Given: Malformed JSON payload received
  When: Parsing fails
  Then: Error logged; message skipped; other device data processing unaffected

**Tasks**
- [ ] Task 1: Integrate shared module JSON parser into RDGC
- [ ] Task 2: Adapt to RDGC parameter model
- [ ] Task 3: Implement JSONPath mapping configuration
- [ ] Task 4: Implement error isolation (per-device processing)
- [ ] Task 5: Write unit tests (various payload formats)

**Definition of Ready checklist**
- [ ] Sample JSON payloads from target MQTT devices collected
- [ ] Probe Server parsing logic documented
- [ ] Estimated by team

---

### STORY: S7 — Monitoring Templates Support for MQTT Device Type

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want to use Monitoring Templates to define monitoring points for MQTT devices,
So that I can quickly onboard new MQTT devices using pre-configured templates (same workflow as SNMP/Modbus).

**Context & notes**:
- Monitoring Templates is an existing feature on Probe Server / RDGC (see "Monitoring Templates" tab in screenshot)
- Define MQTT Template: Topic Pattern + Parameter Mappings + Expected data types
- Support template assignment to Device Type
- Administrator can create custom MQTT templates

**Acceptance Criteria**

AC1:
  Given: Administrator creates an MQTT-type Monitoring Template
  When: Topic pattern and parameter mappings are defined
  Then: Template is saved and available for assignment to MQTT devices

AC2:
  Given: MQTT Template is associated with a device
  When: MQTT messages arrive for that device
  Then: Data is extracted and stored per the template-defined mapping rules

AC3:
  Given: One MQTT Template is assigned to multiple devices of the same type
  When: All devices come online
  Then: All devices use the same mapping rules; data collected correctly

**Tasks**
- [ ] Task 1: Extend Monitoring Template engine to support MQTT protocol type
- [ ] Task 2: Implement MQTT Template configuration fields (Topic, JSONPath Mapping, data types)
- [ ] Task 3: Implement template-to-device association logic
- [ ] Task 4: Support MQTT template creation/editing in the Monitoring Templates UI tab
- [ ] Task 5: Write integration tests

**Definition of Ready checklist**
- [ ] Monitoring Templates architecture documentation confirmed
- [ ] MQTT Template field definitions confirmed
- [ ] Estimated by team

---

### STORY: S8 — Communication Timeout Detection and Alerting

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an Operations Team member,
I want RDGC to detect when an MQTT device stops reporting data and raise an alert,
So that communication failures are identified promptly without manual monitoring.

**Context & notes**:
- MQTT is push/event-driven; there is no polling interval
- Timeout detection based on configurable "Expected Update Interval" per device
- If no message is received within N × expected interval, device status → "Communication Lost"
- Communication Lost events generate alarms and are logged in the event system
- Configurable timeout multiplier (default: 3x expected interval)

**Acceptance Criteria**

AC1:
  Given: MQTT device has Expected Update Interval = 60s and timeout multiplier = 3x
  When: No message received for > 180 seconds
  Then: Device status changes to "Communication Lost"; alarm raised

AC2:
  Given: Device is in "Communication Lost" state
  When: A new message is received from the device
  Then: Device status returns to "Connected"; alarm cleared

AC3:
  Given: Expected Update Interval is configured per device in Monitor Configuration
  When: Administrator sets interval to 30s
  Then: Timeout detection uses 30s × multiplier as the threshold

AC4:
  Given: Communication timeout occurs
  When: Event is generated
  Then: Event includes device name, last message timestamp, and expected interval

**Tasks**
- [ ] Task 1: Add "Expected Update Interval" configuration field to MQTT Monitor Config
- [ ] Task 2: Implement per-device message arrival tracking (last-seen timestamp)
- [ ] Task 3: Implement timeout detection logic (configurable multiplier)
- [ ] Task 4: Integrate with alarm engine (Communication Lost event)
- [ ] Task 5: Implement auto-clear on message receipt recovery
- [ ] Task 6: Write unit tests and integration tests

**Definition of Ready checklist**
- [ ] Timeout multiplier default value agreed (3x)
- [ ] Alarm engine integration point confirmed
- [ ] Estimated by team

---

### STORY: S9 — Device Connection Status and Communication Health Display

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC — Web UI)

**Description**
As an Operations Team member,
I want to see MQTT device connection status and communication health in the device Dashboard,
So that I can quickly identify devices with connectivity issues.

**Context & notes**:
- Display in device Dashboard (see screenshot top tabs: Dashboard, Graphs, Alarm Panel, etc.)
- Status states: Connected / Disconnected / Reconnecting / Communication Lost
- Health indicators: last message time, reconnect count, error count
- Consistent with existing SNMP/Modbus device status display patterns

**Acceptance Criteria**

AC1:
  Given: MQTT device is actively communicating
  When: Viewing device Dashboard
  Then: "Connected" status is displayed (green indicator)

AC2:
  Given: MQTT connection drops
  When: Disconnect event occurs
  Then: Device status updates to "Disconnected" (red indicator) within 10 seconds

AC3:
  Given: Device exceeds timeout threshold without data
  When: Communication Lost threshold reached
  Then: "Communication Lost" displayed with last communication timestamp

AC4:
  Given: Device Dashboard communication panel
  When: Administrator views details
  Then: Shows: last message time, reconnect count, error count, connection duration

**Tasks**
- [ ] Task 1: Implement connection status badge component
- [ ] Task 2: Integrate into device Dashboard view
- [ ] Task 3: Implement communication health detail panel
- [ ] Task 4: Implement real-time status refresh (≤ 10s)
- [ ] Task 5: Write UI tests

**Definition of Ready checklist**
- [ ] UX mockup confirmed
- [ ] Internal state API available (S4 complete)
- [ ] Estimated by team

---

### STORY: S10 — Integration with Existing Monitoring Pipeline (Graph/Alarm/Trap)

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an Operations Team member,
I want MQTT device data to flow into existing Graphs, Alarm Panel, and Traps workflows,
So that MQTT devices are monitored with the same tools and processes as other protocol devices.

**Context & notes**:
- Reference screenshot top tabs: Dashboard, Graphs, Ports, Alarm Panel, Traps, Calendar, Attributes, Monitor
- MQTT-collected parameters should be plottable in Graphs
- Threshold breaches trigger alarms (appear in Alarm Panel)
- Communication anomaly events generate Trap/Event entries
- No changes to existing Graph/Alarm/Trap engines needed — only correct data input format

**Acceptance Criteria**

AC1:
  Given: MQTT device parameter data is stored
  When: Administrator configures a Graph for an MQTT device parameter
  Then: Graph correctly displays historical and real-time trends

AC2:
  Given: MQTT parameter has a threshold alarm configured
  When: Parameter value crosses the threshold
  Then: Alarm entry appears in Alarm Panel

AC3:
  Given: MQTT device communication anomaly (Communication Lost)
  When: Timeout event triggers
  Then: Trap/Event entry is recorded in the event system

AC4:
  Given: Mixed MQTT and SNMP devices
  When: Administrator views Alarm Panel
  Then: MQTT and SNMP device alarms are displayed together; filterable by protocol type

**Tasks**
- [ ] Task 1: Validate MQTT parameter data format compatibility with Graph engine
- [ ] Task 2: Validate MQTT parameter integration with Alarm engine (threshold triggering)
- [ ] Task 3: Implement communication anomaly → Trap/Event recording
- [ ] Task 4: Validate Alarm Panel filter supports protocol type
- [ ] Task 5: Write end-to-end integration tests

**Definition of Ready checklist**
- [ ] Graph/Alarm/Trap engine input interfaces confirmed
- [ ] No engine modifications needed (data integration only) confirmed
- [ ] Estimated by team

---

### STORY: S11 — Performance Validation and Protocol Coexistence Testing (RDGC Hardware)

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As a QA Engineer,
I want to validate MQTT performance and coexistence with other protocols on RDGC target hardware,
So that we confirm the implementation meets NFRs in the production environment.

**Context & notes**:
- RDGC typically runs on edge/remote-site hardware (resource-constrained)
- Test scenario: MQTT + SNMP + Modbus running simultaneously
- Verify CPU, memory, network within NFR bounds
- Verify existing protocols are unaffected by MQTT addition

**Acceptance Criteria**

AC1:
  Given: RDGC running 50 MQTT devices + 100 SNMP devices + 50 Modbus devices
  When: System resources measured
  Then: MQTT additional CPU < 10%, additional RAM < 128 MB

AC2:
  Given: MQTT is running
  When: SNMP/Modbus polling accuracy measured
  Then: Polling interval deviation < 5% (compared to no-MQTT baseline)

AC3:
  Given: MQTT broker disconnect/reconnect storm
  When: Rapid reconnections occur
  Then: Other protocols and Web UI remain unaffected

**Tasks**
- [ ] Task 1: Define performance test plan and RDGC target hardware configuration
- [ ] Task 2: Establish no-MQTT baseline performance data
- [ ] Task 3: Execute maximum MQTT load test
- [ ] Task 4: Execute mixed-protocol coexistence test
- [ ] Task 5: Execute reconnection storm test
- [ ] Task 6: Produce performance test report; update Sizing Guide

**Definition of Ready checklist**
- [ ] RDGC target hardware environment provisioned
- [ ] Performance test tools prepared
- [ ] Baseline data collected
- [ ] Estimated by team

---

### STORY: S12 — End-to-End Integration Test and Probe Server Feature Parity Verification

**Parent Epic**: BDCSPM-70937-RDGC
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As a QA Engineer,
I want to verify RDGC MQTT functionality matches Probe Server's MQTT capability,
So that we confirm feature parity and customer experience consistency.

**Context & notes**:
- Compare Probe Server and RDGC MQTT feature matrix
- End-to-end scenarios: device configuration → MQTT connection → data collection → Graph/Alarm → disconnect recovery
- Cover all Jira-defined Test Cases (TC-MQTT-01 through TC-MQTT-07)
- Produce Feature Parity comparison report

**Acceptance Criteria**

AC1:
  Given: Probe Server and RDGC configured with the same MQTT device
  When: Feature behavior is compared
  Then: All feature items behave identically (Feature Parity achieved)

AC2:
  Given: RDGC MQTT end-to-end tests
  When: TC-MQTT-01 through TC-MQTT-07 are executed
  Then: All test scenarios PASS

AC3:
  Given: Feature Parity comparison identifies differences
  When: Gaps are recorded
  Then: Gap Analysis report produced, marking each as "design difference" or "implementation defect"

**Tasks**
- [ ] Task 1: Write RDGC MQTT Feature Parity comparison matrix
- [ ] Task 2: Execute end-to-end functional tests (full device lifecycle)
- [ ] Task 3: Execute Jira-defined TC-MQTT-01 through TC-MQTT-07
- [ ] Task 4: Compare with Probe Server behavior for differences
- [ ] Task 5: Produce test report and Gap Analysis

**Definition of Ready checklist**
- [ ] S1 through S11 all complete
- [ ] Probe Server feature baseline documented
- [ ] Test environment (RDGC + Probe Server parallel) ready
- [ ] Estimated by team

---

## Validation Checklist

- [x] All AC are testable
- [x] NFRs are quantified (TLS 1.2+, 30s reconnect, 50 devices, 200 topics, <10% CPU, <128MB RAM)
- [x] BLSS product explicitly identified (Distributed IT Performance Management — RDGC)
- [x] Scope boundaries are explicit
- [x] Cross-product dependencies identified (Probe Server shared module)
- [x] On-prem operational impact assessed (new module, port 8883, config backup)
- [x] Dependencies identified (BDCSPM-6541, Probe Server code modularity)
- [x] No invented features — based on existing Probe Server implementation
- [x] Terminology matches existing UI (Monitor Config, Monitoring Templates, Verify, Submit)
- [x] Placeholders marked with [TBD] where applicable
- [x] Operational deliverables included
- [x] Leverages existing Probe Server implementation to reduce risk and effort
- [x] Polling Groups removed — MQTT is event-driven; timeout detection handled via Expected Update Interval (S8)

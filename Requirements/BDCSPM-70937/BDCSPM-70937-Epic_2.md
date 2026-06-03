# BDCSPM-70937 — Epic 2: MQTT Data Ingestion & Device Model Integration

## Capability Reference

| Field | Value |
|---|---|
| **Capability** | BDCSPM-70937: RDGC has the ability to monitor devices using MQTT |
| **Parent** | BDCSPM-6550: RDGC monitoring protocol parity with BLSS |
| **Fix Version** | BLSS 8.2.0 (2027-01-15) |
| **Priority** | High |
| **Dependency** | BDCSPM-6541: RDGC device multi-protocol auto discovery |

---

### EPIC: BDCSPM-70937-E2 — MQTT Data Ingestion & Device Model Integration

**BLSS Product(s)**: Distributed IT Performance Management (RDGC component)

**Summary**
Implement the data ingestion pipeline that processes MQTT messages received from subscribed topics, maps telemetry payloads to the RDGC device model, and makes monitoring parameters available for display and alarming within the Brightlayer On-Premise platform.

**Description**

- **Problem statement**: Raw MQTT messages from heterogeneous devices arrive in varied payload formats (JSON, binary, key-value). RDGC needs a configurable data ingestion layer that can parse these messages, map them to a normalized device model, and feed them into the existing monitoring and alarming infrastructure.
- **Proposed solution**: Implement a message processing pipeline with configurable payload parsers, a topic-to-device mapping engine, and integration with the existing RDGC data store and parameter model.
- **User personas**: IT/OT Administrators (configure mappings), Operations Teams (view monitoring data), System Integrators (set up device profiles)
- **Business value**: Enables seamless integration of MQTT device telemetry into existing monitoring workflows without requiring changes to the rest of the Brightlayer platform.
- **Key workflows**:
  1. MQTT message received on subscribed topic
  2. Payload parser extracts data points based on configured format
  3. Topic-to-device mapping identifies the target device and parameters
  4. Normalized data stored in RDGC data model
  5. Parameters available for display, trending, and alarming
- **Cross-product touchpoints**: Data flows into existing RDGC monitoring database, which is consumed by the Brightlayer platform dashboards and alarm engine.
- **UX/UI references**: Device profile configuration in Epic 3.
- **On-prem operational impact**:
  - Install/upgrade impact: New message processing module; DB schema extension for MQTT device metadata and topic mappings
  - Sizing impact: Additional disk I/O for telemetry storage; dependent on message frequency and device count
  - Network impact: No additional external ports (internal data flow only)
  - Backup impact: MQTT device mappings and ingested telemetry included in standard DB backup

**Non-Functional Requirements (NFRs)**

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Performance | Message processing throughput | ≥ 1000 messages/second per RDGC instance on reference hardware | P0 |
| Performance | End-to-end ingestion latency (message receipt → available in data model) | < 5 seconds at P95 | P0 |
| Reliability | Message loss rate under normal operation | < 0.1% (with QoS 1) | P0 |
| Reliability | Malformed payload handling | No crash; log error, increment counter, skip message | P0 |
| Scalability | MQTT devices per RDGC instance | ≥ 200 MQTT-sourced devices | P1 |
| Scalability | Parameters per device | ≥ 50 monitoring parameters per MQTT device | P1 |
| Data Integrity | Duplicate message handling | Deduplicate by message ID + timestamp within 60s window | P1 |
| Observability | Ingestion metrics | Messages processed/sec, parse errors/sec, dropped messages/sec | P1 |
| Storage | Data retention | Aligned with existing RDGC data retention policies | P2 |

**Acceptance Criteria (Epic-level)**

- [ ] AC1: RDGC successfully ingests telemetry data published to subscribed MQTT topics
- [ ] AC2: Ingested data is visible in the Brightlayer monitoring interface as device parameters
- [ ] AC3: Default monitoring parameters and update frequency are displayed per MQTT device
- [ ] AC4: Malformed or unexpected payloads are handled gracefully without affecting other devices
- [ ] AC5: MQTT-sourced device data integrates with existing alarming infrastructure

**Scope**

- **In scope**:
  - MQTT message reception from the protocol engine (Epic 1)
  - Payload parsing (JSON as primary; extensible for other formats)
  - Topic-to-device/parameter mapping configuration
  - Device model integration (MQTT devices appear alongside SNMP/Modbus devices)
  - Monitoring parameter normalization and storage
  - Update frequency / polling interval display
  - Message deduplication
  - Error handling for malformed payloads
  - Ingestion metrics and logging

- **Out of scope**:
  - Advanced analytics or ML processing of MQTT data
  - Custom payload format plugin system (future roadmap)
  - Real-time streaming to external systems
  - Sub-second data resolution
  - Historical data migration from non-MQTT sources

- **Assumptions**:
  1. ⚠️ ASSUMPTION: JSON is the primary payload format; other formats (Protobuf, MessagePack, CSV) can be added in future iterations
  2. ⚠️ ASSUMPTION: Topic structure follows a convention that allows device identification (e.g., `site/building/device-id/telemetry`)
  3. ⚠️ ASSUMPTION: MQTT devices will reuse the existing RDGC device model and data store schema with minimal extension
  4. ⚠️ ASSUMPTION: Message deduplication window of 60 seconds is sufficient for all expected scenarios
  5. ⚠️ ASSUMPTION: Existing alarming engine can consume MQTT-sourced parameters without modification — [TBD — confirm with alarming team]

**Dependencies & Risks**

| Item | Type | BLSS Product | Owner | Mitigation | Status |
|---|---|---|---|---|---|
| Epic 1 (MQTT Protocol Engine) | Dep | Distributed IT | Dev team | Must complete E1-S1 through E1-S4 before ingestion work | Open |
| RDGC device model schema | Dep | Distributed IT | Architecture | Confirm extensibility for MQTT device type | Open |
| Alarming engine compatibility | Dep | Distributed IT | Alarming team | Validate MQTT parameters trigger alarms correctly | Open |
| Inconsistent topic schemas across vendors | Risk | N/A | Dev team | Configurable parser with validation rules; customer documentation | Open |
| High message volume causing backpressure | Risk | Distributed IT | Dev team | Implement rate limiting and message queue with overflow handling | Open |
| Payload format diversity | Risk | N/A | Dev team | Start with JSON; document extension points for future formats | Open |

**Operational Deliverables** (required for on-prem)

- [ ] Installation/upgrade runbook updated (DB schema migration for MQTT device metadata)
- [ ] Sizing guide updated (disk IOPS based on message rate; storage projections)
- [ ] Backup/restore procedure updated (MQTT mapping configuration and telemetry data)
- [ ] Release notes entry drafted
- [ ] Lifecycle Services training/enablement updated (MQTT device profile configuration)
- [ ] Customer documentation: MQTT topic naming conventions and payload format guide

---

## Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| E2-S1 | MQTT Message Reception & Routing | 5 | Sprint 2 |
| E2-S2 | JSON Payload Parser | 5 | Sprint 2 |
| E2-S3 | Topic-to-Device Mapping Engine | 8 | Sprint 3 |
| E2-S4 | Device Model Integration for MQTT Sources | 8 | Sprint 3 |
| E2-S5 | Monitoring Parameter Normalization & Storage | 5 | Sprint 4 |
| E2-S6 | Message Deduplication & Error Handling | 5 | Sprint 4 |
| E2-S7 | Ingestion Metrics & Observability | 3 | Sprint 5 |

---

## Story Details

---

### STORY: E2-S1 — MQTT Message Reception & Routing

**Parent Epic**: BDCSPM-70937-E2 — MQTT Data Ingestion & Device Model Integration
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As a System Integrator,
I want RDGC to receive MQTT messages from subscribed topics and route them to the appropriate processing pipeline,
So that telemetry data from different devices is correctly handled.

**Context & notes**:
- Consumes messages from the MQTT Protocol Engine (Epic 1, internal interface)
- Routes messages based on topic structure to the appropriate parser and device mapper
- Must handle burst traffic without dropping messages (internal message queue)
- Configurable queue depth with overflow handling (log warning + drop oldest)

**Acceptance Criteria**

AC1:
  Given: RDGC is connected and subscribed to MQTT topics
  When: A message is published to a subscribed topic
  Then: The message is received and placed in the internal processing queue within 1 second

AC2:
  Given: Messages arrive from multiple brokers/topics simultaneously
  When: Burst traffic of 500 messages/second occurs
  Then: All messages are queued for processing without loss (up to configured queue depth)

AC3:
  Given: The internal processing queue reaches maximum depth
  When: Additional messages arrive
  Then: A warning is logged, oldest messages are dropped, and the system remains stable

**Tasks**
- [ ] Task 1: Implement internal message queue between protocol engine and processing pipeline
- [ ] Task 2: Implement message routing based on topic pattern matching
- [ ] Task 3: Add queue depth configuration and overflow handling
- [ ] Task 4: Implement metrics for queue depth, messages received/sec
- [ ] Task 5: Write unit tests for routing logic
- [ ] Task 6: Write integration tests for burst traffic handling

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Internal API contract with Protocol Engine (Epic 1) agreed
- [ ] Queue depth defaults validated with architecture team
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E2-S2 — JSON Payload Parser

**Parent Epic**: BDCSPM-70937-E2 — MQTT Data Ingestion & Device Model Integration
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want RDGC to parse JSON-formatted MQTT payloads and extract monitoring data points,
So that device telemetry can be understood and stored by the platform.

**Context & notes**:
- JSON is the primary supported payload format
- Configurable JSONPath expressions to extract values from nested payloads
- Support common telemetry patterns: flat key-value, nested objects, arrays of readings
- Data type inference: numeric, string, boolean, timestamp
- Must handle schema variations gracefully (missing fields, extra fields)

**Acceptance Criteria**

AC1:
  Given: An MQTT message with a JSON payload like `{"temperature": 23.5, "humidity": 45, "timestamp": "2026-01-15T10:30:00Z"}`
  When: The parser processes the message
  Then: Individual data points are extracted with correct values and data types

AC2:
  Given: A configurable JSONPath mapping (e.g., `$.sensors[*].value`)
  When: A nested JSON payload arrives matching the schema
  Then: All matching values are extracted according to the mapping rules

AC3:
  Given: A malformed JSON payload (syntax error or unexpected structure)
  When: The parser attempts to process it
  Then: A parse error is logged with message details, the message is skipped, and processing continues

AC4:
  Given: A JSON payload with missing expected fields
  When: The parser processes the message
  Then: Available fields are extracted; missing fields are reported as null with a warning log

**Tasks**
- [ ] Task 1: Implement JSON payload parser with configurable field extraction
- [ ] Task 2: Implement JSONPath expression evaluator for nested payloads
- [ ] Task 3: Implement data type inference and conversion
- [ ] Task 4: Implement error handling for malformed payloads (log + skip)
- [ ] Task 5: Add parser configuration schema (field mappings, JSONPath rules, data types)
- [ ] Task 6: Write unit tests with various JSON payload structures
- [ ] Task 7: Write tests for malformed payload handling

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Sample JSON payloads from target MQTT devices collected
- [ ] JSONPath library selection approved
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E2-S3 — Topic-to-Device Mapping Engine

**Parent Epic**: BDCSPM-70937-E2 — MQTT Data Ingestion & Device Model Integration
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want to configure how MQTT topics map to devices and monitoring parameters,
So that RDGC knows which device each MQTT message belongs to and what data it contains.

**Context & notes**:
- Topic structure often encodes device identity (e.g., `site/floor1/ups-001/telemetry`)
- Mapping rules define: topic pattern → device ID extraction → parameter mapping
- Support regex-based topic parsing for device identification
- Must support multiple mapping profiles for different device types/vendors
- Configuration stored persistently in RDGC configuration database

**Acceptance Criteria**

AC1:
  Given: A topic mapping rule `site/+/+/telemetry` with device ID extracted from topic segment 3
  When: A message arrives on `site/floor1/ups-001/telemetry`
  Then: The message is associated with device "ups-001"

AC2:
  Given: Multiple mapping profiles for different device vendors
  When: Messages arrive from different topic structures
  Then: Each message is correctly mapped to its device based on the matching profile

AC3:
  Given: A message arrives on an unmapped topic
  When: No mapping rule matches the topic
  Then: The message is logged as "unmapped" and a metric counter is incremented

AC4:
  Given: A mapping configuration is updated
  When: New messages arrive
  Then: The updated mapping rules are applied without requiring RDGC restart

**Tasks**
- [ ] Task 1: Design topic-to-device mapping configuration schema
- [ ] Task 2: Implement topic pattern matching engine (wildcard and regex support)
- [ ] Task 3: Implement device ID extraction from topic segments
- [ ] Task 4: Implement parameter mapping from payload fields to device model parameters
- [ ] Task 5: Implement mapping profile management (CRUD operations)
- [ ] Task 6: Implement hot-reload of mapping configuration
- [ ] Task 7: Write unit tests for topic pattern matching
- [ ] Task 8: Write integration tests for end-to-end message-to-device mapping

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Topic naming conventions from target customer environments documented
- [ ] Mapping configuration schema reviewed by architecture team
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E2-S4 — Device Model Integration for MQTT Sources

**Parent Epic**: BDCSPM-70937-E2 — MQTT Data Ingestion & Device Model Integration
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an Operations Team member,
I want MQTT-monitored devices to appear in the same device inventory as SNMP and Modbus devices,
So that I have a unified view of all monitored infrastructure regardless of protocol.

**Context & notes**:
- MQTT devices must integrate with existing RDGC device model (device type, name, location, parameters)
- Device type: "MQTT" or vendor-specific type derived from device profile
- Device provisioning: auto-create on first message (with configured profile) or manual pre-provision
- Protocol indicator in device model to distinguish MQTT from SNMP/Modbus
- Existing device list, search, and filtering must include MQTT devices

**Acceptance Criteria**

AC1:
  Given: A device profile is configured for MQTT device type "UPS-MQTT"
  When: The first telemetry message is received from a new MQTT device
  Then: The device is automatically created in the device inventory with type "UPS-MQTT"

AC2:
  Given: MQTT devices exist in the device inventory
  When: An administrator views the device list
  Then: MQTT devices appear alongside SNMP/Modbus devices with a protocol indicator

AC3:
  Given: An MQTT device is registered in the inventory
  When: Its monitoring parameters are queried
  Then: All mapped parameters from MQTT messages are available with current values and timestamps

AC4:
  Given: An MQTT device has not reported data within the configured timeout
  When: The stale-data threshold is exceeded
  Then: The device status changes to "Communication Lost"

**Tasks**
- [ ] Task 1: Extend device model schema to support MQTT device type and protocol indicator
- [ ] Task 2: Implement auto-provisioning logic (first message → device creation)
- [ ] Task 3: Implement manual device provisioning via configuration
- [ ] Task 4: Integrate MQTT devices into existing device list/search/filter APIs
- [ ] Task 5: Implement stale-data detection and "Communication Lost" status
- [ ] Task 6: Write DB migration script for device model extension
- [ ] Task 7: Write unit tests for device provisioning logic
- [ ] Task 8: Write integration tests for device lifecycle (create, update, stale)

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Device model schema extension reviewed by architecture team
- [ ] Auto-provisioning vs. manual provisioning behavior agreed
- [ ] Stale-data timeout default value agreed (e.g., 5 minutes)
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E2-S5 — Monitoring Parameter Normalization & Storage

**Parent Epic**: BDCSPM-70937-E2 — MQTT Data Ingestion & Device Model Integration
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an Operations Team member,
I want MQTT telemetry values to be stored as standard monitoring parameters with proper units and timestamps,
So that I can view trends, set thresholds, and receive alarms on MQTT device data.

**Context & notes**:
- Map parsed values to RDGC parameter model (name, value, unit, timestamp, quality)
- Support engineering unit configuration per parameter
- Update frequency / polling interval equivalent displayed (based on message arrival rate)
- Store historical values aligned with existing data retention policies
- Integration with existing trending and alarming infrastructure

**Acceptance Criteria**

AC1:
  Given: An MQTT message contains a temperature reading of 23.5
  When: The data is stored in the parameter model
  Then: It is accessible as parameter "temperature" with value 23.5, unit "°C", and the message timestamp

AC2:
  Given: MQTT messages arrive at regular intervals for a device
  When: The update frequency is calculated
  Then: The effective polling/update frequency is displayed (e.g., "updates every 60s")

AC3:
  Given: Stored MQTT parameters
  When: An alarm threshold is configured for a parameter
  Then: Alarms trigger correctly when the parameter value crosses the threshold

AC4:
  Given: Data retention policy of 90 days
  When: MQTT parameter data exceeds the retention period
  Then: Data is purged according to the standard RDGC retention policy

**Tasks**
- [ ] Task 1: Implement parameter normalization (value + unit + timestamp + quality)
- [ ] Task 2: Implement engineering unit configuration per parameter
- [ ] Task 3: Implement update frequency calculation from message arrival rate
- [ ] Task 4: Integrate with existing RDGC data store for parameter persistence
- [ ] Task 5: Validate alarm engine compatibility with MQTT-sourced parameters
- [ ] Task 6: Validate data retention policy applies to MQTT data
- [ ] Task 7: Write unit tests for normalization logic
- [ ] Task 8: Write integration tests for trending and alarm integration

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] RDGC parameter model documentation reviewed
- [ ] Alarm engine team confirms no changes needed — [TBD — confirm]
- [ ] Data retention policy confirmed for MQTT data
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E2-S6 — Message Deduplication & Error Handling

**Parent Epic**: BDCSPM-70937-E2 — MQTT Data Ingestion & Device Model Integration
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want RDGC to handle duplicate MQTT messages and processing errors gracefully,
So that data integrity is maintained and one faulty device doesn't affect others.

**Context & notes**:
- QoS 1 may deliver duplicates; deduplicate by message ID + timestamp within 60s window
- Error isolation: one device's malformed data must not affect other devices' processing
- Rate limiting per topic/device to prevent one device from overwhelming the pipeline
- Dead letter queue concept: store failed messages for admin review

**Acceptance Criteria**

AC1:
  Given: A message is received twice (QoS 1 redelivery) within 60 seconds
  When: The deduplication engine processes both
  Then: Only one copy is stored; the duplicate is discarded with a debug log

AC2:
  Given: A device sends malformed data that causes a parse error
  When: The error occurs during processing
  Then: The error is logged, the message is skipped, and other devices' messages continue processing normally

AC3:
  Given: A device sends messages at an extremely high rate (> configured threshold)
  When: Rate limiting is triggered
  Then: Excess messages are dropped, a warning is logged, and a metric indicates throttled messages

**Tasks**
- [ ] Task 1: Implement message deduplication (sliding window, message ID + timestamp)
- [ ] Task 2: Implement error isolation per device/topic (independent processing pipelines)
- [ ] Task 3: Implement rate limiting per device/topic (configurable threshold)
- [ ] Task 4: Implement failed message logging for troubleshooting
- [ ] Task 5: Add error and deduplication metrics
- [ ] Task 6: Write unit tests for deduplication logic
- [ ] Task 7: Write integration tests for error isolation and rate limiting

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Deduplication window and rate limiting defaults agreed
- [ ] Error handling strategy reviewed by architecture team
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E2-S7 — Ingestion Metrics & Observability

**Parent Epic**: BDCSPM-70937-E2 — MQTT Data Ingestion & Device Model Integration
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want visibility into MQTT data ingestion health and throughput,
So that I can identify and troubleshoot issues with device data flow.

**Context & notes**:
- Metrics: messages received/sec, messages processed/sec, parse errors/sec, duplicates/sec, dropped messages/sec
- Per-broker and per-device granularity available
- Metrics exposed via internal API (consumed by UI in Epic 3)
- Log levels configurable for MQTT subsystem independently

**Acceptance Criteria**

AC1:
  Given: MQTT data ingestion is active
  When: The ingestion metrics API is queried
  Then: Current throughput (messages/sec), error rate, and queue depth are returned

AC2:
  Given: Parse errors are occurring for a specific device
  When: The per-device metrics are queried
  Then: The error count and last error details are available for that device

AC3:
  Given: An administrator changes the MQTT subsystem log level to DEBUG
  When: Messages are processed
  Then: Detailed processing logs (payload content, mapping decisions) are emitted

**Tasks**
- [ ] Task 1: Implement ingestion metrics collection (counters, gauges)
- [ ] Task 2: Implement per-broker and per-device metric aggregation
- [ ] Task 3: Expose metrics via internal API endpoint
- [ ] Task 4: Implement configurable log level for MQTT subsystem
- [ ] Task 5: Write unit tests for metric collection accuracy
- [ ] Task 6: Write integration tests for metrics API

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Metrics API contract agreed with UI team
- [ ] Log level configuration mechanism agreed
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

## Validation Checklist

- [x] All AC are testable
- [x] NFRs are quantified (1000 msg/sec, <5s latency, <0.1% loss, ≥200 devices, ≥50 params)
- [x] BLSS product explicitly identified (Distributed IT Performance Management — RDGC)
- [x] Scope boundaries are explicit
- [x] Cross-product dependencies identified (alarming engine compatibility)
- [x] On-prem operational impact assessed (DB schema migration, disk sizing, retention)
- [x] Dependencies identified (Epic 1, device model, alarming engine)
- [x] No invented features, UI labels, device models, or protocol details
- [x] Terminology matches BLSS product glossary
- [x] Placeholders marked with [TBD] where applicable
- [x] Operational deliverables included

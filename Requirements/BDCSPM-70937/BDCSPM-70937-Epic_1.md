# BDCSPM-70937 — Epic 1: MQTT Protocol Engine & Secure Device Connectivity

## Capability Reference

| Field | Value |
|---|---|
| **Capability** | BDCSPM-70937: RDGC has the ability to monitor devices using MQTT |
| **Parent** | BDCSPM-6550: RDGC monitoring protocol parity with BLSS |
| **Fix Version** | BLSS 8.2.0 (2027-01-15) |
| **Priority** | High |
| **Dependency** | BDCSPM-6541: RDGC device multi-protocol auto discovery |
| **Labels** | Customer-SustainingEngineering, DIT, RDG |

---

### EPIC: BDCSPM-70937-E1 — MQTT Protocol Engine & Secure Device Connectivity

**BLSS Product(s)**: Distributed IT Performance Management (RDGC component)

**Summary**
Implement a secure MQTT client engine within RDGC that enables the gateway to establish TLS-encrypted connections to customer-managed MQTT brokers or device-hosted brokers, subscribe to configured topics, and maintain reliable connectivity with automatic reconnect capabilities.

**Description**

- **Problem statement**: RDGC currently only supports polling-based protocols (SNMP, Modbus, BACnet) for device monitoring. Customers with MQTT-enabled edge devices or IoT systems cannot integrate these devices into the Brightlayer On-Premise monitoring platform without deploying separate tools.
- **Proposed solution**: Add an MQTT client module to RDGC that establishes secure connections to MQTT brokers, subscribes to configurable topics, and provides a well-defined internal interface for downstream data ingestion services.
- **User personas**: IT/OT Administrators, System Integrators
- **Business value**: Expands device compatibility without requiring protocol-specific integrations; enables higher device density per installation, driving increased license revenue.
- **Key workflows**:
  1. Administrator configures MQTT broker endpoint (host, port, credentials, TLS settings)
  2. RDGC establishes secure MQTT connection
  3. RDGC subscribes to configured topics
  4. Connection status is reported internally for UI consumption
  5. On disconnect, RDGC auto-reconnects with configurable backoff
- **Cross-product touchpoints**: Shares device inventory model with Brightlayer Power (future). No direct cross-product dependency in this Epic.
- **UX/UI references**: Configuration UI covered in Epic 3. This Epic is backend/engine only.
- **On-prem operational impact**:
  - Install/upgrade impact: New MQTT client service/module within RDGC; new configuration file or section
  - Sizing impact: Additional RAM/CPU for MQTT client threads (see NFR table)
  - Network impact: New outbound port(s) — default MQTT 8883 (TLS); customer firewall rules documentation required
  - Backup impact: MQTT configuration data included in standard RDGC backup

**Non-Functional Requirements (NFRs)**

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Security | MQTT connections must use encrypted transport | TLS 1.2+ mandatory; mTLS optional | P0 |
| Security | Credential storage | Credentials encrypted at rest using RDGC key management | P0 |
| Reliability | Auto-reconnect on broker disconnect | Reconnect within 30s with exponential backoff (max 5 min) | P0 |
| Reliability | Connection state tracking | Connection state changes logged and surfaced via internal API within 5s | P1 |
| Scalability | Concurrent MQTT broker connections per RDGC | ≥ 10 broker connections on reference hardware | P1 |
| Scalability | Topic subscriptions per connection | ≥ 50 topics per broker connection | P1 |
| Performance | MQTT engine CPU overhead | < 5% additional CPU at steady state on reference hardware | P1 |
| Performance | Memory footprint | < 256 MB additional RAM for MQTT engine | P1 |
| Compatibility | MQTT protocol version | MQTT v3.1.1 and v5.0 | P1 |
| Observability | Connection metrics | Connection uptime, reconnect count, last-seen timestamp available via internal metrics | P2 |

**Acceptance Criteria (Epic-level)**

- [ ] AC1: RDGC can establish a TLS-encrypted MQTT connection to an external broker using configured credentials
- [ ] AC2: RDGC successfully subscribes to one or more configured MQTT topics
- [ ] AC3: RDGC automatically reconnects after broker disconnect within 30 seconds (first attempt)
- [ ] AC4: Invalid or expired credentials prevent connection establishment and generate a clear error log
- [ ] AC5: MQTT engine operates without degrading existing RDGC protocol monitoring (SNMP, Modbus, etc.)
- [ ] AC6: Connection state (connected/disconnected/error) is available via internal API for UI consumption

**Scope**

- **In scope**:
  - MQTT client library integration (MQTT v3.1.1 and v5.0)
  - TLS 1.2+ encrypted transport (server certificate validation)
  - Optional mutual TLS (client certificate)
  - Username/password authentication
  - Configurable broker endpoint (host, port, client ID, keep-alive)
  - Topic subscription management (subscribe/unsubscribe)
  - Auto-reconnect with exponential backoff
  - Internal API exposing connection state and health metrics
  - Credential encryption at rest
  - Coexistence with existing protocol engines

- **Out of scope**:
  - MQTT message publishing (device control)
  - MQTT broker hosting or lifecycle management
  - Certificate management UI (uses existing RDGC cert store)
  - QoS 2 "exactly once" delivery guarantee (QoS 0 and 1 supported)
  - MQTT over WebSocket

- **Assumptions**:
  1. ⚠️ ASSUMPTION: Customer MQTT brokers support standard MQTT v3.1.1 or v5.0 — proprietary extensions not supported
  2. ⚠️ ASSUMPTION: RDGC reference hardware has sufficient capacity for MQTT engine alongside existing workloads — validate with performance testing
  3. ⚠️ ASSUMPTION: Existing RDGC key management infrastructure can be reused for MQTT credential encryption
  4. ⚠️ ASSUMPTION: Network latency to broker < 500ms for reliable keep-alive operation

**Dependencies & Risks**

| Item | Type | BLSS Product | Owner | Mitigation | Status |
|---|---|---|---|---|---|
| BDCSPM-6541: Multi-protocol auto discovery | Dep | Distributed IT | TBD | Coordinate discovery mechanism for MQTT devices | Ready for Development |
| MQTT client library selection | Risk | Distributed IT | Dev team | Evaluate Eclipse Paho / HiveMQ client; POC performance | Open |
| Customer broker compatibility variance | Risk | N/A | QA | Test against Mosquitto, HiveMQ, EMQX, AWS IoT Core (bridged) | Open |
| TLS certificate chain validation issues | Risk | Distributed IT | Dev team | Support custom CA cert import; clear error messages | Open |
| High-frequency telemetry overwhelming RDGC | Risk | Distributed IT | Architecture | Implement message rate limiting; document max throughput | Open |

**Operational Deliverables** (required for on-prem)

- [ ] Installation/upgrade runbook updated (new MQTT module deployment steps)
- [ ] Sizing guide updated (MQTT engine resource requirements)
- [ ] Backup/restore procedure updated (MQTT configuration data)
- [ ] Firewall/port documentation updated (port 8883/TLS, configurable)
- [ ] Release notes entry drafted
- [ ] Lifecycle Services training/enablement updated

---

## Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| E1-S1 | MQTT Client Library Integration & Base Connection | 8 | Sprint 1 |
| E1-S2 | TLS Encryption & Certificate Validation | 5 | Sprint 1 |
| E1-S3 | Authentication (Username/Password & mTLS) | 5 | Sprint 2 |
| E1-S4 | Topic Subscription Management | 5 | Sprint 2 |
| E1-S5 | Auto-Reconnect & Connection Resilience | 8 | Sprint 3 |
| E1-S6 | Internal Connection State API | 5 | Sprint 3 |
| E1-S7 | Coexistence & Performance Validation | 5 | Sprint 4 |

---

## Story Details

---

### STORY: E1-S1 — MQTT Client Library Integration & Base Connection

**Parent Epic**: BDCSPM-70937-E1 — MQTT Protocol Engine & Secure Device Connectivity
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want RDGC to establish a basic MQTT connection to a configured broker,
So that the platform can communicate with MQTT-enabled devices and infrastructure.

**Context & notes**:
- Evaluate and integrate MQTT client library (e.g., Eclipse Paho, HiveMQ client)
- Support MQTT v3.1.1 and v5.0 protocol versions
- Connection parameters: host, port, client ID, keep-alive interval, clean session flag
- This story covers unencrypted connection for initial validation; TLS in S2
- On-prem consideration: Library must be distributable without external download at runtime

**Acceptance Criteria**

AC1:
  Given: A valid MQTT broker endpoint is configured in RDGC settings
  When: RDGC MQTT service starts
  Then: A connection is established to the broker and CONNACK is received within 10 seconds

AC2:
  Given: An invalid broker host/port is configured
  When: RDGC MQTT service attempts to connect
  Then: A clear error is logged with the connection failure reason and no crash occurs

AC3:
  Given: A connected MQTT session
  When: The broker sends a PING response
  Then: The keep-alive mechanism maintains the connection without timeout

**Tasks**
- [ ] Task 1: Evaluate MQTT client libraries (Paho C/C++ vs HiveMQ) — produce decision doc
- [ ] Task 2: Integrate selected library into RDGC build system
- [ ] Task 3: Implement MQTT connection manager with configurable parameters
- [ ] Task 4: Implement connection lifecycle (connect, disconnect, destroy)
- [ ] Task 5: Add MQTT configuration schema (host, port, clientId, keepAlive, protocolVersion)
- [ ] Task 6: Write unit tests for connection manager
- [ ] Task 7: Write integration test with Mosquitto test broker

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] MQTT library evaluation complete
- [ ] RDGC architecture team approves integration approach
- [ ] Test broker environment available
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E1-S2 — TLS Encryption & Certificate Validation

**Parent Epic**: BDCSPM-70937-E1 — MQTT Protocol Engine & Secure Device Connectivity
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want MQTT connections from RDGC to use TLS encryption,
So that device telemetry data is transmitted securely and is protected from eavesdropping.

**Context & notes**:
- TLS 1.2 minimum; TLS 1.3 preferred where broker supports it
- Server certificate validation against system trust store + custom CA certs
- Support custom CA certificate import (using existing RDGC certificate store)
- Reject connections with invalid/expired/self-signed certs unless explicitly configured
- Port 8883 (MQTTS) as default

**Acceptance Criteria**

AC1:
  Given: MQTT broker is configured with TLS enabled and a valid server certificate
  When: RDGC connects to the broker
  Then: Connection is established over TLS 1.2+ and data is encrypted in transit

AC2:
  Given: MQTT broker presents an expired or invalid certificate
  When: RDGC attempts to connect
  Then: Connection is rejected and a security warning is logged with certificate details

AC3:
  Given: A custom CA certificate has been imported to the RDGC certificate store
  When: RDGC connects to a broker whose cert is signed by that CA
  Then: The connection succeeds with proper certificate chain validation

**Tasks**
- [ ] Task 1: Implement TLS wrapper for MQTT client connections
- [ ] Task 2: Integrate with RDGC certificate store for CA cert retrieval
- [ ] Task 3: Implement server certificate validation logic
- [ ] Task 4: Add TLS configuration parameters (minVersion, caCertPath, verifyHostname)
- [ ] Task 5: Write integration tests (valid cert, expired cert, self-signed cert, custom CA)
- [ ] Task 6: Update firewall documentation with port 8883 requirement

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] RDGC certificate store API documented
- [ ] Test certificates (valid, expired, custom CA) prepared
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E1-S3 — Authentication (Username/Password & mTLS)

**Parent Epic**: BDCSPM-70937-E1 — MQTT Protocol Engine & Secure Device Connectivity
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want RDGC to authenticate to the MQTT broker using credentials or client certificates,
So that only authorized clients can subscribe to device telemetry topics.

**Context & notes**:
- Support username/password authentication (MQTT CONNECT packet)
- Support mutual TLS (client certificate) as an alternative or additional auth mechanism
- Credentials must be encrypted at rest in RDGC configuration
- Use existing RDGC key management for encryption
- Failed auth must produce clear, actionable error logs

**Acceptance Criteria**

AC1:
  Given: Valid username/password credentials are configured for an MQTT broker
  When: RDGC connects to the broker
  Then: Authentication succeeds and subscription is allowed

AC2:
  Given: Invalid credentials are configured
  When: RDGC attempts to connect
  Then: Connection is rejected with CONNACK reason code logged; no retry with same credentials

AC3:
  Given: Mutual TLS is configured with a valid client certificate
  When: RDGC connects to a broker requiring client cert auth
  Then: mTLS handshake completes and connection is authenticated

AC4:
  Given: Credentials are stored in RDGC configuration
  When: An administrator inspects the configuration file
  Then: Credential values are encrypted and not visible in plaintext

**Tasks**
- [ ] Task 1: Implement username/password auth in MQTT CONNECT packet
- [ ] Task 2: Implement credential encryption at rest using RDGC key management
- [ ] Task 3: Implement mTLS client certificate configuration
- [ ] Task 4: Add auth-related configuration parameters (authType, username, encryptedPassword, clientCertPath, clientKeyPath)
- [ ] Task 5: Implement auth failure handling (log, no-retry with same creds)
- [ ] Task 6: Write unit tests for credential encryption/decryption
- [ ] Task 7: Write integration tests for auth success/failure scenarios

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] RDGC key management API documented
- [ ] Test broker with auth enforcement configured
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E1-S4 — Topic Subscription Management

**Parent Epic**: BDCSPM-70937-E1 — MQTT Protocol Engine & Secure Device Connectivity
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an IT/OT Administrator,
I want to configure which MQTT topics RDGC subscribes to,
So that only relevant device telemetry is ingested by the monitoring platform.

**Context & notes**:
- Support wildcard subscriptions (+ single-level, # multi-level)
- Configurable QoS level per subscription (QoS 0 and QoS 1)
- Dynamic subscribe/unsubscribe without full reconnection
- Topic configuration stored as part of MQTT device/broker profile

**Acceptance Criteria**

AC1:
  Given: RDGC is connected to an MQTT broker
  When: A topic subscription is configured (e.g., "site1/devices/+/telemetry")
  Then: RDGC subscribes to the topic and begins receiving matching messages

AC2:
  Given: RDGC is subscribed to multiple topics
  When: A topic subscription is removed from configuration
  Then: RDGC unsubscribes from the topic and stops receiving messages for it

AC3:
  Given: A wildcard topic subscription is configured
  When: Messages arrive on matching topics
  Then: All matching messages are received and routed to the data ingestion layer

AC4:
  Given: A subscription request fails (e.g., ACL denied)
  When: The broker returns SUBACK with failure code
  Then: The failure is logged with the specific topic and reason

**Tasks**
- [ ] Task 1: Implement topic subscription manager (add/remove/list subscriptions)
- [ ] Task 2: Support MQTT wildcard patterns (+ and #)
- [ ] Task 3: Implement QoS level configuration per subscription
- [ ] Task 4: Implement dynamic subscribe/unsubscribe without disconnect
- [ ] Task 5: Implement SUBACK handling and failure logging
- [ ] Task 6: Add topic configuration schema and persistence
- [ ] Task 7: Write unit tests for topic manager
- [ ] Task 8: Write integration tests with wildcard subscriptions

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Topic naming convention guidance documented for customer configuration
- [ ] Test broker with ACL configured for subscription denial tests
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E1-S5 — Auto-Reconnect & Connection Resilience

**Parent Epic**: BDCSPM-70937-E1 — MQTT Protocol Engine & Secure Device Connectivity
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an Operations Team member,
I want RDGC to automatically recover MQTT connections after broker outages,
So that monitoring is restored without manual intervention.

**Context & notes**:
- Exponential backoff: initial 1s, max 5 minutes
- Configurable max retry attempts (default: unlimited)
- Re-subscribe to all topics on successful reconnect
- Connection state transitions must be logged for audit/troubleshooting
- On-prem consideration: broker may be restarted during maintenance windows

**Acceptance Criteria**

AC1:
  Given: An active MQTT connection
  When: The broker becomes unavailable (network failure, broker restart)
  Then: RDGC detects the disconnect and initiates reconnection within 5 seconds

AC2:
  Given: A reconnection attempt is in progress
  When: The first attempt fails
  Then: Subsequent attempts use exponential backoff (1s, 2s, 4s, 8s, ... max 5min)

AC3:
  Given: A successful reconnection after broker outage
  When: Connection is restored
  Then: All previously configured topic subscriptions are re-established automatically

AC4:
  Given: Repeated reconnection failures
  When: The connection state is queried
  Then: The state reports "disconnected" with the last error reason and reconnect attempt count

**Tasks**
- [ ] Task 1: Implement disconnect detection (keep-alive timeout, TCP reset)
- [ ] Task 2: Implement exponential backoff reconnect logic
- [ ] Task 3: Implement automatic re-subscription on reconnect
- [ ] Task 4: Add reconnect configuration (initialDelay, maxDelay, maxRetries)
- [ ] Task 5: Implement connection state machine (connected → disconnecting → reconnecting → connected)
- [ ] Task 6: Add connection event logging (connect, disconnect, reconnect-attempt, reconnect-success)
- [ ] Task 7: Write integration tests simulating broker restart scenarios

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Test environment supports broker restart simulation
- [ ] Reconnect behavior agreed (unlimited retries as default)
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E1-S6 — Internal Connection State API

**Parent Epic**: BDCSPM-70937-E1 — MQTT Protocol Engine & Secure Device Connectivity
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As a Web UI developer,
I want an internal API that exposes MQTT connection states and health metrics,
So that the UI layer can display device connectivity status to administrators.

**Context & notes**:
- Internal REST or message-bus API (not externally exposed)
- States: connected, disconnected, reconnecting, error
- Metrics: uptime, reconnect count, last message received timestamp, messages/sec
- Per-broker and per-device granularity
- Event-driven notifications for state changes (e.g., via internal message bus)

**Acceptance Criteria**

AC1:
  Given: RDGC has active MQTT connections
  When: The internal state API is queried
  Then: Current connection state, uptime, and last activity timestamp are returned for each broker

AC2:
  Given: An MQTT connection transitions from connected to disconnected
  When: The state change occurs
  Then: An internal event/notification is emitted within 5 seconds

AC3:
  Given: Multiple MQTT brokers are configured
  When: The state API is queried with a specific broker ID
  Then: Only that broker's state and metrics are returned

**Tasks**
- [ ] Task 1: Design internal API contract (REST or message bus)
- [ ] Task 2: Implement connection state endpoint (GET /internal/mqtt/connections)
- [ ] Task 3: Implement per-broker state and metrics (uptime, reconnectCount, lastMessageAt)
- [ ] Task 4: Implement state-change event emission (internal message bus)
- [ ] Task 5: Write unit tests for API responses
- [ ] Task 6: Write integration tests for state transition notifications

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Internal API contract reviewed by UI team
- [ ] Message bus integration approach agreed (if event-driven)
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E1-S7 — Coexistence & Performance Validation

**Parent Epic**: BDCSPM-70937-E1 — MQTT Protocol Engine & Secure Device Connectivity
**BLSS Product**: Distributed IT Performance Management (RDGC)

**Description**
As an Operations Team member,
I want MQTT monitoring to operate without degrading existing RDGC protocols,
So that adding MQTT devices does not impact the reliability of current monitoring.

**Context & notes**:
- Performance test on reference hardware with mixed workload (SNMP + Modbus + MQTT)
- Validate CPU, memory, and network utilization within NFR targets
- Test with maximum broker connections (10) and topic subscriptions (50/broker)
- Validate no impact on existing SNMP/Modbus polling intervals

**Acceptance Criteria**

AC1:
  Given: RDGC is monitoring 100 SNMP devices and 50 Modbus devices
  When: 10 MQTT broker connections with 50 topics each are added
  Then: Existing SNMP/Modbus polling intervals remain within 5% of baseline

AC2:
  Given: MQTT engine is running at maximum configured capacity
  When: System resources are measured
  Then: Additional CPU < 5% and additional RAM < 256 MB vs. baseline (no MQTT)

AC3:
  Given: An MQTT broker connection experiences rapid disconnect/reconnect cycles
  When: Reconnection storms occur
  Then: Other RDGC services (SNMP, Modbus, Web UI) are not affected

**Tasks**
- [ ] Task 1: Define performance test plan and reference hardware configuration
- [ ] Task 2: Establish baseline metrics (CPU, RAM, polling accuracy) without MQTT
- [ ] Task 3: Execute load test with maximum MQTT connections and subscriptions
- [ ] Task 4: Execute coexistence test with mixed protocol workload
- [ ] Task 5: Execute reconnection storm test
- [ ] Task 6: Document results and update sizing guide
- [ ] Task 7: Identify and fix any performance regressions

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Reference hardware environment provisioned
- [ ] Performance test tools and scripts prepared
- [ ] Baseline measurements captured
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

## Validation Checklist

- [x] All AC are testable
- [x] NFRs are quantified (TLS 1.2+, 30s reconnect, 10 brokers, 50 topics, <5% CPU, <256MB RAM)
- [x] BLSS product explicitly identified (Distributed IT Performance Management — RDGC)
- [x] Scope boundaries are explicit
- [x] Cross-product dependencies identified (none in this Epic)
- [x] On-prem operational impact assessed (new module, port 8883, config backup)
- [x] Dependencies identified (BDCSPM-6541, library selection)
- [x] No invented features, UI labels, device models, or protocol details
- [x] Terminology matches BLSS product glossary
- [x] Placeholders marked with [TBD] where applicable
- [x] Operational deliverables included

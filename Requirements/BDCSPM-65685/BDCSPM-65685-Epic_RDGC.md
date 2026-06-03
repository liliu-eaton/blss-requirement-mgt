# BDCSPM-65685 — RDGC Architecture Enhancement: Connection Stability, Resilience & Diagnostics

## EPIC 1: RDGC-RDGS Connection Stability, Self-Healing & Remote Diagnostics

**BLSS Product(s)**: DCPM / Distributed IT Performance Management (Remote Data Gateway — RDGC & RDGS components)

**Summary**
Enhance the RDGC architecture to achieve stable, resilient connections between RDGC and RDGS, provide self-healing and remote recovery capabilities, and enable end-users to collect diagnostic information (logs, configurations, system state) via the UI for efficient problem isolation by BLSS support.

---

### Description

- **Problem statement**: The current RDGC-RDGS communication architecture suffers from chronic stability issues. The MQTT-based connection between RDGC and RDGS experiences frequent disconnections (see related defects: BDCSPM-66638 "All RDGCs disconnected with RDGS", BDCSPM-66338 "RDGC keeps reporting MQTT disconnected warnings", BDCSPM-37494 "RDGC will occasionally disconnect", BDCSPM-56990 "RDGC disconnects every five minutes"). The heartbeat mechanism used for online/offline detection is overly complex, causing false-offline reporting while RDGC continues to publish data. When disconnections occur, troubleshooting is difficult: there is no self-service diagnostic data collection for end-users, recovery is manual, and BLSS support cannot efficiently isolate root causes remotely.

- **Proposed solution**:
  1. **Connection Stability Layer**: Redesign the RDGC-RDGS communication mechanism to be more resilient — simplify the heartbeat/keepalive logic, implement robust MQTT reconnection strategies with exponential backoff, and ensure the online/offline state accurately reflects actual data publishing status.
  2. **Self-Healing & Remote Recovery**: Implement automatic recovery mechanisms when connection anomalies are detected (e.g., automatic reconnection, service restart orchestration, configuration re-synchronization) without requiring manual intervention.
  3. **Diagnostic Data Collection & Download**: Provide a Server Admin UI page where administrators can trigger diagnostic data bundle generation (logs, RDGC/RDGS configuration, connection state, system metrics) and download the bundle for submission to BLSS support.
  4. **Connection Health Monitoring**: Real-time connection status dashboard with historical connection metrics, reconnection event timeline, and alerting for persistent disconnection patterns.

- **User personas**: On-Prem IT/System Admin, BLSS Technical Support Engineer, Commissioning Engineer, Lifecycle Services Engineer

- **Business value**:
  - Eliminates recurring field escalations caused by RDGC disconnection issues (top support ticket category for RDG)
  - Reduces mean-time-to-resolution (MTTR) for connection issues from hours/days to minutes via self-service diagnostics
  - Reduces on-site intervention requirements through remote recovery capabilities
  - Improves overall system reliability and customer confidence in RDG architecture
  - Enables BLSS support to perform root-cause analysis without VPN access to customer site

- **Key workflows**:
  1. RDGC establishes MQTT connection to RDGS with simplified, reliable keepalive mechanism
  2. On connection loss, RDGC automatically attempts reconnection using exponential backoff strategy
  3. After configurable retry threshold, RDGC triggers self-healing sequence (service restart, config reload)
  4. System accurately reports RDGC online/offline status based on actual connection + data publishing state
  5. Admin navigates to Server Admin → RDG Management → Diagnostics page
  6. Admin triggers "Generate Diagnostic Bundle" which collects logs, config, connection history, system state
  7. Admin downloads the bundle (.zip) and provides to BLSS support for analysis
  8. BLSS support uses the bundle to identify root cause without requiring remote access

- **Cross-product touchpoints**: RDGC connection stability impacts all BLSS products that rely on remote site data collection (DCPM for remote facility monitoring, Distributed IT for edge site management). Diagnostic bundle download uses the ServerAdmin UI framework shared across all products.

- **UX/UI references**: `[TBD — UX mockup for RDG Diagnostics page and Connection Health dashboard]`

- **On-prem operational impact**:
  - Install/upgrade impact: New diagnostics service component on RDGC; updated RDGS connection management module. Upgrade must preserve existing RDGC registrations and connection configurations.
  - Sizing impact: Additional disk space on RDGC for diagnostic log retention (configurable, default 500MB rolling). Minimal CPU/RAM overhead for connection health monitoring.
  - Network impact: No new ports required — uses existing MQTT channel. Diagnostic bundle download via existing ServerAdmin HTTPS port.
  - Backup impact: Connection configuration and health history included in RDGC snapshot/backup scope.

---

### Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Reliability | RDGC-RDGS connection uptime under normal network conditions | ≥ 99.9% (< 8.76 hours downtime/year) | P0 |
| Reliability | Automatic reconnection success rate after transient network failure (< 30s) | ≥ 99% within 60 seconds | P0 |
| Reliability | Self-healing recovery success rate after persistent disconnection | ≥ 95% without manual intervention | P1 |
| Performance | Heartbeat/keepalive overhead on MQTT channel | < 1% of available bandwidth | P1 |
| Performance | Diagnostic bundle generation time (typical system with 30-day logs) | < 60 seconds | P2 |
| Performance | Diagnostic bundle size (compressed) | < 50MB for 30-day retention | P2 |
| Availability | Maximum time to detect actual RDGC offline state | < 90 seconds | P0 |
| Availability | Maximum false-positive offline detection rate | < 0.1% (< 1 per 1000 heartbeat cycles) | P0 |
| Maintainability | Diagnostic bundle must contain sufficient information for L2 support to identify root cause | ≥ 80% of common issues identifiable without remote access | P1 |
| Security | Diagnostic bundle must NOT contain credentials, certificates, or private keys | Zero credential leakage | P0 |
| Security | Diagnostic bundle download requires Administrator role authentication | Role-based access enforced | P0 |
| Scalability | Connection stability under maximum supported RDGC count per RDGS | Stable with 100 concurrent RDGCs per RDGS instance | P1 |

---

### Acceptance Criteria (Epic-level)

- [ ] AC1: RDGC maintains a stable MQTT connection to RDGS for ≥ 72 continuous hours under normal network conditions without spurious disconnection events
- [ ] AC2: When network interruption occurs (< 30s), RDGC automatically reconnects within 60 seconds without manual intervention
- [ ] AC3: When RDGC experiences persistent disconnection (> 5 minutes), the self-healing mechanism triggers and restores connection within 5 minutes in ≥ 95% of cases
- [ ] AC4: The RDGS online/offline status display matches actual RDGC connection + data publishing state with < 0.1% false-positive rate
- [ ] AC5: Administrator can generate and download a diagnostic bundle from the Server Admin UI containing logs, configuration, and connection history
- [ ] AC6: Diagnostic bundle does NOT contain any credentials, private keys, or security-sensitive secrets
- [ ] AC7: BLSS L2 support can identify root cause of ≥ 80% of common RDGC connection issues using only the diagnostic bundle (validated via support team review)
- [ ] AC8: Connection health metrics (uptime, reconnection events, latency) are visible in the Server Admin UI
- [ ] AC9: All self-healing and recovery actions are logged with timestamp, trigger reason, and outcome for audit purposes

---

### Scope

- **In scope**:
  - RDGC-RDGS MQTT connection stability improvements (keepalive simplification, reconnection strategies)
  - Accurate online/offline state detection based on actual connection + data publishing status
  - Automatic reconnection with exponential backoff and jitter
  - Self-healing mechanism (service restart, configuration re-sync) for persistent failures
  - Connection health monitoring dashboard in Server Admin UI
  - Diagnostic bundle generation and download (logs, config, connection history, system state)
  - Sanitization of sensitive data from diagnostic bundles
  - Connection event timeline and historical metrics
  - Configurable log retention policy for RDGC diagnostic data
  - Alert/notification for persistent disconnection patterns

- **Out of scope**:
  - RDGC registration via RSA (covered separately in BDCSPM-52780)
  - RSA-based communication gateway changes
  - RDGC container lifecycle management via RSA
  - Changes to RDGC data collection protocols (SNMP, Modbus, BACnet)
  - RDGS horizontal scaling / clustering
  - Third-party monitoring tool integration (Prometheus, Grafana export)
  - Customer network infrastructure troubleshooting

- **Assumptions**:
  - ⚠️ ASSUMPTION 1: Existing MQTT broker infrastructure (within RDGS) is retained; stability improvements are at the client connection management layer
  - ⚠️ ASSUMPTION 2: RDGC runs as a containerized service with restart capability via local orchestrator
  - ⚠️ ASSUMPTION 3: Server Admin UI framework supports file download (zip bundle) via existing HTTPS endpoint
  - ⚠️ ASSUMPTION 4: Diagnostic log retention default of 30 days / 500MB is acceptable; confirm with Support team
  - ⚠️ ASSUMPTION 5: The maximum supported RDGC count per RDGS instance is 100; confirm with architecture team

---

### Dependencies & Risks

| Item | Type | BLSS Product | Owner | Mitigation | Status |
|---|---|---|---|---|---|
| MQTT broker version compatibility for enhanced keepalive | Dep | RDG (RDGS) | Platform Engineering | Validate with current EMQX/Mosquitto version in use | `[TBD]` |
| Server Admin UI framework file download capability | Dep | Shared Platform | UI Team | Confirm existing download pattern or implement new | `[TBD]` |
| RDGC container orchestration (restart capability) | Dep | RDG (RDGC) | Platform Engineering | Confirm Docker/systemd restart hooks available | `[TBD]` |
| Network firewall may block MQTT reconnection in some customer environments | Risk | N/A | Customer | Document network requirements; implement connection diagnostics in bundle | Open |
| Self-healing restart may cause brief data collection interruption | Risk | DCPM / Distributed IT | RDG Team | Implement graceful restart with data buffering | Open |
| Diagnostic bundle may be too large in high-volume environments | Risk | RDG | RDG Team | Implement configurable retention and bundle size limits | Open |
| Coordination with BDCSPM-52780 (RSA gateway) to avoid conflicting architecture changes | Dep | RDG | PO (Lena Liu) | Scope boundaries clearly defined; RSA registration excluded | Open |

---

### Operational Deliverables (required for on-prem)

- [ ] Installation/upgrade runbook updated with new diagnostics service configuration
- [ ] Sizing guide updated with diagnostic log storage requirements
- [ ] Backup/restore procedure updated to include connection configuration and health history
- [ ] Release notes entry: connection stability improvements, diagnostic bundle feature, self-healing capability
- [ ] Lifecycle Services troubleshooting guide updated with diagnostic bundle interpretation
- [ ] BLSS Support playbook: how to analyze diagnostic bundle for common RDGC issues

---

### Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| S1 | Simplify RDGC keepalive/heartbeat mechanism | 8 | `[TBD]` |
| S2 | Implement robust MQTT reconnection with exponential backoff | 5 | `[TBD]` |
| S3 | Accurate online/offline state detection based on connection + data flow | 8 | `[TBD]` |
| S4 | Self-healing mechanism for persistent RDGC disconnection | 8 | `[TBD]` |
| S5 | Connection health monitoring & metrics collection | 5 | `[TBD]` |
| S6 | Server Admin UI - Connection Health Dashboard | 5 | `[TBD]` |
| S7 | Diagnostic bundle generation service (logs, config, state) | 8 | `[TBD]` |
| S8 | Server Admin UI - Diagnostic bundle download page | 5 | `[TBD]` |
| S9 | Sensitive data sanitization for diagnostic bundles | 3 | `[TBD]` |
| S10 | Connection event alerting for persistent disconnection patterns | 3 | `[TBD]` |

---

## Story Details

---

### STORY S1: Simplify RDGC Keepalive/Heartbeat Mechanism

**Parent Epic**: BDCSPM-65685-Epic_1 — RDGC-RDGS Connection Stability, Self-Healing & Remote Diagnostics
**BLSS Product**: RDG (RDGC component)

**Description**
As a Platform Engineer,
I want a simplified and reliable heartbeat mechanism between RDGC and RDGS,
So that the system can quickly and accurately detect RDGC connectivity status without complex payload construction that leads to false-offline events.

**Context & notes**:
- Current heartbeat payload construction is overly complex, causing heartbeats to be missing in certain scenarios (per BDCSPM-65685 description)
- RDGC already publishes data via MQTT; heartbeat should leverage the same channel with minimal overhead
- Simplified heartbeat = lightweight message (e.g., timestamp + RDGC ID only) on fixed interval
- Must maintain backward compatibility with existing RDGS versions during rolling upgrade

**Acceptance Criteria**

AC1:
  Given: RDGC is connected to RDGS and publishing data normally
  When: Heartbeat interval timer fires (default 30s, configurable)
  Then: A lightweight heartbeat message (< 256 bytes) is published to the MQTT heartbeat topic within 1 second

AC2:
  Given: RDGC is under high data publishing load (100+ devices reporting simultaneously)
  When: Heartbeat interval timer fires
  Then: Heartbeat is still sent on time (within ±5s of interval) without being blocked by data publishing

AC3:
  Given: The RDGC heartbeat configuration is set to default values
  When: System operates for 72 continuous hours
  Then: Zero missed heartbeats are observed in the RDGS log

AC4 (negative):
  Given: RDGC network connection to RDGS is physically severed
  When: Heartbeat interval fires
  Then: Heartbeat failure is logged locally on RDGC with timestamp and error reason; reconnection flow is triggered

**Tasks**
- [ ] Task 1: Design simplified heartbeat message schema (RDGC ID + timestamp + sequence number)
- [ ] Task 2: Refactor heartbeat sender to decouple from data publishing pipeline (separate thread/task)
- [ ] Task 3: Make heartbeat interval configurable via RDGC config file (default 30s, min 10s, max 120s)
- [ ] Task 4: Update RDGS heartbeat receiver to accept new simplified format (backward compatible)
- [ ] Task 5: Add heartbeat send/receive metrics (count, latency, missed) to internal metrics
- [ ] Task 6: Write unit tests for heartbeat sender under various load conditions
- [ ] Task 7: Write integration test: 72-hour stability test with concurrent data publishing

---

### STORY S2: Implement Robust MQTT Reconnection with Exponential Backoff

**Parent Epic**: BDCSPM-65685-Epic_1
**BLSS Product**: RDG (RDGC component)

**Description**
As a System Administrator,
I want RDGC to automatically reconnect to RDGS after a network interruption using intelligent retry logic,
So that temporary network issues do not require manual intervention to restore the data gateway connection.

**Context & notes**:
- Current behavior: RDGC may remain disconnected or enter rapid reconnection loop (thundering herd problem)
- New behavior: exponential backoff with jitter (initial 2s, max 5min, jitter ±20%)
- Must handle both clean disconnect (MQTT DISCONNECT) and unclean disconnect (TCP timeout)
- On successful reconnection, must re-subscribe to all command topics and resume data publishing
- Reconnection attempts and outcomes must be logged for diagnostics

**Acceptance Criteria**

AC1:
  Given: RDGC is connected to RDGS
  When: Network interruption occurs for < 30 seconds
  Then: RDGC automatically reconnects within 60 seconds and resumes data publishing without data loss (buffered during disconnect)

AC2:
  Given: RDGC has been disconnected for > 5 minutes
  When: Network is restored
  Then: RDGC reconnects using backoff schedule and resumes within 30 seconds of network availability

AC3:
  Given: 50+ RDGCs are disconnected simultaneously from one RDGS
  When: Network is restored
  Then: RDGCs reconnect with randomized jitter, preventing thundering herd; RDGS CPU stays < 80% during reconnection storm

AC4:
  Given: RDGC is in reconnection backoff state
  When: Administrator views RDGC status in Server Admin
  Then: Status shows "Reconnecting" with next retry timestamp and attempt count

**Tasks**
- [ ] Task 1: Implement exponential backoff algorithm (base 2s, multiplier 2x, max 5min, jitter ±20%)
- [ ] Task 2: Implement local message buffer during disconnection (configurable size, default 1000 messages or 10MB)
- [ ] Task 3: Implement reconnection state machine (Connected → Disconnected → Reconnecting → Connected)
- [ ] Task 4: Add reconnection metrics (attempt count, total disconnect duration, buffer utilization)
- [ ] Task 5: Test thundering herd scenario with 100 concurrent RDGCs
- [ ] Task 6: Implement graceful buffer flush on reconnection (rate-limited to avoid RDGS overload)

---

### STORY S3: Accurate Online/Offline State Detection

**Parent Epic**: BDCSPM-65685-Epic_1
**BLSS Product**: RDG (RDGS component)

**Description**
As a System Administrator,
I want the RDGC online/offline status displayed in the system to accurately reflect the real connection and data publishing state,
So that I can trust the monitoring dashboard and make informed operational decisions.

**Context & notes**:
- Current issue: RDGC may be marked "offline" while still actively publishing data (false positive)
- Root cause: heartbeat-only detection misses cases where heartbeat fails but data flow continues
- New approach: dual-signal detection — connection state (MQTT session) AND data flow activity (last message timestamp)
- RDGC is "online" if MQTT session is active AND (heartbeat received within threshold OR data message received within threshold)
- RDGC is "offline" only if MQTT session is disconnected AND no data messages received within 2× heartbeat interval

**Acceptance Criteria**

AC1:
  Given: RDGC has an active MQTT session and is publishing data
  When: Heartbeat is missed (e.g., due to temporary processing delay)
  Then: RDGC status remains "Online" because data messages are still being received

AC2:
  Given: RDGC MQTT session disconnects
  When: 2× heartbeat interval (default 60s) passes with no data messages received
  Then: RDGC status changes to "Offline" with timestamp of last known activity

AC3:
  Given: RDGC reconnects after being offline
  When: First heartbeat or data message is received by RDGS
  Then: RDGC status changes to "Online" within 5 seconds

AC4:
  Given: System has 100 registered RDGCs
  When: All are operating normally for 24 hours
  Then: Zero false-positive "Offline" events are generated

**Tasks**
- [ ] Task 1: Implement dual-signal state detector on RDGS (MQTT session state + last message timestamp)
- [ ] Task 2: Add "last data message received" timestamp tracking per RDGC on RDGS
- [ ] Task 3: Implement configurable offline detection threshold (default: 2× heartbeat interval)
- [ ] Task 4: Update status API to return connection state with confidence level and evidence
- [ ] Task 5: Update Server Admin UI status display to reflect new state logic
- [ ] Task 6: Write integration test: simulate heartbeat miss while data flows (verify no false offline)
- [ ] Task 7: Write integration test: 24-hour soak test with 100 RDGCs verifying zero false positives

---

### STORY S4: Self-Healing Mechanism for Persistent RDGC Disconnection

**Parent Epic**: BDCSPM-65685-Epic_1
**BLSS Product**: RDG (RDGC component)

**Description**
As a System Administrator,
I want RDGC to automatically attempt self-recovery when a persistent disconnection is detected,
So that remote sites can recover without requiring on-site or remote manual intervention.

**Context & notes**:
- "Persistent disconnection" = failed to reconnect after exhausting the exponential backoff cycle (e.g., > 5 minutes)
- Self-healing sequence: (1) Reset MQTT client → (2) Reload configuration → (3) Restart RDGC service → (4) Report to RDGS if all recovery failed
- Each step is attempted before escalating to the next
- All recovery actions must be logged for audit and post-mortem analysis
- Data buffered during recovery; maximum buffer prevents disk exhaustion

**Acceptance Criteria**

AC1:
  Given: RDGC has failed to reconnect after exponential backoff cycle (> 5 minutes disconnected)
  When: Self-healing mechanism activates
  Then: RDGC attempts recovery steps in sequence (MQTT client reset → config reload → service restart)

AC2:
  Given: Self-healing step 1 (MQTT client reset) succeeds
  When: Connection is restored
  Then: Normal operation resumes within 30 seconds; recovery event is logged with "resolved at step 1"

AC3:
  Given: All self-healing steps fail
  When: Maximum recovery attempts exhausted (3 cycles)
  Then: RDGC enters "Recovery Failed" state, logs detailed failure information, and RDGS displays alert for administrator attention

AC4:
  Given: Self-healing triggers a service restart
  When: Restart completes
  Then: All previously buffered data messages are flushed to RDGS in order; no data loss occurs

**Tasks**
- [ ] Task 1: Implement self-healing state machine (Normal → Recovering → Recovered / FailedRecovery)
- [ ] Task 2: Implement Step 1: MQTT client session reset without service restart
- [ ] Task 3: Implement Step 2: Configuration reload (re-read connection params, certificates)
- [ ] Task 4: Implement Step 3: Graceful RDGC service restart with data buffer preservation
- [ ] Task 5: Implement recovery exhaustion detection and "Recovery Failed" state notification to RDGS
- [ ] Task 6: Add configurable recovery parameters (max cycles, step timeout, cooldown period)
- [ ] Task 7: Write integration test: simulate persistent failure and verify self-healing sequence

---

### STORY S5: Connection Health Monitoring & Metrics Collection

**Parent Epic**: BDCSPM-65685-Epic_1
**BLSS Product**: RDG (RDGC + RDGS)

**Description**
As a BLSS Support Engineer,
I want connection health metrics collected and stored for each RDGC,
So that I can analyze connection patterns, identify degradation trends, and perform root-cause analysis.

**Context & notes**:
- Metrics collected on RDGC side: connection uptime, reconnection count, heartbeat latency, buffer utilization, self-healing events
- Metrics aggregated on RDGS side: per-RDGC connection timeline, aggregate stability score
- Stored in local time-series format with configurable retention (default 30 days)
- Exposed via internal API for UI consumption and inclusion in diagnostic bundle

**Acceptance Criteria**

AC1:
  Given: RDGC is operating normally
  When: Metrics collection interval fires (every 60 seconds)
  Then: Connection metrics (uptime, latency, heartbeat count, buffer level) are recorded locally

AC2:
  Given: A reconnection event occurs
  When: Connection is restored
  Then: Event is recorded with: disconnect timestamp, reconnect timestamp, duration, cause (if identifiable), recovery method

AC3:
  Given: 30 days of metrics are stored
  When: Retention policy check runs
  Then: Metrics older than retention period are automatically purged; disk usage stays within configured limit

**Tasks**
- [ ] Task 1: Define metrics schema (connection_uptime, reconnect_count, heartbeat_latency_ms, buffer_utilization_pct, etc.)
- [ ] Task 2: Implement metrics collector on RDGC (60s interval, local storage)
- [ ] Task 3: Implement metrics aggregation on RDGS (per-RDGC summary via heartbeat/report)
- [ ] Task 4: Implement retention policy with automatic purge
- [ ] Task 5: Expose metrics via internal REST API for UI and diagnostic bundle consumption
- [ ] Task 6: Write unit tests for metrics collection and retention

---

### STORY S6: Server Admin UI — Connection Health Dashboard

**Parent Epic**: BDCSPM-65685-Epic_1
**BLSS Product**: Shared Platform (Server Admin UI)

**Description**
As a System Administrator,
I want a visual dashboard showing RDGC connection health status and history,
So that I can quickly assess the health of all remote data gateways and identify problematic connections.

**Context & notes**:
- Dashboard located at Server Admin → RDG Management → Connection Health
- Shows: list of all RDGCs with current status (Online/Reconnecting/Offline/Recovery Failed), uptime percentage, last connected time
- Click into individual RDGC shows: connection timeline (graph), reconnection events, self-healing history
- Auto-refresh every 30 seconds; manual refresh available

**Acceptance Criteria**

AC1:
  Given: Administrator navigates to Server Admin → RDG Management → Connection Health
  When: Page loads
  Then: All registered RDGCs are listed with current status, uptime %, and last activity timestamp (page loads < 3s)

AC2:
  Given: An RDGC transitions from Online to Offline
  When: Dashboard auto-refreshes (30s interval)
  Then: Status indicator updates to Offline with visual alert (color change to red)

AC3:
  Given: Administrator clicks on a specific RDGC entry
  When: Detail view opens
  Then: Connection timeline graph (last 7 days), reconnection events list, and self-healing history are displayed

**Tasks**
- [ ] Task 1: Design UI wireframe for Connection Health dashboard (list view + detail view)
- [ ] Task 2: Implement RDGC list view with status indicators, uptime %, last activity
- [ ] Task 3: Implement RDGC detail view with connection timeline chart (last 7d/30d toggle)
- [ ] Task 4: Implement auto-refresh (30s) and manual refresh
- [ ] Task 5: Integrate with metrics API (Story S5)
- [ ] Task 6: Write E2E tests for dashboard navigation and data display

---

### STORY S7: Diagnostic Bundle Generation Service

**Parent Epic**: BDCSPM-65685-Epic_1
**BLSS Product**: RDG (RDGC component)

**Description**
As a BLSS Support Engineer,
I want RDGC to generate a comprehensive diagnostic data bundle on demand,
So that I can analyze connection issues remotely without requiring VPN access to the customer environment.

**Context & notes**:
- Diagnostic bundle contents: RDGC application logs (last 7 days), RDGC configuration (sanitized), connection metrics history, MQTT client state, system info (OS, memory, disk, network interfaces), container state (if applicable), recent self-healing event log
- Bundle format: compressed ZIP file, named `rdgc-diag-{rdgc-id}-{timestamp}.zip`
- Generation triggered via REST API (consumed by Server Admin UI)
- Must sanitize credentials, certificates private keys, and tokens before inclusion
- Bundle size target: < 50MB compressed for 30-day log retention

**Acceptance Criteria**

AC1:
  Given: Administrator triggers diagnostic bundle generation via API
  When: Bundle generation completes
  Then: A ZIP file is created containing: application logs, sanitized configuration, connection metrics, system info, self-healing history

AC2:
  Given: RDGC configuration contains passwords, private keys, or API tokens
  When: Configuration is included in the diagnostic bundle
  Then: All sensitive values are replaced with "[REDACTED]" placeholder

AC3:
  Given: RDGC has 30 days of logs at normal verbosity
  When: Bundle is generated
  Then: Bundle generation completes within 60 seconds and resulting file is < 50MB

AC4 (negative):
  Given: RDGC disk is > 95% full
  When: Bundle generation is triggered
  Then: Generation fails gracefully with error message "Insufficient disk space"; no partial file left on disk

**Tasks**
- [ ] Task 1: Define diagnostic bundle manifest (file list, formats, size limits per component)
- [ ] Task 2: Implement log collector (last 7 days of RDGC application logs, rotated files included)
- [ ] Task 3: Implement configuration sanitizer (regex-based redaction of passwords, keys, tokens)
- [ ] Task 4: Implement system info collector (OS version, memory, disk, network, container state)
- [ ] Task 5: Implement metrics export (connection history in CSV/JSON format)
- [ ] Task 6: Implement ZIP packager with size validation and disk space pre-check
- [ ] Task 7: Expose bundle generation REST API endpoint (POST /api/v1/rdgc/diagnostics/bundle)
- [ ] Task 8: Write integration test: generate bundle, verify contents, verify no secrets present

---

### STORY S8: Server Admin UI — Diagnostic Bundle Download Page

**Parent Epic**: BDCSPM-65685-Epic_1
**BLSS Product**: Shared Platform (Server Admin UI)

**Description**
As a System Administrator,
I want to generate and download RDGC diagnostic bundles from the Server Admin web interface,
So that I can easily collect troubleshooting data and provide it to BLSS support without command-line access.

**Context & notes**:
- Page located at Server Admin → RDG Management → Diagnostics
- Shows list of RDGCs; admin selects one or more RDGCs and clicks "Generate Diagnostic Bundle"
- Progress indicator during generation; download link when complete
- Historical bundles listed with generation timestamp (retained for 7 days on server, auto-purged)
- Role-based access: Administrator role required

**Acceptance Criteria**

AC1:
  Given: Administrator navigates to Server Admin → RDG Management → Diagnostics
  When: Page loads
  Then: List of registered RDGCs is displayed with last bundle generation timestamp (if any)

AC2:
  Given: Administrator selects an RDGC and clicks "Generate Diagnostic Bundle"
  When: Bundle generation is in progress
  Then: Progress indicator shows generation status; page remains responsive

AC3:
  Given: Bundle generation completes successfully
  When: Administrator clicks "Download"
  Then: Browser downloads the ZIP file (named `rdgc-diag-{rdgc-id}-{timestamp}.zip`)

AC4:
  Given: A user with Operator role (not Administrator) navigates to the Diagnostics page
  When: Page loads
  Then: "Generate" and "Download" buttons are disabled with tooltip "Administrator role required"

**Tasks**
- [ ] Task 1: Design UI wireframe for Diagnostics page (RDGC list + generate + download + history)
- [ ] Task 2: Implement RDGC selector with multi-select capability
- [ ] Task 3: Implement "Generate" button with progress indicator (polling generation status API)
- [ ] Task 4: Implement download link generation and file download via browser
- [ ] Task 5: Implement historical bundle list with auto-purge (7-day retention)
- [ ] Task 6: Implement role-based access control (Administrator only)
- [ ] Task 7: Write E2E tests for generate/download workflow

---

### STORY S9: Sensitive Data Sanitization for Diagnostic Bundles

**Parent Epic**: BDCSPM-65685-Epic_1
**BLSS Product**: RDG (RDGC component)

**Description**
As a Security-conscious Administrator,
I want all sensitive information automatically removed from diagnostic bundles before download,
So that I can safely share the bundle with BLSS support without risk of credential exposure.

**Context & notes**:
- Sanitization targets: passwords, private keys (PEM format), API tokens, MQTT credentials, database connection strings, certificate private keys
- Approach: pattern-based scanning + known config key redaction
- Configuration keys to redact: any key containing "password", "secret", "token", "key", "credential", "private"
- PEM private key blocks: `-----BEGIN.*PRIVATE KEY-----` through `-----END.*PRIVATE KEY-----`
- Sanitized values replaced with `[REDACTED]`
- Sanitization log included in bundle (list of redacted items without values)

**Acceptance Criteria**

AC1:
  Given: RDGC configuration file contains `mqtt.password=MySecretPass123`
  When: Diagnostic bundle is generated
  Then: Bundle contains `mqtt.password=[REDACTED]`

AC2:
  Given: Log file contains a line with an embedded certificate private key
  When: Diagnostic bundle is generated
  Then: Private key content is replaced with `[REDACTED - PRIVATE KEY]`

AC3:
  Given: Diagnostic bundle is generated
  When: Sanitization completes
  Then: A `sanitization-report.txt` file is included listing count and categories of redacted items (without revealing values)

**Tasks**
- [ ] Task 1: Define sanitization rules (regex patterns for secrets, config key patterns, PEM blocks)
- [ ] Task 2: Implement config file sanitizer (key-based + pattern-based redaction)
- [ ] Task 3: Implement log file sanitizer (pattern-based scanning for embedded secrets)
- [ ] Task 4: Implement PEM private key block detection and redaction
- [ ] Task 5: Generate sanitization report (summary only, no values)
- [ ] Task 6: Write unit tests with representative sensitive data patterns (no real secrets in test fixtures)

---

### STORY S10: Connection Event Alerting for Persistent Disconnection Patterns

**Parent Epic**: BDCSPM-65685-Epic_1
**BLSS Product**: RDG (RDGS component)

**Description**
As a System Administrator,
I want to be alerted when RDGC connections show persistent instability patterns,
So that I can proactively address network or configuration issues before they impact data collection reliability.

**Context & notes**:
- Alert triggers: (1) RDGC offline > configurable threshold (default 10 min), (2) Reconnection frequency > configurable threshold (default 5 reconnections in 1 hour), (3) Self-healing failure
- Alert delivery: System alarm in Server Admin notification center + optional email notification
- Alert includes: RDGC ID, site name, trigger reason, duration, recommendation
- De-duplication: same alert not repeated within 1 hour if condition persists

**Acceptance Criteria**

AC1:
  Given: An RDGC has been offline for > 10 minutes (configurable)
  When: Alert threshold is reached
  Then: A system alarm is raised in Server Admin notification center with RDGC details and recommended action

AC2:
  Given: An RDGC has reconnected 5 times within 1 hour (configurable)
  When: Pattern threshold is reached
  Then: A "Connection Instability" alarm is raised indicating frequent reconnection pattern

AC3:
  Given: An alert has been raised for an RDGC
  When: The condition persists for another hour
  Then: No duplicate alert is generated (de-duplication within 1-hour window)

AC4:
  Given: An offline RDGC reconnects and remains stable for > 1 hour
  When: Alert auto-clear check runs
  Then: Previous alert is automatically cleared with "Resolved" status

**Tasks**
- [ ] Task 1: Implement alert rule engine (configurable thresholds for offline duration, reconnection frequency)
- [ ] Task 2: Implement alert de-duplication logic (1-hour suppression window)
- [ ] Task 3: Integrate with Server Admin notification/alarm system
- [ ] Task 4: Implement alert auto-clear on condition resolution
- [ ] Task 5: Add alert configuration UI (threshold settings) in Server Admin → RDG Management → Settings
- [ ] Task 6: Write integration test for alert lifecycle (trigger → active → clear)

---

## Review Checklist

- [x] All AC are testable
- [x] NFRs are quantified
- [x] BLSS product(s) explicitly identified for every Epic and Story
- [x] Scope boundaries are explicit (RSA registration explicitly excluded)
- [x] Cross-product dependencies identified (Server Admin UI, DCPM/Distributed IT data flow)
- [x] On-prem operational impact assessed (install, upgrade, sizing, network, backup)
- [x] Dependencies identified (MQTT broker, Server Admin UI framework, container orchestration)
- [x] No invented features, UI labels, device models, or protocol details
- [x] Terminology matches BLSS product glossary (RDGC, RDGS, Server Admin, MQTT)
- [x] Placeholders marked with `[TBD]` where validation needed
- [x] Operational deliverables included (runbooks, sizing guide, release notes, support playbook)
- [ ] ⚠️ NEEDS VALIDATION: Maximum RDGC count per RDGS (assumed 100)
- [ ] ⚠️ NEEDS VALIDATION: MQTT broker implementation in RDGS (EMQX / Mosquitto / custom)
- [ ] ⚠️ NEEDS VALIDATION: RDGC container orchestration mechanism (Docker / systemd)
- [ ] ⚠️ NEEDS VALIDATION: Diagnostic log retention defaults (30 days / 500MB) acceptable to Support team

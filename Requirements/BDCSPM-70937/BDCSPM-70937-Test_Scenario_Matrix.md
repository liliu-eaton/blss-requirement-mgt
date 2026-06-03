# BDCSPM-70937 — Test Scenario Matrix

## Capability: RDGC has the ability to monitor devices using MQTT

| Test ID | Category | Test Scenario | Expected Result | Priority | Epic | Mapped AC |
|---|---|---|---|---|---|---|
| TC-MQTT-01 | Connection | Verify RDGC can establish a secure MQTT connection to a broker | TLS connection established; CONNACK received | P0 | E1 | E1-AC1 |
| TC-MQTT-02 | Connection | Verify successful subscription to configured topics | SUBACK received for all configured topics | P0 | E1 | E1-AC2 |
| TC-MQTT-03 | Connection | Verify connection with invalid broker host/port | Clear error logged; no crash; status = Error | P0 | E1 | E1-S1-AC2 |
| TC-MQTT-04 | Security | Verify MQTT connections require TLS encrypted transport | Non-TLS connections rejected or not supported | P0 | E1 | E1-S2-AC1 |
| TC-MQTT-05 | Security | Verify invalid/expired certificates prevent connection | Connection rejected; security warning logged | P0 | E1 | E1-S2-AC2 |
| TC-MQTT-06 | Security | Verify custom CA certificate support | Connection succeeds with customer CA-signed cert | P1 | E1 | E1-S2-AC3 |
| TC-MQTT-07 | Security | Verify invalid credentials prevent data ingestion | CONNACK failure; no subscription; error logged | P0 | E1 | E1-S3-AC2 |
| TC-MQTT-08 | Security | Verify mTLS authentication | Client cert handshake completes; connection authenticated | P1 | E1 | E1-S3-AC3 |
| TC-MQTT-09 | Security | Verify credentials are encrypted at rest | Config file shows encrypted values, not plaintext | P0 | E1 | E1-S3-AC4 |
| TC-MQTT-10 | Connection | Verify wildcard topic subscription (+ and #) | Messages matching wildcard are received | P1 | E1 | E1-S4-AC3 |
| TC-MQTT-11 | Connection | Verify dynamic unsubscribe without disconnect | Topic messages stop without full reconnection | P1 | E1 | E1-S4-AC2 |
| TC-MQTT-12 | Resilience | Simulate MQTT broker unavailability | RDGC detects disconnect and initiates reconnect within 5s | P0 | E1 | E1-S5-AC1 |
| TC-MQTT-13 | Resilience | Verify exponential backoff on reconnect failure | Retry intervals increase: 1s, 2s, 4s, 8s... max 5min | P0 | E1 | E1-S5-AC2 |
| TC-MQTT-14 | Resilience | Verify auto-resubscription after reconnect | All previous subscriptions restored on reconnect | P0 | E1 | E1-S5-AC3 |
| TC-MQTT-15 | Resilience | RDGC handles disconnect gracefully (no crash, no data corruption) | System remains stable; other protocols unaffected | P0 | E1 | E1-S7-AC3 |
| TC-MQTT-16 | Data Ingestion | Publish telemetry to MQTT topics and verify ingestion | Data ingested and visible in Brightlayer within 5s | P0 | E2 | E2-AC1 |
| TC-MQTT-17 | Data Ingestion | Verify JSON payload parsing with flat key-value | All values extracted with correct types | P0 | E2 | E2-S2-AC1 |
| TC-MQTT-18 | Data Ingestion | Verify JSON payload parsing with nested/JSONPath | Nested values extracted per configured mapping | P1 | E2 | E2-S2-AC2 |
| TC-MQTT-19 | Data Ingestion | Verify malformed JSON payload handling | Error logged; message skipped; other processing continues | P0 | E2 | E2-S2-AC3 |
| TC-MQTT-20 | Data Ingestion | Verify topic-to-device mapping | Message correctly associated with device from topic | P0 | E2 | E2-S3-AC1 |
| TC-MQTT-21 | Data Ingestion | Verify unmapped topic handling | Message logged as unmapped; metric incremented | P1 | E2 | E2-S3-AC3 |
| TC-MQTT-22 | Data Ingestion | Verify auto-provisioning of new MQTT device on first message | Device created in inventory with correct type | P1 | E2 | E2-S4-AC1 |
| TC-MQTT-23 | Data Ingestion | Verify MQTT device appears in device inventory | MQTT devices listed alongside SNMP/Modbus devices | P0 | E2 | E2-S4-AC2 |
| TC-MQTT-24 | Data Ingestion | Verify stale-data detection (no message within timeout) | Device status changes to "Communication Lost" | P1 | E2 | E2-S4-AC4 |
| TC-MQTT-25 | Data Ingestion | Verify message deduplication (QoS 1 redelivery) | Duplicate discarded; only one copy stored | P1 | E2 | E2-S6-AC1 |
| TC-MQTT-26 | Data Ingestion | Verify rate limiting per device | Excess messages dropped; warning logged | P2 | E2 | E2-S6-AC3 |
| TC-MQTT-27 | UI | Verify device connection status displayed in Web UI | Status badge shows connected/disconnected/error | P0 | E3 | E3-AC1 |
| TC-MQTT-28 | UI | Verify status updates on connect and disconnect events | UI reflects state change within 10 seconds | P0 | E3 | E3-AC2 |
| TC-MQTT-29 | UI | Verify default monitoring parameters are visible | Parameter table shows name, value, unit, timestamp | P0 | E3 | E3-AC3 |
| TC-MQTT-30 | UI | Verify polling/update frequency is displayed | Frequency shown per device (e.g., "Every 60s") | P0 | E3 | E3-AC4 |
| TC-MQTT-31 | UI | Verify UI indicates communication failure | Error indicator with details (reason, timestamp) | P0 | E3 | E3-AC5 |
| TC-MQTT-32 | UI | Verify MQTT broker configuration via Web UI | Admin can add/edit/delete broker profiles | P1 | E3 | E3-AC6 |
| TC-MQTT-33 | UI | Verify "Test Connection" in broker configuration | Connection test executes and shows result | P1 | E3 | E3-S1-AC2 |
| TC-MQTT-34 | UI | Verify credentials masked in UI | Password fields show **** after save | P0 | E3 | E3-S1-AC3 |
| TC-MQTT-35 | UI | Verify protocol filter in device list | Filtering by "MQTT" shows only MQTT devices | P1 | E3 | E3-S6-AC2 |
| TC-MQTT-36 | Coexistence | Monitor MQTT and non-MQTT devices simultaneously | Both protocols function correctly in parallel | P0 | E1 | E1-S7-AC1 |
| TC-MQTT-37 | Coexistence | Verify no degradation of existing RDGC monitoring | SNMP/Modbus polling within 5% of baseline | P0 | E1 | E1-S7-AC1 |
| TC-MQTT-38 | Performance | MQTT engine CPU overhead at max load | < 5% additional CPU on reference hardware | P1 | E1 | E1-S7-AC2 |
| TC-MQTT-39 | Performance | Message processing throughput | ≥ 1000 messages/second on reference hardware | P1 | E2 | E2-NFR |
| TC-MQTT-40 | Performance | UI rendering 200 MQTT devices | Device list renders within 2 seconds | P1 | E3 | E3-NFR |
| TC-MQTT-41 | Browser | Chrome v120+ compatibility | All UI features functional | P0 | E3 | E3-NFR |
| TC-MQTT-42 | Browser | Edge v120+ compatibility | All UI features functional | P0 | E3 | E3-NFR |
| TC-MQTT-43 | Browser | Firefox v115+ compatibility | All UI features functional | P1 | E3 | E3-NFR |
| TC-MQTT-44 | Browser | Safari v17+ compatibility | All UI features functional | P1 | E3 | E3-NFR |

---

## Traceability to Original Test Cases (from Jira)

| Original TC | Maps to |
|---|---|
| TC‑MQTT‑01: MQTT Device Connection | TC-MQTT-01, TC-MQTT-02, TC-MQTT-03 |
| TC‑MQTT‑02: Data Ingestion | TC-MQTT-16, TC-MQTT-17, TC-MQTT-18 |
| TC‑MQTT‑03: UI Connectivity Status | TC-MQTT-27, TC-MQTT-28 |
| TC‑MQTT‑04: Monitoring Parameters Display | TC-MQTT-29, TC-MQTT-30 |
| TC‑MQTT‑05: Communication Failure Handling | TC-MQTT-12, TC-MQTT-15, TC-MQTT-31 |
| TC‑MQTT‑06: Security Validation | TC-MQTT-04, TC-MQTT-05, TC-MQTT-07, TC-MQTT-09 |
| TC‑MQTT‑07: Coexistence with Existing Protocols | TC-MQTT-36, TC-MQTT-37, TC-MQTT-38 |

---

## Test Environment Requirements

| Component | Requirement |
|---|---|
| MQTT Brokers | Mosquitto (open source), HiveMQ (commercial), EMQX |
| TLS Certificates | Valid CA-signed, expired, self-signed, custom CA for testing |
| Test Devices | MQTT simulators publishing JSON telemetry at configurable intervals |
| Reference Hardware | RDGC reference hardware (spec per sizing guide) |
| Existing Workload | 100 SNMP devices + 50 Modbus devices for coexistence testing |
| Browsers | Chrome 120+, Edge 120+, Firefox 115+, Safari 17+ |
| Network Simulation | Tools for simulating broker outage, latency, and packet loss |

# BDCSPM-74932: DAC Analytics Algorithm MQTT Output Data Channel

> **Parent**: BDCSPM-66856 — BLSS Analytics Plugin Platform Foundation (DAC Framework)  
> **Status**: Analyzing  
> **Fix Version**: BLSS 8.2.0  
> **Product**: BLSS (DCPM)

## Title

**DAC Analytics Algorithm MQTT Output Data Channel — SparkplugB-Based Result Ingestion via BLSS Monitoring Infrastructure**

## Description

Enable analytics algorithm plugins to deliver their analysis result data back to the BLSS platform via the built-in MQTT broker, using the standard SparkplugB payload format and BLSS MQTT monitoring templates, so that BLSS can leverage its existing monitoring data pipeline to receive, parse, and process algorithm output without any additional broker deployment or custom data handling.

## Problem Statement

Analytics algorithm plugins produce analysis results that need to be ingested by BLSS for further processing, visualization, and alerting. A standardized, governed output channel is needed so that:

- Algorithm plugins deliver results in a uniform way regardless of algorithm type
- BLSS reuses its existing monitoring data reception and processing mechanisms (no new data pipeline)
- The data format and delivery method are pre-declared at plugin registration time, enabling platform-level validation and governance

## Key Design Principles

### 1. Registration-Driven Declaration

When a new analytics algorithm plugin registers with BLSS, the following metadata is declared as part of the registration contract:

- **Algorithm identity** — algorithm name, version, applicable device model(s)
- **Output data template** — the BLSS standard MQTT monitoring template that defines the schema of the algorithm's output data
- **Data transmission method** — declares MQTT as the delivery protocol (MQTT is the standard method; other methods may be supported in future)
- **Applicable device models** — which device models this algorithm applies to

All declarations are persisted in the BLSS system at registration time, enabling the platform to automatically validate, route, and process data.

### 2. BLSS Built-in MQTT Broker Reuse

- Algorithm plugins connect directly to the **BLSS internal MQTT broker** (already deployed as part of the BLSS platform infrastructure)
- No additional broker deployment, configuration, or maintenance required
- Connection credentials and topic assignments are provisioned by the platform during plugin registration

### 3. SparkplugB Standard Payload Format

- All algorithm output data **MUST** use the **SparkplugB** protocol format
- SparkplugB provides self-describing payloads (with metrics, timestamps, and data types), ensuring interoperability
- Compatible with BLSS's existing data parsing infrastructure

### 4. BLSS Standard MQTT Monitoring Template

- The output data schema is defined by a **BLSS standard MQTT monitoring template**
- The template declares the metrics, data types, and structure that the algorithm will publish
- BLSS receives the data using its existing monitoring template-based processing pipeline — the same mechanism used for device monitoring data

### 5. Existing Monitoring Mechanism Reuse

- BLSS treats algorithm output data as monitoring data from a "virtual device" perspective
- The platform's existing monitoring data reception, parsing, storage, and alerting mechanisms are reused without modification
- No new data processing pipeline is required

## In Scope

- Define the registration schema for algorithm output declarations (algorithm identity, output template, applicable device models, transmission method)
- Implement platform-side validation of output declarations at plugin registration time
- Implement MQTT topic provisioning and credential allocation for registered algorithm plugins
- Implement SparkplugB payload validation for incoming algorithm result messages
- Map algorithm output to the declared BLSS MQTT monitoring template
- Route algorithm output data into the existing BLSS monitoring data pipeline for processing
- Implement per-plugin topic isolation and ACL enforcement
- Implement delivery channel observability (message counts, error rates, delivery status)

## Out of Scope

- Data delivery TO plugins (input channel — separate concern)
- New MQTT broker deployment or configuration (reuse existing)
- Non-MQTT output methods (future scope)
- Algorithm logic or algorithm execution runtime
- Plugin lifecycle management (belongs to Plugin Manager capability)
- Plugin manifest signing and security verification (belongs to Plugin Contract)
- Alerting/visualization rules based on algorithm output (downstream concern)

## Dependencies

| Dependency | Owner | Status | Risk if Delayed |
| --- | --- | --- | --- |
| Plugin Registration Contract (Point 1) | DAC Platform | — | High — registration schema must support output declarations |
| Runtime Plugin Manager (Point 2) | DAC Platform | — | High — lifecycle events trigger MQTT channel provisioning/teardown |
| BLSS internal MQTT broker availability | BLSS Infra | Available | Low — reuses existing internal broker |
| BLSS MQTT Monitoring Template framework | BLSS Monitoring | Available | Medium — output templates must be compatible with existing template engine |

## Data Flow Overview

```
┌─────────────────────┐       SparkplugB         ┌──────────────────────────┐
│  Analytics Plugin   │ ──── MQTT Publish ──────► │  BLSS Internal MQTT      │
│  (Algorithm Output) │    (declared template)    │  Broker                  │
└─────────────────────┘                           └───────────┬──────────────┘
                                                              │
                                                              ▼
                                                  ┌──────────────────────────┐
                                                  │  BLSS Monitoring Data    │
                                                  │  Pipeline (existing)     │
                                                  │  - Template-based parse  │
                                                  │  - Storage               │
                                                  │  - Alerting/Processing   │
                                                  └──────────────────────────┘
```

## Registration-Time Metadata Example (Conceptual)

```json
{
  "algorithm": {
    "id": "thermal-anomaly-detector",
    "version": "1.2.0",
    "applicableDeviceModels": ["UPS-9395", "UPS-93PM"]
  },
  "output": {
    "transmissionMethod": "MQTT",
    "payloadFormat": "SparkplugB",
    "monitoringTemplate": "blss-standard-analytics-output-v1",
    "publishFrequency": "on-event"
  }
}
```

## Acceptance Criteria

### AC-1: Plugin Declares MQTT Output Channel at Registration

**Given** an analytics algorithm plugin submits a registration request to BLSS  
**When** the registration contract includes an output declaration (transmissionMethod=MQTT, payloadFormat=SparkplugB, monitoring template name, applicable device model list)  
**Then** BLSS platform successfully validates and persists the output declaration, returning a registration success confirmation

### AC-2: Output Declaration Validation at Registration

**Given** a plugin submits a registration contract containing an output declaration  
**When** the declared monitoring template does not exist, or the applicable device models are not known to the system, or the payload format is not SparkplugB  
**Then** BLSS rejects the registration and returns a clear validation error indicating the failed field(s) and reason(s)

### AC-3: Automatic MQTT Topic and Credential Provisioning After Registration

**Given** a plugin has been successfully registered with a validated output declaration  
**When** the registration process completes  
**Then** the platform allocates a dedicated MQTT topic for the plugin, generates connection credentials, and configures ACL rules (allowing only that plugin to publish to its dedicated topic)

### AC-4: Algorithm Plugin Publishes SparkplugB Data via MQTT

**Given** a registered algorithm plugin connects to the BLSS built-in MQTT broker using its allocated credentials  
**When** the plugin publishes an analysis result message in SparkplugB format to its dedicated topic  
**Then** BLSS successfully receives the message and it passes SparkplugB payload validation

### AC-5: SparkplugB Payload Format Validation

**Given** BLSS receives a message from a plugin's dedicated topic  
**When** the message payload does not conform to SparkplugB format specification  
**Then** BLSS rejects processing of the message, logs an error (including plugin ID, topic, error details), and increments the plugin's error counter

### AC-6: Data Parsing and Processing via Monitoring Template

**Given** BLSS receives an algorithm output message that passes SparkplugB validation  
**When** the message content matches the BLSS standard MQTT monitoring template declared at the plugin's registration  
**Then** BLSS processes the data through the existing monitoring data pipeline (parse, store), and the data is queryable and displayable in the monitoring interface

### AC-7: Monitoring Template Mismatch Handling

**Given** BLSS receives a message that passes SparkplugB format validation  
**When** the message's metrics structure does not match the registered monitoring template (e.g., missing fields, type mismatch)  
**Then** BLSS logs a template mismatch warning, and the message is routed to an exception queue rather than the normal processing pipeline

### AC-8: Topic Isolation and ACL Enforcement

**Given** multiple algorithm plugins are registered with their own dedicated topics  
**When** Plugin A attempts to publish a message to Plugin B's topic  
**Then** the MQTT broker rejects the publish operation and logs an ACL violation

### AC-9: Delivery Channel Observability

**Given** one or more algorithm plugins are sending data through the MQTT channel  
**When** an administrator views the channel monitoring dashboard  
**Then** per-plugin message counts, error rates, last message timestamps, and delivery status are visible

### AC-10: Channel Cleanup on Plugin Deregistration

**Given** a registered algorithm plugin is deregistered (unregistered) from BLSS  
**When** the deregistration process completes  
**Then** the platform revokes the plugin's MQTT credentials, removes ACL rules, closes topic subscriptions, and subsequent connection attempts from that plugin are rejected

### AC-11: No Additional Broker Deployment Required

**Given** the BLSS system is deployed and running with its built-in MQTT broker  
**When** a new analytics algorithm plugin registers and begins sending data back via MQTT  
**Then** the entire process requires no manual deployment, configuration, or maintenance of any additional MQTT broker instance by the administrator

---

## Key Differences from Current Jira Version

| # | Current Version | Revised Version |
|---|----------------|-----------------|
| 1 | Positioned as data "input" channel to plugins | Clearly defined as algorithm result "output" channel back to BLSS |
| 2 | No payload format specified | SparkplugB explicitly mandated as standard payload format |
| 3 | No monitoring template mentioned | BLSS standard MQTT monitoring template defines data schema |
| 4 | Registration-time declaration not emphasized | Core design is registration-driven |
| 5 | Implies a new proxy/pipeline is needed | Explicitly reuses existing monitoring data reception mechanism, no new pipeline |

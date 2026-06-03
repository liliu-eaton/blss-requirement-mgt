# BDCSPM-70937 — Epic 3: Web UI — MQTT Device Monitoring & Communication Health

## Capability Reference

| Field | Value |
|---|---|
| **Capability** | BDCSPM-70937: RDGC has the ability to monitor devices using MQTT |
| **Parent** | BDCSPM-6550: RDGC monitoring protocol parity with BLSS |
| **Fix Version** | BLSS 8.2.0 (2027-01-15) |
| **Priority** | High |
| **Dependency** | Epic 1 (Protocol Engine), Epic 2 (Data Ingestion) |

---

### EPIC: BDCSPM-70937-E3 — Web UI — MQTT Device Monitoring & Communication Health

**BLSS Product(s)**: Distributed IT Performance Management (RDGC — Web UI)

**Summary**
Provide Web UI visibility for MQTT-monitored devices including device connection status, monitoring parameters with update frequency, and communication health indicators to support operational awareness and troubleshooting.

**Description**

- **Problem statement**: Once MQTT connectivity and data ingestion are implemented (Epics 1 & 2), administrators and operators need a visual interface to see device status, monitor parameters, and troubleshoot communication issues — consistent with the existing UI for SNMP/Modbus devices.
- **Proposed solution**: Extend the existing RDGC Web UI to display MQTT device connection status, monitoring parameters, update frequency, and communication failure indicators. Provide MQTT broker configuration management through the UI.
- **User personas**: IT/OT Administrators, Operations Teams, System Integrators
- **Business value**: Delivers the "single pane of glass" monitoring experience that customers expect; reduces the need for separate MQTT tools and improves operational efficiency.
- **Key workflows**:
  1. Administrator configures MQTT broker and device profiles via Web UI
  2. Operator views MQTT device connectivity status (connected/disconnected/error)
  3. Operator views device monitoring parameters and current values
  4. Operator sees update frequency per device/parameter
  5. Operator identifies communication failures via health indicators
  6. Administrator troubleshoots using communication logs and metrics
- **Cross-product touchpoints**: Uses existing Brightlayer Design System components. Integrates with existing device list and monitoring views.
- **UX/UI references**: Extend existing device monitoring views; follow Brightlayer Design System patterns for status indicators, data tables, and configuration forms.
- **On-prem operational impact**:
  - Install/upgrade impact: UI bundle update; no additional server-side services
  - Sizing impact: Minimal (UI is client-rendered)
  - Network impact: No additional external ports
  - Backup impact: UI configuration preferences stored in existing user preferences DB

**Non-Functional Requirements (NFRs)**

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Performance | UI status update latency | Connection status changes reflected within 10 seconds | P0 |
| Performance | Device list rendering | ≤ 2s to render 200 MQTT devices with status indicators | P1 |
| Usability | UI accessibility | WCAG 2.1 Level AA compliant | P1 |
| Usability | Responsive design | Usable on 1280px+ screen widths | P1 |
| Browser Compatibility | Supported browsers | Chrome v120+, Edge v120+, Firefox v115+, Safari v17+ | P0 |
| Reliability | UI degradation on API failure | Show last known state with "stale data" indicator; no crash | P1 |
| Security | Configuration access | MQTT broker credentials not visible in UI after save | P0 |

**Acceptance Criteria (Epic-level)**

- [ ] AC1: Device connection status (connected/disconnected/error) is visible in the Web UI for MQTT devices
- [ ] AC2: Status updates reflect connect and disconnect events within 10 seconds
- [ ] AC3: Default monitoring parameters and their current values are visible per MQTT device
- [ ] AC4: Polling/update frequency is displayed per device
- [ ] AC5: Communication failures are clearly indicated with error details for troubleshooting
- [ ] AC6: MQTT broker configuration can be managed (add/edit/delete) through the Web UI
- [ ] AC7: MQTT devices are visually distinguishable from SNMP/Modbus devices in the device list

**Scope**

- **In scope**:
  - MQTT device connection status display (connected, disconnected, reconnecting, error)
  - Monitoring parameter values display with timestamps
  - Update frequency display per device/parameter
  - Communication health indicators (last-seen, error count, reconnect count)
  - MQTT broker configuration management UI (add, edit, delete, test connection)
  - MQTT device profile/mapping configuration UI
  - Integration with existing device list and filter views
  - Protocol-type filter for MQTT devices
  - Communication failure details view (error log, last error, timestamp)

- **Out of scope**:
  - MQTT message payload raw view/debug tool
  - MQTT topic browser/explorer
  - Device control commands via MQTT
  - Custom dashboard widgets for MQTT-specific data
  - Mobile-responsive design (< 1280px)

- **Assumptions**:
  1. ⚠️ ASSUMPTION: Existing device list UI can accommodate a new "Protocol" column/filter without major refactoring
  2. ⚠️ ASSUMPTION: Brightlayer Design System has suitable status indicator components (badge, icon, color coding)
  3. ⚠️ ASSUMPTION: Internal APIs from Epics 1 & 2 provide all data needed for UI rendering
  4. ⚠️ ASSUMPTION: Existing role-based access control (RBAC) applies — admin for configuration, operator for viewing

**Dependencies & Risks**

| Item | Type | BLSS Product | Owner | Mitigation | Status |
|---|---|---|---|---|---|
| Epic 1 (Connection State API — E1-S6) | Dep | Distributed IT | Dev team | UI cannot show status without internal API | Open |
| Epic 2 (Device Model & Parameters — E2-S4, E2-S5) | Dep | Distributed IT | Dev team | UI cannot show parameters without data layer | Open |
| Brightlayer Design System components | Dep | Cross-product | UX team | Confirm available components for status indicators | Open |
| RBAC for MQTT configuration pages | Dep | Distributed IT | Auth team | Confirm existing roles support new config pages | Open |
| UX design mockups | Risk | Distributed IT | UX team | No mockups yet — design sprint needed before UI development | Open |

**Operational Deliverables** (required for on-prem)

- [ ] Installation/upgrade runbook updated (UI bundle deployment)
- [ ] Release notes entry drafted
- [ ] Lifecycle Services training/enablement updated (MQTT configuration workflows)
- [ ] Online help / user guide updated (MQTT device setup, status interpretation, troubleshooting)
- [ ] Admin guide updated (broker configuration, device profile setup)

---

## Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| E3-S1 | MQTT Broker Configuration UI | 8 | Sprint 4 |
| E3-S2 | MQTT Device Connection Status Display | 5 | Sprint 4 |
| E3-S3 | MQTT Device Monitoring Parameters View | 8 | Sprint 5 |
| E3-S4 | Update Frequency Display | 3 | Sprint 5 |
| E3-S5 | Communication Health & Failure Indicators | 5 | Sprint 5 |
| E3-S6 | Device List Integration & Protocol Filter | 5 | Sprint 6 |
| E3-S7 | MQTT Device Profile Configuration UI | 8 | Sprint 6 |

---

## Story Details

---

### STORY: E3-S1 — MQTT Broker Configuration UI

**Parent Epic**: BDCSPM-70937-E3 — Web UI — MQTT Device Monitoring & Communication Health
**BLSS Product**: Distributed IT Performance Management (RDGC — Web UI)

**Description**
As an IT/OT Administrator,
I want to configure MQTT broker connections through the Web UI,
So that I can set up MQTT monitoring without editing configuration files manually.

**Context & notes**:
- CRUD operations for MQTT broker profiles
- Form fields: broker name, host, port, protocol version, TLS settings, auth type, credentials
- "Test Connection" button to validate settings before saving
- Credentials masked after save (display as ****)
- Uses existing RDGC configuration API; RBAC restricts to admin role
- Brightlayer Design System form components

**Acceptance Criteria**

AC1:
  Given: An administrator navigates to the MQTT configuration page
  When: They fill in broker connection details and click "Save"
  Then: The broker profile is created and appears in the broker list

AC2:
  Given: A broker profile form is filled with valid host and credentials
  When: The administrator clicks "Test Connection"
  Then: A connection test is executed and success/failure result is displayed within 15 seconds

AC3:
  Given: An existing broker profile is displayed
  When: The administrator views the configuration
  Then: Password/credential fields show masked values (****), not plaintext

AC4:
  Given: A broker profile exists
  When: The administrator deletes the broker profile
  Then: A confirmation dialog is shown; on confirm, the profile is removed and associated subscriptions stop

AC5:
  Given: A non-admin user is logged in
  When: They navigate to the MQTT configuration page
  Then: Configuration editing is disabled; read-only view of broker names/status is shown

**Tasks**
- [ ] Task 1: Design UI wireframe for broker configuration page (list + form view)
- [ ] Task 2: Implement broker list view (name, host, port, status, actions)
- [ ] Task 3: Implement broker add/edit form (all connection parameters)
- [ ] Task 4: Implement "Test Connection" button with async result display
- [ ] Task 5: Implement credential masking (display) and secure storage (API call)
- [ ] Task 6: Implement delete confirmation dialog
- [ ] Task 7: Implement RBAC restrictions (admin vs. read-only)
- [ ] Task 8: Write UI component tests
- [ ] Task 9: Write E2E tests for broker CRUD operations

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached
- [ ] API endpoints for broker management documented (from backend team)
- [ ] RBAC roles confirmed (admin can configure, operator can view)
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E3-S2 — MQTT Device Connection Status Display

**Parent Epic**: BDCSPM-70937-E3 — Web UI — MQTT Device Monitoring & Communication Health
**BLSS Product**: Distributed IT Performance Management (RDGC — Web UI)

**Description**
As an Operations Team member,
I want to see the real-time connection status of MQTT-monitored devices,
So that I can quickly identify which devices are online, offline, or experiencing issues.

**Context & notes**:
- Status states: Connected (green), Disconnected (red), Reconnecting (yellow), Error (orange)
- Status badge/icon in device list and device detail view
- Real-time updates via polling or WebSocket (≤ 10s refresh)
- Last state change timestamp visible on hover or in detail view
- Consumes internal Connection State API (E1-S6)

**Acceptance Criteria**

AC1:
  Given: An MQTT device is actively connected and reporting data
  When: The operator views the device list
  Then: The device shows a "Connected" status with a green indicator

AC2:
  Given: An MQTT device loses connection to the broker
  When: The disconnect is detected by RDGC
  Then: The UI updates the device status to "Disconnected" (red) within 10 seconds

AC3:
  Given: RDGC is attempting to reconnect to an MQTT broker
  When: Reconnection is in progress
  Then: Affected devices show "Reconnecting" status with a yellow indicator

AC4:
  Given: A device status indicator is displayed
  When: The operator hovers over or clicks the status badge
  Then: A tooltip or panel shows the last state change timestamp and duration in current state

**Tasks**
- [ ] Task 1: Implement status badge component (Connected/Disconnected/Reconnecting/Error)
- [ ] Task 2: Integrate status badge into device list view
- [ ] Task 3: Implement device detail view status panel
- [ ] Task 4: Implement real-time status polling (configurable interval, default 10s)
- [ ] Task 5: Implement tooltip/popover with state change timestamp
- [ ] Task 6: Write UI component tests for status states
- [ ] Task 7: Write E2E tests for status transition display

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (status badge styles, colors, placement)
- [ ] Internal Connection State API contract documented
- [ ] Status color coding approved by UX team
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E3-S3 — MQTT Device Monitoring Parameters View

**Parent Epic**: BDCSPM-70937-E3 — Web UI — MQTT Device Monitoring & Communication Health
**BLSS Product**: Distributed IT Performance Management (RDGC — Web UI)

**Description**
As an Operations Team member,
I want to view the monitoring parameters and current values of MQTT devices,
So that I can assess device health and operating conditions.

**Context & notes**:
- Parameter table: parameter name, current value, unit, timestamp, quality/status
- Sortable and filterable parameter list
- Consistent with existing SNMP/Modbus device parameter views
- Support for numeric, string, and boolean parameter display
- Timestamp shows "last updated" per parameter

**Acceptance Criteria**

AC1:
  Given: An MQTT device has multiple monitoring parameters (temperature, humidity, voltage)
  When: The operator opens the device detail view
  Then: All parameters are displayed in a table with name, value, unit, and last-updated timestamp

AC2:
  Given: An MQTT device is reporting data
  When: A new telemetry message arrives
  Then: The parameter values update in the UI without page reload (within 10 seconds)

AC3:
  Given: A parameter has not been updated within the configured stale-data timeout
  When: The operator views the parameter
  Then: A "stale data" indicator is shown next to the parameter value

AC4:
  Given: A device has 50+ monitoring parameters
  When: The operator views the parameters
  Then: The list is scrollable, sortable by name/value/timestamp, and filterable by name

**Tasks**
- [ ] Task 1: Implement parameter table component (columns: name, value, unit, timestamp, quality)
- [ ] Task 2: Implement real-time value refresh (polling or subscription)
- [ ] Task 3: Implement stale-data indicator logic
- [ ] Task 4: Implement sorting and filtering for parameter table
- [ ] Task 5: Integrate with device detail page
- [ ] Task 6: Write UI component tests
- [ ] Task 7: Write E2E tests for parameter display and refresh

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (parameter table layout)
- [ ] API endpoint for device parameters documented
- [ ] Stale-data timeout threshold confirmed
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E3-S4 — Update Frequency Display

**Parent Epic**: BDCSPM-70937-E3 — Web UI — MQTT Device Monitoring & Communication Health
**BLSS Product**: Distributed IT Performance Management (RDGC — Web UI)

**Description**
As an Operations Team member,
I want to see the polling/update frequency for each MQTT device,
So that I understand how often device data is refreshed and can identify devices with irregular reporting.

**Context & notes**:
- Display calculated average update frequency (from message arrival timestamps)
- Show per-device and optionally per-parameter frequency
- Format: "Every 30s", "Every 5 min", "Irregular (avg 45s)"
- Highlight devices with irregular or significantly degraded update rates
- Visible in device list (summary) and device detail (per-parameter)

**Acceptance Criteria**

AC1:
  Given: An MQTT device sends telemetry every 60 seconds
  When: The operator views the device in the device list
  Then: The update frequency column shows "Every 60s"

AC2:
  Given: An MQTT device has irregular message intervals (ranging 10s to 120s)
  When: The operator views the device
  Then: The frequency shows "Irregular (avg Xs)" with a warning indicator

AC3:
  Given: An MQTT device has not sent data for longer than 2x its average interval
  When: The operator views the device
  Then: A "Delayed" indicator is shown alongside the expected frequency

**Tasks**
- [ ] Task 1: Implement update frequency calculation display component
- [ ] Task 2: Implement frequency formatting logic (human-readable intervals)
- [ ] Task 3: Implement irregular frequency detection and warning indicator
- [ ] Task 4: Implement "Delayed" indicator when data is overdue
- [ ] Task 5: Integrate into device list and device detail views
- [ ] Task 6: Write unit tests for frequency formatting
- [ ] Task 7: Write UI component tests

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached
- [ ] Frequency calculation algorithm reviewed
- [ ] "Irregular" threshold defined (e.g., coefficient of variation > 50%)
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E3-S5 — Communication Health & Failure Indicators

**Parent Epic**: BDCSPM-70937-E3 — Web UI — MQTT Device Monitoring & Communication Health
**BLSS Product**: Distributed IT Performance Management (RDGC — Web UI)

**Description**
As an Operations Team member,
I want clear communication health indicators and failure details for MQTT devices,
So that I can quickly troubleshoot connectivity issues.

**Context & notes**:
- Health indicators: broker connection health, per-device last-seen, error count
- Failure details: last error message, timestamp, reconnect attempt count
- Communication log view: recent connection events (connect, disconnect, error, reconnect)
- Visual cues: color-coded health bar, error badges, attention icons
- Actionable information: what's wrong and what to check

**Acceptance Criteria**

AC1:
  Given: An MQTT broker connection has failed
  When: The operator views the MQTT status panel
  Then: The failure is clearly indicated with the broker name, error reason, and timestamp

AC2:
  Given: A device communication failure has occurred
  When: The operator clicks on the device's health indicator
  Then: A detail panel shows: last error, error count (last 24h), reconnect attempts, last successful communication

AC3:
  Given: Multiple devices connected to the same broker are affected by a broker outage
  When: The operator views the device list
  Then: All affected devices show "Disconnected" with a visual link to the broker issue

AC4:
  Given: Communication is restored after a failure
  When: The connection is re-established
  Then: The status indicator returns to "Connected" (green) and the failure is moved to the communication history log

**Tasks**
- [ ] Task 1: Implement communication health summary panel (broker-level health)
- [ ] Task 2: Implement per-device health detail (last error, error count, reconnect attempts)
- [ ] Task 3: Implement communication event log view (recent events list)
- [ ] Task 4: Implement broker-to-device failure correlation display
- [ ] Task 5: Implement health indicator color coding and icons
- [ ] Task 6: Write UI component tests
- [ ] Task 7: Write E2E tests for failure scenario display

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (health panel, error detail view)
- [ ] Internal metrics API (E1-S6, E2-S7) contract confirmed
- [ ] Communication event log retention policy confirmed
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E3-S6 — Device List Integration & Protocol Filter

**Parent Epic**: BDCSPM-70937-E3 — Web UI — MQTT Device Monitoring & Communication Health
**BLSS Product**: Distributed IT Performance Management (RDGC — Web UI)

**Description**
As an Operations Team member,
I want MQTT devices to appear in the existing device list with a protocol filter,
So that I can view all monitored devices together or filter by protocol type.

**Context & notes**:
- MQTT devices appear in the existing device list alongside SNMP and Modbus devices
- Protocol type column/badge: "MQTT", "SNMP", "Modbus", etc.
- Filter: filter device list by protocol type
- MQTT devices show same standard columns (name, location, status, last-updated)
- Additional column or badge indicating MQTT protocol

**Acceptance Criteria**

AC1:
  Given: MQTT devices exist in the RDGC inventory
  When: The operator views the device list without filters
  Then: MQTT devices appear alongside SNMP/Modbus devices in the unified list

AC2:
  Given: The device list contains mixed protocol devices
  When: The operator applies the protocol filter "MQTT"
  Then: Only MQTT-monitored devices are displayed

AC3:
  Given: MQTT devices are displayed in the device list
  When: The operator views the list
  Then: Each device shows a protocol indicator badge ("MQTT") distinguishing it from other protocols

AC4:
  Given: Existing SNMP/Modbus device list functionality
  When: MQTT devices are added
  Then: All existing sorting, searching, and pagination functions work correctly with mixed protocol devices

**Tasks**
- [ ] Task 1: Add protocol type column/badge to device list
- [ ] Task 2: Implement protocol type filter option
- [ ] Task 3: Ensure MQTT devices populate standard columns (name, location, status)
- [ ] Task 4: Validate existing sorting/search/pagination with mixed protocols
- [ ] Task 5: Write UI component tests
- [ ] Task 6: Write E2E tests for filtering and mixed-protocol display

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (protocol badge style, filter placement)
- [ ] Device list API supports protocol type field and filter parameter
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: E3-S7 — MQTT Device Profile Configuration UI

**Parent Epic**: BDCSPM-70937-E3 — Web UI — MQTT Device Monitoring & Communication Health
**BLSS Product**: Distributed IT Performance Management (RDGC — Web UI)

**Description**
As an IT/OT Administrator,
I want to configure MQTT device profiles (topic mappings, payload parsing rules) through the Web UI,
So that I can onboard new MQTT device types without editing configuration files.

**Context & notes**:
- Device profile: defines how to map MQTT topics to devices and parse payloads
- Form fields: profile name, topic pattern, device ID extraction rule, payload format, field mappings
- Support creating, editing, cloning, and deleting profiles
- Validation: test profile against sample message
- Profiles are reusable across multiple devices of the same type

**Acceptance Criteria**

AC1:
  Given: An administrator navigates to the MQTT device profiles page
  When: They create a new profile with topic pattern, device ID extraction, and field mappings
  Then: The profile is saved and available for assignment to MQTT broker configurations

AC2:
  Given: A device profile is being configured
  When: The administrator enters a sample MQTT message (topic + JSON payload)
  Then: The system previews the parsed result (extracted device ID, parameter values)

AC3:
  Given: An existing device profile is in use by active devices
  When: The administrator edits the profile
  Then: A warning indicates the change will affect X active devices; changes apply on save

AC4:
  Given: A device profile exists
  When: The administrator clones the profile
  Then: A copy is created with "(Copy)" suffix for modification without affecting the original

**Tasks**
- [ ] Task 1: Implement device profile list view (name, topic pattern, assigned devices count)
- [ ] Task 2: Implement profile add/edit form (topic pattern, extraction rules, field mappings)
- [ ] Task 3: Implement "Test with sample message" preview functionality
- [ ] Task 4: Implement profile clone and delete operations
- [ ] Task 5: Implement impact warning for editing active profiles
- [ ] Task 6: Implement validation rules for topic patterns and JSONPath expressions
- [ ] Task 7: Write UI component tests
- [ ] Task 8: Write E2E tests for profile CRUD and testing

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (profile form, test preview)
- [ ] API endpoints for profile management documented
- [ ] Sample MQTT payloads from target devices available for testing
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

## Validation Checklist

- [x] All AC are testable
- [x] NFRs are quantified (10s status update, 2s render 200 devices, WCAG 2.1 AA)
- [x] BLSS product explicitly identified (Distributed IT Performance Management — RDGC Web UI)
- [x] Scope boundaries are explicit
- [x] Cross-product dependencies identified (Brightlayer Design System)
- [x] On-prem operational impact assessed (UI bundle update only)
- [x] Dependencies identified (Epics 1 & 2, UX mockups, RBAC)
- [x] No invented features, UI labels, device models, or protocol details
- [x] Terminology matches BLSS product glossary
- [x] Placeholders marked with [TBD] where applicable
- [x] Operational deliverables included (user guide, admin guide, training)

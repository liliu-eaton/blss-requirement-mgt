# BDCSPM-73568 — ServerAdmin Self-Service Configuration for Network, Identity, and Time Services

## EPIC 2: ServerAdmin Network, Session Timeout, and Time Services Configuration

**BLSS Product(s)**: Shared platform services (ServerAdmin)
**Summary** (1-2 sentences)
Provide administrators with UI-based management of network settings (IP address, hostname), user session auto-logout timeout, and NTP time services (client and server mode) directly within the ServerAdmin page, eliminating CLI dependency for these critical system configurations.

---

### Description

- **Problem statement**: Changing the server IP address, hostname, session timeout, and NTP time configuration currently requires Linux CLI access (`nmcli`, `hostnamectl`, `chronyc`, `vdctools`). Customers in industrial OT environments often lack Linux expertise, resulting in misconfiguration, extended deployment timelines, and increased support ticket volume. In isolated/air-gapped networks, NTP misconfiguration leads to inconsistent timestamps across all BLSS products, making alarm and event correlation unreliable.

- **Proposed solution**: Add new configuration sections directly within the ServerAdmin page:
  1. **Network Configuration**: IP address and hostname changes with validation and safety mechanisms (auto-revert timer for IP changes, TLS certificate impact warning for hostname changes)
  2. **Session Auto-Logout Timeout**: Configurable inactivity timeout for all user sessions (wrapping existing `vdctools` functionality)
  3. **Time Services (NTP)**: Configure upstream NTP source, enable/disable the BLoP server as an NTP server for connected devices, and view synchronization status

  All configurations include validation-before-apply, audit logging, and admin-only RBAC.

- **User personas**: Commissioning Engineer, System Integrator, On-Prem IT / System Admin, Eaton Field Service Engineer

- **Business value**:
  - Eliminates CLI dependency for network and time configuration
  - IP auto-revert safety prevents accidental server lockout from bad network config
  - NTP server mode enables time sync in isolated networks (critical for event correlation)
  - Session timeout self-service reduces support tickets for security policy compliance
  - Accelerates commissioning (network + NTP + session config in one UI session)

- **Key workflows**:
  1. Admin navigates to ServerAdmin → Network Configuration section
  2. Admin changes IP/hostname with validation; confirms in modal with impact warning
  3. For IP changes: auto-revert countdown starts; admin confirms at new IP to persist
  4. Admin sets session auto-logout timeout from dropdown or custom value
  5. Admin configures NTP upstream source; tests reachability; applies
  6. Admin enables NTP server mode for device time sync; reviews connected clients
  7. All changes audit-logged

- **Cross-product touchpoints**: Network changes (IP/hostname) affect connectivity to all BLSS products. Session timeout applies to all product login sessions. NTP configuration affects timestamp accuracy for alarms, events, and audit logs across DCPM, EPMS Standard, Distributed IT, and Brightlayer Power.

- **UX/UI references**: See prototype `Requirements/BDCSPM-73568/prototype-serveradmin-self-service-config.html` — "Network" and "Time Services" tabs. `[TBD — final UX mockups]`

- **On-prem operational impact**:
  - Install/upgrade impact: New configuration sections on ServerAdmin page. Upgrade must detect and surface existing CLI-configured network/NTP settings in the UI.
  - Sizing impact: Minimal — NTP server mode serves responses to local device subnet only.
  - Network impact: NTP server mode opens UDP port 123 for inbound NTP requests (must be documented in firewall guide). IP change inherently disrupts connectivity temporarily.
  - Backup impact: Network/NTP/session configuration included in backup scope.

---

### Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Performance | Configuration save response time | < 2s at P95 on reference hardware (Small tier) | P1 |
| Performance | NTP source reachability test response time | < 5s (depends on network) | P2 |
| Reliability | IP change auto-revert if connectivity lost | Revert within 60s timeout `[TBD — confirm]` | P0 |
| Reliability | Configuration validation prevents invalid settings from being applied | 100% of submissions validated | P0 |
| Security | All configuration changes audit-logged (user, timestamp, old value, new value) | 100% of changes | P0 |
| Security | Configuration pages accessible only to admin role | RBAC enforced at UI and API layer | P0 |
| Usability | All configuration completable without CLI access | Zero CLI requirement | P0 |
| Usability | Inline validation with clear error messages | Per-field validation | P1 |
| Usability | Confirmation dialogs for high-impact changes (IP, hostname) with impact summary | All destructive actions | P0 |
| Compatibility | Browser support: Chrome v120+, Edge v120+, Firefox v115+, Safari v17+ | All listed browsers | P1 |
| Compatibility | OS support: RHEL 9, RHEL 10 | All supported OS versions | P0 |
| Availability | ServerAdmin remains accessible during config updates (except IP change) | No unplanned downtime | P1 |
| Recoverability | IP auto-revert restores previous configuration on timeout | Automatic, no user action needed | P0 |

---

### Acceptance Criteria (Epic-level)

- [ ] AC1: An admin can change the server IP address via ServerAdmin UI; system validates, warns about connectivity impact, and applies with 60-second auto-revert if confirmation is not received.
- [ ] AC2: An admin can change the server hostname via ServerAdmin UI; system warns about TLS certificate and service restart impact before applying.
- [ ] AC3: An admin can set user session inactivity auto-logout time via ServerAdmin UI; new timeout applies to all subsequent sessions.
- [ ] AC4: An admin can configure NTP upstream source (IP/hostname) via ServerAdmin UI; server synchronizes time with configured source.
- [ ] AC5: An admin can enable/disable the BLoP server as NTP server for connected devices; when enabled, devices can sync time from the server.
- [ ] AC6: An admin can view NTP synchronization status (sync state, offset, stratum, last sync time, connected clients) in the UI.
- [ ] AC7: All configuration changes are audit-logged with user identity, timestamp, old value, and new value.
- [ ] AC8: All configuration sections enforce admin-only RBAC.
- [ ] AC9: No configuration workflow requires CLI/SSH access.

---

### Scope

- **In scope**:
  - ServerAdmin → Network section: IP address configuration, hostname configuration
  - IP change safety: validation, confirmation modal with impact warning, auto-revert countdown timer
  - Hostname change: validation (RFC 1123), TLS certificate warning, service restart notification
  - ServerAdmin → Session Timeout section: inactivity timeout configuration (preset values + custom)
  - ServerAdmin → Time Services section: NTP upstream source configuration, reachability test, manual "Sync Now"
  - NTP server mode: enable/disable toggle, subnet allowlist, connected clients display
  - NTP status display: sync state, offset, stratum, last sync time
  - Audit logging for all changes
  - Admin-only RBAC on all configuration sections

- **Out of scope**:
  - Advanced network configuration (routing tables, VLANs, firewall rules, DNS server, multi-NIC/bonding)
  - IPv6 support `[TBD — confirm if needed in future]`
  - Multi-node NTP synchronization orchestration
  - Certificate re-issuance automation after hostname change (manual guidance only)
  - SAML/LDAP configuration (covered in Epic 1)

- **Assumptions**:
  1. ⚠️ ASSUMPTION: NTP daemon is `chrony` (standard on RHEL 9/10)
  2. ⚠️ ASSUMPTION: IP change auto-revert timeout is 60 seconds `[TBD — confirm with architecture]`
  3. ⚠️ ASSUMPTION: Session timeout wraps existing `vdctools` functionality
  4. ⚠️ ASSUMPTION: Server has single NIC (multi-NIC/bonding out of scope for v1)
  5. ⚠️ ASSUMPTION: IPv4 only for v1
  6. ⚠️ ASSUMPTION: OS network configuration via NetworkManager/nmcli API on RHEL 9/10
  7. ⚠️ ASSUMPTION: Hostname change requires manual TLS certificate re-issuance (automated cert handling is future scope)
  8. ⚠️ ASSUMPTION: NTP server allows all local subnets by default; configurable subnet allowlist is optional

---

### Dependencies & Risks

| Item | Type (Dep/Risk) | Owner | Mitigation | Status |
|---|---|---|---|---|
| OS NetworkManager/nmcli API stability on RHEL 9/10 | Dep | Platform Engineering | Validate API across supported OS versions | `[TBD]` |
| chrony NTP daemon configuration API | Dep | Platform Engineering | Validate chrony conf management approach | `[TBD]` |
| vdctools session timeout interface | Dep | Platform Engineering | Document vdctools CLI for session timeout | `[TBD]` |
| firewalld for NTP port management | Dep | Platform Engineering | Validate firewalld integration for UDP 123 | `[TBD]` |
| RBAC framework (admin-only page access) | Dep | Platform Engineering | Confirm existing RBAC supports section-level access | `[TBD]` |
| IP change causes admin session/connectivity loss | Risk | Platform Engineering | Auto-revert timer (60s); pre-change warning modal | Open |
| Hostname change breaks TLS certificates | Risk | Platform Engineering | Warn user; provide certificate re-issuance guidance | Open |
| NTP misconfiguration causes timestamp drift across all products | Risk | Platform Engineering | Validate NTP source reachability before applying; drift warning at >1000ms | Open |
| NTP server mode amplification attack | Risk | Platform Engineering | Rate limiting; subnet allowlist; document security implications | Open |

---

### Operational Deliverables (required for on-prem)

- [ ] Installation/upgrade runbook updated (new ServerAdmin config sections, preserve CLI settings)
- [ ] Sizing guide updated (NTP server mode: document UDP 123 traffic)
- [ ] Firewall/port documentation updated (NTP server: UDP 123 inbound from device subnets)
- [ ] Backup/restore procedure updated (include network, session, NTP config)
- [ ] Release notes entry drafted
- [ ] Lifecycle Management Services training/enablement updated
- [ ] Admin user guide: Network configuration (IP + hostname) with safety mechanisms
- [ ] Admin user guide: Session timeout configuration
- [ ] Admin user guide: NTP client and server configuration

---

### Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| S1 | Configure Server IP Address via ServerAdmin UI | `[TBD]` | `[TBD]` |
| S2 | Configure Server Hostname via ServerAdmin UI | `[TBD]` | `[TBD]` |
| S3 | IP/Hostname Change Safety Mechanisms (validation, auto-revert, warnings) | `[TBD]` | `[TBD]` |
| S4 | Configure User Session Auto-Logout Timeout via ServerAdmin UI | `[TBD]` | `[TBD]` |
| S5 | Configure NTP Upstream Source via ServerAdmin UI | `[TBD]` | `[TBD]` |
| S6 | Enable/Disable BLoP Server as NTP Server for Connected Devices | `[TBD]` | `[TBD]` |
| S7 | NTP Synchronization Status Display & Manual Sync | `[TBD]` | `[TBD]` |
| S8 | Audit Logging & RBAC for ServerAdmin Configuration | `[TBD]` | `[TBD]` |

---

---

## STORY: S1 — Configure Server IP Address via ServerAdmin UI

**Parent Epic**: Epic 2 — ServerAdmin Network, Session Timeout, and Time Services Configuration
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As a Commissioning Engineer / System Integrator,
I want to change the BLSS Application Server IP address through the ServerAdmin web UI,
So that I can reconfigure the server's network identity without requiring Linux CLI access.

**Context & notes**:
- Highest-risk configuration change — applying a new IP disrupts the admin's current browser session
- Must interact with OS-level network configuration (NetworkManager / nmcli on RHEL 9/10)
- IPv4 only for v1 `[TBD — confirm IPv6 requirement]`
- Single NIC assumption `[TBD — confirm]`
- Safety mechanism (auto-revert) handled in Story S3

**Acceptance Criteria**

AC1:
  Given: Admin is logged into ServerAdmin with admin role
  When: Admin navigates to Network Configuration section
  Then: Current server IP address, subnet mask, gateway, and interface are displayed (read from OS)

AC2:
  Given: Admin is on Network Configuration section
  When: Admin enters a valid new IP address (with subnet mask and gateway) and clicks Apply
  Then: System displays confirmation modal showing current IP, new IP, and warning about connectivity loss and auto-revert behavior

AC3:
  Given: Admin has confirmed IP change in modal
  When: System applies the new IP address
  Then: OS network configuration is updated; ServerAdmin becomes accessible at the new IP

AC4 (negative):
  Given: Admin enters an invalid IP address (malformed, reserved range 0.0.0.0, 255.255.255.255)
  When: Admin attempts to apply
  Then: Inline validation error displayed; change is not submitted

AC5:
  Given: IP change has been applied and confirmed
  When: Change completes successfully
  Then: Audit log records: user, timestamp, old IP, new IP, result=success

**Tasks**
- [ ] Task 1: <Backend> Create API endpoint `POST /api/serveradmin/network/ip` — accepts new IP, subnet, gateway; validates; applies via nmcli
- [ ] Task 2: <Backend> Implement OS-level IP configuration wrapper (NetworkManager D-Bus API / nmcli)
- [ ] Task 3: <Backend> Implement current network info reader `GET /api/serveradmin/network/status`
- [ ] Task 4: <Frontend> Implement Network Configuration section: current info display, IP/subnet/gateway form
- [ ] Task 5: <Frontend> Implement inline validation (IPv4 format, reserved range exclusion)
- [ ] Task 6: <Test> Integration tests: valid IP change, invalid IP rejection
- [ ] Task 7: <Documentation> Update admin guide with IP change workflow

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] UX mockups attached
- [ ] NetworkManager API approach confirmed
- [ ] Single NIC vs. multi-NIC scope confirmed
- [ ] IPv4-only scope confirmed
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

## STORY: S2 — Configure Server Hostname via ServerAdmin UI

**Parent Epic**: Epic 2 — ServerAdmin Network, Session Timeout, and Time Services Configuration
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As a System Integrator / IT Admin,
I want to change the BLSS Application Server hostname through the ServerAdmin web UI,
So that I can align the server naming with customer network standards without CLI access.

**Context & notes**:
- Hostname change affects: TLS certificates (if hostname-bound), internal service references, log entries
- Must use `hostnamectl` or equivalent on RHEL 9/10
- DNS registration is out of scope (customer responsibility)

**Acceptance Criteria**

AC1:
  Given: Admin is logged into ServerAdmin with admin role
  When: Admin navigates to Network Configuration section
  Then: Current hostname is displayed

AC2:
  Given: Admin enters a valid new hostname (RFC 1123 compliant)
  When: Admin clicks Apply
  Then: Confirmation modal warns about impacts (TLS cert invalidation, service restarts)

AC3:
  Given: Admin confirms hostname change
  When: System applies the change
  Then: OS hostname updated; affected services restarted; ServerAdmin reflects new hostname

AC4 (negative):
  Given: Admin enters an invalid hostname (special chars, >63 chars, empty)
  When: Admin attempts to apply
  Then: Inline validation error shown; change not submitted

AC5:
  Given: Hostname change is applied
  When: Change completes
  Then: Audit log records: user, timestamp, old hostname, new hostname, result

**Tasks**
- [ ] Task 1: <Backend> Create API endpoint `POST /api/serveradmin/network/hostname` — validates RFC 1123 and applies via hostnamectl
- [ ] Task 2: <Backend> Implement service restart orchestration for hostname-dependent services
- [ ] Task 3: <Frontend> Add hostname field to Network Configuration section with confirmation modal
- [ ] Task 4: <Frontend> Inline validation for RFC 1123 rules
- [ ] Task 5: <Test> Integration tests: valid change, invalid rejection, service restart verification
- [ ] Task 6: <Documentation> Document hostname change workflow with TLS re-issuance guidance

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] UX mockups attached
- [ ] List of hostname-dependent services confirmed
- [ ] TLS certificate impact documented
- [ ] Estimated by team

---

## STORY: S3 — IP/Hostname Change Safety Mechanisms

**Parent Epic**: Epic 2 — ServerAdmin Network, Session Timeout, and Time Services Configuration
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As an On-Prem IT / System Admin,
I want safety mechanisms that automatically revert IP changes if connectivity is lost,
So that I cannot accidentally lock myself out of the server by applying a bad network configuration.

**Context & notes**:
- Auto-revert timer pattern (similar to network switch "commit confirmed")
- After IP change: admin must confirm at new IP within timeout; otherwise auto-revert
- Timeout: 60 seconds `[TBD — confirm with architecture]`
- Hostname changes: no auto-revert, but enhanced warning with impact list

**Acceptance Criteria**

AC1:
  Given: Admin has initiated an IP address change
  When: System applies the new IP
  Then: Countdown timer starts (60s); system prompts admin to confirm at the new IP

AC2:
  Given: Auto-revert timer is running
  When: Admin accesses ServerAdmin at new IP and clicks "Confirm Change"
  Then: Timer cancelled; new IP permanently applied; audit log updated

AC3:
  Given: Auto-revert timer is running
  When: Timer expires without admin confirmation
  Then: System automatically reverts to previous IP; audit log records revert event

AC4:
  Given: IP or hostname change confirmation modal is displayed
  When: Admin reviews the modal
  Then: Modal shows: current value → new value, list of impacts, auto-revert behavior (IP), TLS impact (hostname)

**Tasks**
- [ ] Task 1: <Backend> Implement auto-revert timer service (persist state to survive process restart)
- [ ] Task 2: <Backend> Create confirmation endpoint `POST /api/serveradmin/network/ip/confirm`
- [ ] Task 3: <Backend> Implement automatic revert logic (restore previous nmcli config on timer expiry)
- [ ] Task 4: <Frontend> Implement countdown bar with timer display and "Confirm Change" button
- [ ] Task 5: <Frontend> Implement enhanced confirmation modals with change summary and impact list
- [ ] Task 6: <Test> Integration tests: timer expiry → revert, confirm → persist, modal content
- [ ] Task 7: <Test> Simulate connectivity loss scenario

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] Auto-revert timeout value confirmed (60s)
- [ ] Timer persistence mechanism agreed (systemd timer, DB, file)
- [ ] Estimated by team

---

## STORY: S4 — Configure User Session Auto-Logout Timeout via ServerAdmin UI

**Parent Epic**: Epic 2 — ServerAdmin Network, Session Timeout, and Time Services Configuration
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As an Administrator,
I want to set the user session inactivity timeout through the ServerAdmin UI,
So that I can enforce security policies requiring automatic logout without running CLI commands.

**Context & notes**:
- Wraps existing `vdctools` session timeout functionality
- Applies to all BLSS web application user sessions
- Preset options: 5, 10, 15, 30, 60 minutes + custom entry
- Min: 5 min, Max: 480 min `[TBD — confirm bounds]`
- Changes apply to new sessions only; active sessions keep their current timeout

**Acceptance Criteria**

AC1:
  Given: Admin is logged into ServerAdmin with admin role
  When: Admin navigates to Session Timeout configuration
  Then: Current timeout value is displayed along with active session count and last changed date

AC2:
  Given: Admin is on Session Timeout configuration
  When: Admin selects a new timeout (preset or custom) and saves
  Then: New timeout applied; new sessions use updated timeout

AC3 (negative):
  Given: Admin enters out-of-bounds custom value (e.g., <5 or >480 minutes)
  When: Admin attempts to save
  Then: Validation error shown with acceptable range

AC4:
  Given: Session timeout configured to N minutes
  When: A user is inactive for N minutes
  Then: Session terminated; user redirected to login with "session expired" message

AC5:
  Given: Timeout change is saved
  When: Change completes
  Then: Audit log records: user, timestamp, old value, new value

**Tasks**
- [ ] Task 1: <Backend> Create API endpoint `GET/PUT /api/serveradmin/session-timeout`
- [ ] Task 2: <Backend> Integrate with vdctools session management
- [ ] Task 3: <Frontend> Implement Session Timeout section: current value, preset selector, custom input, active session count
- [ ] Task 4: <Frontend> Inline validation for min/max bounds
- [ ] Task 5: <Test> Integration tests: set timeout, verify session expiry, boundary validation
- [ ] Task 6: <Documentation> Update admin guide

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] Min/max timeout bounds confirmed
- [ ] vdctools session interface documented
- [ ] Impact on active sessions confirmed (next login only)
- [ ] Estimated by team

---

## STORY: S5 — Configure NTP Upstream Source via ServerAdmin UI

**Parent Epic**: Epic 2 — ServerAdmin Network, Session Timeout, and Time Services Configuration
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As an IT Admin / Commissioning Engineer,
I want to configure the NTP upstream time source through the ServerAdmin UI,
So that the BLSS server synchronizes its clock with the correct time source for the environment.

**Context & notes**:
- Configures chrony client on RHEL 9/10
- Supports primary and optional secondary NTP source
- Must validate reachability before applying (with option to override for sources not yet online)
- Critical for air-gapped environments where default NTP pool is unreachable

**Acceptance Criteria**

AC1:
  Given: Admin is on Time Services section
  When: Page loads
  Then: Current NTP source configuration is displayed

AC2:
  Given: Admin enters a new upstream NTP source (IP or hostname) and clicks Apply
  When: System processes the request
  Then: System validates reachability (NTP query test), then applies to chrony configuration

AC3:
  Given: NTP source configured and applied
  When: System performs synchronization
  Then: Server clock syncs with upstream; status shows offset

AC4 (negative):
  Given: Admin enters an unreachable NTP source
  When: Admin clicks Apply
  Then: Warning displayed (source unreachable); admin can override (for future availability) or cancel

AC5:
  Given: NTP source configuration changed
  When: Change applied
  Then: Audit log records: user, timestamp, old source, new source

**Tasks**
- [ ] Task 1: <Backend> Create API endpoint `GET/PUT /api/serveradmin/time/ntp-source`
- [ ] Task 2: <Backend> Implement chrony configuration writer (`/etc/chrony.conf` management)
- [ ] Task 3: <Backend> Implement NTP source reachability check
- [ ] Task 4: <Frontend> Implement NTP source configuration: primary/secondary inputs, apply, reachability indicator
- [ ] Task 5: <Frontend> Warning for unreachable source with override option
- [ ] Task 6: <Test> Integration tests: valid source, unreachable warning, override, config persistence
- [ ] Task 7: <Documentation> Update admin guide

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] chrony conf approach confirmed (direct edit vs. chronyc commands)
- [ ] Single vs. multiple NTP source confirmed
- [ ] Estimated by team

---

## STORY: S6 — Enable/Disable BLoP Server as NTP Server for Connected Devices

**Parent Epic**: Epic 2 — ServerAdmin Network, Session Timeout, and Time Services Configuration
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As a Commissioning Engineer,
I want to enable the BLSS Application Server as an NTP server for connected devices,
So that all devices in isolated networks can synchronize to a common time source without an external NTP server.

**Context & notes**:
- Enables chrony server mode (`allow` directive for client subnets)
- Opens UDP port 123 (via firewalld) for inbound NTP requests
- Critical use case: simple industrial networks with no NTP infrastructure
- Must inform user about firewall implications
- Subnet allowlist: configurable or allow all local subnets `[TBD]`

**Acceptance Criteria**

AC1:
  Given: Admin is on Time Services section
  When: Admin views NTP Server subsection
  Then: Toggle shows current NTP server status (Enabled/Disabled) and subnet allowlist

AC2:
  Given: NTP server disabled
  When: Admin enables toggle and saves
  Then: chrony server mode enabled; UDP 123 accepts NTP requests from allowed subnets; firewall advisory displayed

AC3:
  Given: NTP server enabled
  When: Connected device sends NTP request
  Then: Server responds with current time

AC4:
  Given: NTP server enabled
  When: Admin disables toggle and saves
  Then: chrony server mode disabled; UDP 123 closed

AC5:
  Given: NTP server status changes
  When: Change applied
  Then: Audit log records: user, timestamp, action (enable/disable), allowed subnets

**Tasks**
- [ ] Task 1: <Backend> Create API endpoint `GET/PUT /api/serveradmin/time/ntp-server`
- [ ] Task 2: <Backend> Implement chrony server mode configuration (allow directives, restart)
- [ ] Task 3: <Backend> Implement firewalld rule management for UDP 123
- [ ] Task 4: <Frontend> Implement NTP Server section: toggle, subnet allowlist, firewall advisory
- [ ] Task 5: <Test> Integration tests: enable → NTP responses, disable → no responses
- [ ] Task 6: <Security> Verify NTP amplification mitigation (rate limiting, subnet restriction)
- [ ] Task 7: <Documentation> Update admin guide and firewall documentation

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] Subnet allowlist approach confirmed
- [ ] Firewall management approach confirmed (firewalld)
- [ ] Security mitigation for NTP amplification confirmed
- [ ] Estimated by team

---

## STORY: S7 — NTP Synchronization Status Display & Manual Sync

**Parent Epic**: Epic 2 — ServerAdmin Network, Session Timeout, and Time Services Configuration
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As an IT Admin,
I want to see NTP synchronization status at a glance and trigger manual sync,
So that I can verify time synchronization is working and diagnose issues without CLI access.

**Context & notes**:
- Displays: sync state, current offset, stratum, upstream source, last sync time
- If NTP server mode enabled: show connected client count and list
- "Sync Now" button for manual trigger
- Drift warning if offset exceeds threshold (>1000ms)

**Acceptance Criteria**

AC1:
  Given: Admin is on Time Services section
  When: Page loads
  Then: Status panel shows: sync state (Synchronized/Not Synchronized), offset (ms), stratum, last sync time

AC2:
  Given: NTP server mode is enabled
  When: Admin views status
  Then: Additional info shows connected client count and last request times

AC3:
  Given: Time offset exceeds 1000ms
  When: Status is displayed
  Then: Warning indicator shown with guidance to check NTP source

AC4:
  Given: Admin clicks "Sync Now"
  When: System processes request
  Then: chrony forced sync triggered; status updates after completion

AC5 (negative):
  Given: NTP upstream source unreachable
  When: Status is displayed
  Then: Shows "Not Synchronized" with error detail and last known good sync time

**Tasks**
- [ ] Task 1: <Backend> Create API endpoint `GET /api/serveradmin/time/ntp-status` (parse chrony tracking/sources)
- [ ] Task 2: <Backend> Create API endpoint `POST /api/serveradmin/time/ntp-sync` (trigger makestep/burst)
- [ ] Task 3: <Backend> Implement NTP client counter (chrony clients output) for server mode
- [ ] Task 4: <Frontend> Implement status panel: sync state, offset, stratum, last sync, client count
- [ ] Task 5: <Frontend> Implement drift warning indicator and "Sync Now" button
- [ ] Task 6: <Test> Integration tests: synced status, unsynced status, manual sync, drift warning
- [ ] Task 7: <Documentation> NTP status interpretation and troubleshooting guide

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] Drift warning threshold confirmed (1000ms)
- [ ] Refresh interval confirmed (manual vs. auto-refresh)
- [ ] Estimated by team

---

## STORY: S8 — Audit Logging & RBAC for ServerAdmin Configuration

**Parent Epic**: Epic 2 — ServerAdmin Network, Session Timeout, and Time Services Configuration
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As a Security Administrator,
I want all ServerAdmin configuration changes (network, session, NTP) to be audit-logged and accessible only to admin users,
So that I can track changes for compliance and prevent unauthorized modifications.

**Context & notes**:
- Applies to ALL configuration changes in this Epic (Network, Session, NTP)
- Audit: user, timestamp, category, action, old value, new value, result
- RBAC: all new sections admin-only at UI and API layer
- Non-admin users should not see configuration sections in ServerAdmin nav

**Acceptance Criteria**

AC1:
  Given: Any configuration change is made (IP, hostname, session timeout, NTP source, NTP server toggle)
  When: Change is applied
  Then: Audit log entry created with: user, timestamp, category, action, old value, new value, result

AC2:
  Given: A user with admin role is authenticated
  When: User navigates to ServerAdmin
  Then: Network, Session, Time Services sections are visible and accessible

AC3:
  Given: A user without admin role is authenticated
  When: User views ServerAdmin or attempts direct API access to configuration endpoints
  Then: Configuration sections not visible in UI; API returns 403 Forbidden

AC4:
  Given: Unauthorized access is attempted
  When: API denies the request
  Then: Denial is logged in audit log

**Tasks**
- [ ] Task 1: <Backend> Implement audit logging middleware for all Epic 2 configuration endpoints
- [ ] Task 2: <Backend> Add authorization middleware (admin role check) to all endpoints
- [ ] Task 3: <Frontend> Conditionally render configuration sections based on user role
- [ ] Task 4: <Frontend> Implement access-denied handling for direct URL access
- [ ] Task 5: <Test> Integration tests: admin access granted, non-admin denied, audit entries created
- [ ] Task 6: <Documentation> Document audit schema and RBAC requirements

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] Existing RBAC framework confirmed
- [ ] Audit log storage mechanism confirmed
- [ ] Estimated by team

---

---

## Validation Checklist (Epic 2)

- [x] All AC are testable
- [x] NFRs are quantified
- [x] BLSS product explicitly identified (Shared platform services / ServerAdmin)
- [x] Scope boundaries are explicit
- [x] Cross-product dependencies identified (IP/hostname/NTP affect all products)
- [x] On-prem operational impact assessed (install, upgrade, firewall, backup)
- [x] Dependencies are identified (NetworkManager, chrony, vdctools, firewalld, RBAC)
- [x] No invented features, UI labels, device models, or protocol details
- [x] Terminology matches BLSS product glossary
- [x] Placeholders marked with `[TBD]`
- [x] Operational deliverables included
- [x] Service enablement (Lifecycle Management Services) considered

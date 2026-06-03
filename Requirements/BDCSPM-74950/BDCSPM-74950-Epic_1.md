# BDCSPM-74950 — ServerAdmin Network, Session Timeout, and Time Services Configuration

## EPIC 1: ServerAdmin Network, Session Timeout, and Time Services Configuration

**BLSS Product(s)**: Shared platform services (ServerAdmin)
**Parent Capability**: [BDCSPM-73568](https://eaton-corp.atlassian.net/browse/BDCSPM-73568) — ServerAdmin Self-Service Configuration for Network, Identity, and Time Services
**Fix Version**: BLSS 8.2.0
**Priority**: High
**Status**: Analyzing

**Summary**
Provide administrators with UI-based management of network settings (IP address, hostname), user session auto-logout timeout, and NTP time services (client and server mode) directly within the ServerAdmin page, eliminating CLI dependency for these critical system configurations.

---

### Description

- **Problem statement**: Changing the server IP address, hostname, session timeout, and NTP time configuration currently requires Linux CLI access (`nmcli`, `hostnamectl`, `chronyc`, `vdctools`). Customers in industrial OT environments often lack Linux expertise, resulting in misconfiguration, extended deployment timelines, and increased support ticket volume. In isolated/air-gapped networks, NTP misconfiguration leads to inconsistent timestamps across all BLSS products, making alarm and event correlation unreliable.

- **Proposed solution**: Add new configuration sections directly within the ServerAdmin page:
  1. **Network Configuration**: IP address and hostname changes with validation and safety mechanisms (commit/confirm auto-revert for IP changes, TLS certificate impact warning for hostname changes)
  2. **Session Auto-Logout Timeout**: Configurable inactivity timeout for all user sessions (wrapping existing `vdctools` functionality, in-process apply via etcd broadcast)
  3. **Time Services (NTP)**: Configure upstream NTP source, enable/disable the BLSS Application Server as an NTP server for connected devices, and view synchronization status

  All configurations must preserve existing platform behavior and data consistency per domain: IP and hostname changes must follow the same storage and update strategy as the current `newip`/`newurl` command workflows; session timeout continues to use the existing DB-backed strategy; NTP is not currently persisted as ServerAdmin configuration and current time sync is performed through `/usr/bin/rdate -u -s`. This Epic keeps admin workflows in UI, ensures high-risk changes are safely recoverable, and enforces admin-only access plus full auditability.

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
  - Install/upgrade impact: New configuration sections on ServerAdmin page. Existing persisted configuration and command-managed configuration state must remain available after upgrade. Existing OS-level NTP configuration/status must be correctly displayed and used after upgrade even if it is not stored as ServerAdmin configuration. If this Epic introduces new persisted NTP configuration, upgrade must include that storage so NTP settings also remain available after upgrade.
  - Sizing impact: Minimal — NTP server mode serves responses to admin-configured local subnets only.
  - Network impact: NTP server mode opens UDP port 123 for inbound NTP requests (must be documented in firewall guide). IP change inherently disrupts connectivity temporarily; recovery via 120s commit/confirm auto-revert.
  - Backup impact: Configuration state managed by existing command-aligned IP/hostname storage strategy plus session timeout (in DB), together with revision history, is included in the platform backup/restore pipeline. NTP is not currently backed by ServerAdmin-persisted configuration.

---

### Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Performance | Syntactic validation response time | p95 < 2s on reference hardware | P1 |
| Performance | Network reachability probes (NTP UDP/123, IP ARP, hostname uniqueness) | p95 < 10s, hard timeout 30s, cancellable | P2 |
| Reliability | IP change auto-revert (commit/confirm TTL) | 120s TTL; if admin does not confirm at the new IP within 120s, RSA `compensate_ip` is dispatched and revision moves to `rolled_back` | P0 |
| Reliability | Apply path always re-runs semantic probes immediately before dispatch (TOCTOU) | 100% of mutating apply calls | P0 |
| Reliability | Saga `pending` records reconciled by background Reconciler | Default poll 60s, max age 5min | P0 |
| Reliability | Optimistic concurrency on all mutating endpoints | `If-Match` ETag; conflicts return `409` with diff | P1 |
| Security | All configuration changes audit-logged (intent + outcome) with `correlation_id`, `saga_id`, `schema_version` | 100% of mutations | P0 |
| Security | Audit log is append-only (no UPDATE/DELETE grants on audit table) | Enforced at DB role level | P0 |
| Security | Configuration endpoints gated by RBAC permission `config.system.write` (read = `config.system.read`) | UI + API layer | P0 |
| Security | NTP server mode hardened defaults: `noclientlog`, `cmdport 0`, `ratelimit interval 3 burst 8`, `monlist` disabled, no `0.0.0.0/0` allowed | Enforced in rendered template | P0 |
| Usability | All configuration completable without CLI access | Zero CLI requirement | P0 |
| Usability | Inline validation with clear error messages and per-domain `POST /validate` endpoint | Per-field + domain-level | P1 |
| Usability | Confirmation dialogs for high-impact changes (IP, hostname) with impact summary | All destructive actions | P0 |
| Compatibility | Browser support: Chrome v120+, Edge v120+, Firefox v115+, Safari v17+ | All listed browsers | P1 |
| Compatibility | OS support: RHEL 9, RHEL 10 | All supported OS versions | P0 |
| Availability | Hot-apply tier honored: `in-process` (session) / `reload` (hostname, NTP) / `restart` (IP) | No unplanned restarts beyond IP | P1 |
| Observability | Prometheus metrics: `serveradmin_config_apply_total{domain,result}`, `_apply_duration_seconds`, `_pending_total`, `_rsa_call_duration_seconds` | All domains emit | P1 |
| Observability | Health endpoints `/healthz` (liveness), `/readyz` (DB+etcd+RSA reachability) | Both implemented | P1 |
| Recoverability | Each domain keeps revision history (default N=10) with rollback through the same validate/apply pipeline | All 6 config items | P1 |

---

### Acceptance Criteria (Epic-level)

- [ ] AC1: An admin can change the server IP address via ServerAdmin UI; system validates (incl. ARP collision probe), warns about connectivity impact, and applies with **120-second commit/confirm** auto-revert if confirmation is not received at the new IP.
- [ ] AC2: An admin can change the server hostname via ServerAdmin UI; change applies via service **reload** (no full restart); system warns about TLS certificate impact and cluster name uniqueness.
- [ ] AC3: An admin can set user session inactivity auto-logout time via ServerAdmin UI within bounds **1–1440 minutes**; new timeout applies in-process (etcd broadcast) to subsequent sessions without restart.
- [ ] AC4: An admin can configure NTP upstream source (IP/hostname) via ServerAdmin UI; system enforces "at least one source remains after change"; server synchronizes time with configured source.
- [ ] AC5: An admin can enable/disable the BLSS Application Server as NTP server for connected devices with hardened safe defaults (admin-defined ACL, no `0.0.0.0/0`, `noclientlog`, `cmdport 0`, rate limit, no monlist).
- [ ] AC6: An admin can view NTP synchronization status (sync state, offset, stratum, last sync time, connected clients) in the UI.
- [ ] AC7: All configuration changes are audit-logged with user identity, timestamp, old value, new value, `correlation_id`, `saga_id`, and `schema_version`; audit log is append-only.
- [ ] AC8: All configuration sections enforce admin-only RBAC at UI and API layer; mutating endpoints require optimistic concurrency `If-Match` and reject conflicts with `409 + diff`.
- [ ] AC9: No configuration workflow requires CLI/SSH access.
- [ ] AC10: After upgrade, existing persisted configuration and command-managed configuration state remain available in ServerAdmin without manual re-entry. Existing OS-level NTP configuration/status is correctly displayed and used after upgrade. If new persisted NTP configuration is introduced, upgrade also preserves that configuration.

---

### Common Security, RBAC, and Audit Requirements

These requirements apply to every business story in this Epic and should be implemented inside S1, S2, S4, S5, and S9 rather than as a standalone Jira story.

- All read operations require `config.system.read`; all configuration changes require `config.system.write`.
- Non-admin users must not see configuration sections and direct unauthorized API access must return `403 Forbidden`.
- All configuration change attempts must be audit-logged, including successful changes, failed changes, rejected validation, denied access, rollback, auto-revert, and migration events.
- Audit records must include actor, actor role, source IP, timestamp, domain, action, before/after values, outcome, error when applicable, `correlation_id`, `saga_id` where applicable, and `schema_version`.
- Audit logs must be append-only; existing audit records must not be updated or deleted.
- System-initiated actions must use clear system actor identities, including `system:reconciler` for recovery/auto-revert actions and `system:migration` for upgrade or migration actions.
- Sensitive values must not be written to audit logs in plaintext; use redaction or fingerprinting where needed.

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
  - SAML/LDAP configuration (covered in separate Epic under BDCSPM-73568)

- **Assumptions**:
  1. Current NTP time sync is not stored as ServerAdmin configuration and is performed through `/usr/bin/rdate -u -s`; the target implementation should use chronyd to configure and retrieve NTP information.
  2. IP change auto-revert TTL is **120 seconds** (commit/confirm pattern); if not confirmed at the new IP within the window, the system automatically reverts to the previous confirmed configuration.
  3. Session timeout wraps existing `vdctools` functionality and is broadcast in-process via etcd to all listeners; no service restart needed.
  4. ⚠️ ASSUMPTION: Server has single NIC (multi-NIC/bonding out of scope for v1).
  5. ⚠️ ASSUMPTION: IPv4 only for v1.
  6. Network changes must support safe apply and rollback outcomes with clear admin feedback.
  6a. **IP and hostname storage must remain consistent with existing command workflows** (`newip`/`newurl`), including all current values and stores those workflows update. The application does **not** introduce a parallel source of truth for IP/hostname.
  7. Hostname change requires manual TLS certificate re-issuance (automated cert handling is future scope).
  8. NTP server ACL is admin-defined; the system rejects `0.0.0.0/0` (or any wildcard) and enforces hardened chrony defaults (`noclientlog`, `cmdport 0`, `ratelimit interval 3 burst 8`, no `monlist`).

---

### Dependencies & Risks

| Item | Type (Dep/Risk) | BLSS Product | Owner | Mitigation | Status |
|---|---|---|---|---|---|
| OS NetworkManager/nmcli API stability on RHEL 9/10 | Dep | Shared platform | Platform Engineering | Validate API across supported OS versions | `[TBD]` |
| chrony NTP daemon configuration API | Dep | Shared platform | Platform Engineering | Validate chrony conf management approach | `[TBD]` |
| vdctools session timeout interface | Dep | Shared platform | Platform Engineering | Document vdctools CLI for session timeout | `[TBD]` |
| firewalld for NTP port management | Dep | Shared platform | Platform Engineering | Validate firewalld integration for UDP 123 | `[TBD]` |
| RBAC framework (admin-only page access) | Dep | Shared platform | Platform Engineering | Confirm existing RBAC supports section-level access | `[TBD]` |
| IP change causes admin session/connectivity loss | Risk | Shared platform | Platform Engineering | 120s commit/confirm auto-revert; ARP collision probe; pre-change warning modal | Open |
| Hostname change breaks TLS certificates | Risk | Shared platform | Platform Engineering | Warn user; provide certificate re-issuance guidance | Open |
| NTP misconfiguration causes timestamp drift across all products | Risk | Cross-product | Platform Engineering | Validate NTP source reachability before applying; drift warning at >1000ms | Open |
| NTP server mode amplification attack | Risk | Shared platform | Platform Engineering | Rate limiting; subnet allowlist; document security implications | Open |

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

### Story Breakdown (summary)

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| BDCSPM-74950-S1 | Configure Server IP Address via ServerAdmin UI | `[TBD]` | `[TBD]` |
| BDCSPM-74950-S2 | Configure Server Hostname via ServerAdmin UI | `[TBD]` | `[TBD]` |
| BDCSPM-74950-S3 | Configure User Session Auto-Logout Timeout via ServerAdmin UI | `[TBD]` | `[TBD]` |
| BDCSPM-74950-S4 | Configure and Monitor NTP Time Services via ServerAdmin UI | `[TBD]` | `[TBD]` |
| BDCSPM-74950-S5 | Preserve Existing Configuration on Upgrade | `[TBD]` | `[TBD]` |

---

---

## STORY: BDCSPM-74950-S1 — Configure Server IP Address via ServerAdmin UI

**Parent Epic**: [BDCSPM-74950](https://eaton-corp.atlassian.net/browse/BDCSPM-74950) — ServerAdmin Network, Session Timeout, and Time Services Configuration
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As a Commissioning Engineer / System Integrator,
I want to change the BLSS Application Server IP address through the ServerAdmin web UI instead of using the existing `newip` command,
So that I can reconfigure the server's network identity with the same supported capability without requiring command-line access.

**Context & notes**:
- Highest-risk configuration change — applying a new IP disrupts the admin's current browser session
- **Existing command capability must be replaced**: IP configuration in ServerAdmin must fully cover the current `newip` command workflow and supported behavior, including current-value display, IP/subnet/gateway input, validation, apply, reconnect/confirm, rollback or recovery behavior, and clear outcome reporting
- **Existing storage strategy must be preserved**: the current `newip` command may update multiple IP-related values across existing platform stores (for example etcd, DB, or other existing locations). ServerAdmin must follow the same read/write storage strategy and update all existing IP state required by `newip`; it must not introduce a new parallel IP source of truth.
- Hot-apply tier: **restart** (network interface)
- IPv4 only for v1; single NIC assumption
- **Safety mechanism is mandatory**: IP change must include validation, impact warning, 120-second commit/confirm at the new IP, auto-revert if not confirmed, reconciliation for incomplete apply attempts, and audit trail
- See prototype: `Requirements/BDCSPM-73568/prototype-serveradmin-self-service-config.html` — "Network" tab and IP change modal with countdown

**Acceptance Criteria**

AC1:
  Given: Admin is logged into ServerAdmin with admin role
  When: Admin navigates to Network Configuration section
  Then: Current server IP address is displayed using the same existing storage and host-state sources used by the `newip` workflow

AC2:
  Given: Admin is on Network Configuration section
  When: Admin enters a valid new IP address and clicks Apply
  Then: System validates the input and displays a confirmation modal showing current IP, new IP, connectivity-loss impact, restart impact, and 120-second auto-revert behavior

AC3:
  Given: Admin has confirmed IP change in modal
  When: System applies the new IP address
  Then: System applies the change with behavior equivalent to the existing `newip` command capability, starts a 120-second confirm window, and prompts the admin to reconnect at the new IP and confirm the change

AC3a:
  Given: The 120-second confirm window is running
  When: Admin reaches ServerAdmin at the new IP and confirms the change
  Then: The new IP is retained across all existing IP storage locations required by the `newip` storage strategy; auto-revert is cancelled; audit log records the successful change

AC3b:
  Given: The 120-second confirm window is running
  When: Admin does not confirm the change within the window
  Then: System automatically reverts to the previous confirmed IP configuration; audit log records the system-initiated rollback

AC3c:
  Given: The IP apply attempt is interrupted, times out, or the ServerAdmin process restarts during the confirm window
  When: The system evaluates the unfinished change
  Then: The incomplete change is resolved to a final safe state (confirmed or reverted); the final outcome is visible to the admin and recorded in audit

AC4 (negative):
  Given: Admin enters an invalid IP address (malformed, reserved range 0.0.0.0, 255.255.255.255) or an IP with active ARP collision on the target subnet
  When: Admin attempts to apply
  Then: Inline validation error displayed (with reason); change is not submitted

AC5:
  Given: IP change has been applied and confirmed
  When: Change completes successfully
  Then: Audit log records the actor, old IP, new IP, timestamp, outcome, and correlation information; revision history remains available according to the existing `newip` storage and history strategy

AC6 (concurrency):
  Given: Two admins edit the IP concurrently
  When: The second admin submits based on stale configuration
  Then: System rejects the stale update and prompts the admin to refresh and retry with the latest value

**Tasks**
- [ ] Task 1: <Backend> Map the existing `newip` command workflow and supported behavior to ServerAdmin UI/API requirements
- [ ] Task 2: <Backend> Map the existing `newip` storage strategy, including every IP-related value and store it currently updates, and use the same read/write strategy in ServerAdmin without adding a new parallel IP source of truth
- [ ] Task 3: <Backend> Implement IP validation before submit and immediately before apply, including format, subnet/gateway consistency, reserved ranges, and ARP collision detection
- [ ] Task 4: <Backend/RSA> Apply IP changes using the existing RSA host execution mechanism with behavior parity to the current `newip` capability and support rollback to the previous IP if confirmation is not completed
- [ ] Task 5: <Backend> Implement 120-second confirm/auto-revert flow and reconciliation for interrupted or incomplete apply attempts
- [ ] Task 6: <Backend> Enforce admin RBAC, optimistic concurrency, revision history, and audit logging for intent, success, failure, and auto-revert outcomes
- [ ] Task 7: <Frontend> Implement IP configuration UI with current value, validation feedback, impact confirmation modal, restart/hot-apply warning, 120-second countdown, and confirm action at the new IP
- [ ] Task 8: <Test> Cover behavior parity with the existing `newip` command, valid apply + confirm, invalid input, ARP collision, stale update rejection, auto-revert after timeout, interrupted apply reconciliation, and audit record creation
- [ ] Task 9: <Documentation> Update admin guide with replacement of the `newip` command workflow, IP change workflow, auto-revert behavior, and console break-glass instructions

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] UX mockups attached
- [ ] NetworkManager API approach confirmed
- [ ] Existing `newip` command behavior documented and mapped to ServerAdmin UI/API coverage
- [ ] Existing `newip` storage strategy documented, including etcd/DB/other IP-related values it updates
- [ ] Single NIC vs. multi-NIC scope confirmed
- [ ] IPv4-only scope confirmed
- [ ] Existing IP storage contract confirmed; ServerAdmin will preserve the same storage strategy as `newip` and will not create a new parallel source of truth
- [ ] Auto-revert TTL confirmed (**120s**) and recovery behavior agreed
- [ ] Existing RSA apply/rollback interface confirmed
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

## STORY: BDCSPM-74950-S2 — Configure Server Hostname via ServerAdmin UI

**Parent Epic**: [BDCSPM-74950](https://eaton-corp.atlassian.net/browse/BDCSPM-74950) — ServerAdmin Network, Session Timeout, and Time Services Configuration
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As a System Integrator / IT Admin,
I want to change the BLSS Application Server hostname through the ServerAdmin web UI instead of using the existing `newurl` command,
So that I can align the server naming with customer network standards with the same supported capability without requiring command-line access.

**Context & notes**:
- Hostname change affects: TLS certificates (if hostname-bound), internal service references, log entries
- **Existing command capability must be replaced**: hostname configuration in ServerAdmin must fully cover the current `newurl` command workflow and supported behavior, including current-value display, hostname input, validation, apply, service reload behavior, rollback or recovery behavior, dependent URL/reference updates where applicable, and clear outcome reporting
- **Existing storage strategy must be preserved**: ServerAdmin must follow the same read/write storage strategy as the current `newurl` command, including every hostname/URL-related value and existing store it updates; it must not introduce a new parallel hostname source of truth.
- Hot-apply tier: **reload** (no full system restart)
- DNS registration is out of scope (customer responsibility)
- **Safety mechanism is mandatory**: hostname change must include RFC 1123 validation, cluster-name uniqueness check, impact warning, immediate re-validation before apply, recovery for incomplete apply attempts, and audit trail
- Hostname change does not require the 120-second IP commit/confirm window, but the UI must clearly warn about TLS certificate impact, service reload impact, and any manual follow-up needed after the hostname changes
- See prototype: `Requirements/BDCSPM-73568/prototype-serveradmin-self-service-config.html` — "Network" tab

**Acceptance Criteria**

AC1:
  Given: Admin is logged into ServerAdmin with admin role
  When: Admin navigates to Network Configuration section
  Then: Current hostname is displayed using the same existing storage and host-state sources used by the `newurl` workflow

AC2:
  Given: Admin enters a valid new hostname (RFC 1123 compliant)
  When: Admin clicks Apply
  Then: System validates the hostname with coverage equivalent to the existing `newurl` command behavior and displays a confirmation modal warning about TLS certificate impact, service reload impact, and cluster-name uniqueness

AC3:
  Given: Admin confirms hostname change
  When: System applies the change
  Then: System applies the hostname change with behavior equivalent to the existing `newurl` command capability, applies required service reload behavior, and ServerAdmin reflects the new hostname

AC4 (negative):
  Given: Admin enters an invalid hostname (special chars, >63 chars, empty)
  When: Admin attempts to apply
  Then: Inline validation error shown; change not submitted

AC5:
  Given: Hostname change is applied
  When: Change completes
  Then: Audit log records the actor, old hostname, new hostname, timestamp, outcome, and correlation information; revision history remains available according to the existing hostname storage and history strategy used by `newurl`

AC6:
  Given: Validation succeeded earlier but the hostname becomes invalid or conflicts with another known cluster member before apply
  When: Admin submits the change
  Then: System re-validates before apply, rejects the stale or conflicting change, and does not update the host hostname

AC7:
  Given: Hostname apply is interrupted, times out, or returns an uncertain outcome
  When: The system evaluates the unfinished change
  Then: The incomplete change is resolved to a final safe state (confirmed or rolled back); the final outcome is visible to the admin and recorded in audit

AC8 (concurrency):
  Given: Two admins edit the hostname concurrently
  When: The second admin submits based on stale configuration
  Then: System rejects the stale update and prompts the admin to refresh and retry with the latest value

**Tasks**
- [ ] Task 1: <Backend> Map the existing `newurl` command workflow and supported behavior to ServerAdmin UI/API requirements
- [ ] Task 2: <Backend> Read and update hostname configuration through the existing storage strategy used by `newurl`, including every hostname/URL-related value and store it currently updates
- [ ] Task 3: <Backend> Implement hostname validation before submit and immediately before apply, including RFC 1123 rules and cluster-name uniqueness
- [ ] Task 4: <Backend/RSA> Apply hostname changes using the existing RSA host execution mechanism with behavior parity to the current `newurl` capability; reload hostname-dependent services without full restart and support rollback to the previous hostname if apply cannot be completed safely
- [ ] Task 5: <Backend> Implement reconciliation for interrupted or incomplete hostname apply attempts
- [ ] Task 6: <Backend> Enforce admin RBAC, optimistic concurrency, revision history, and audit logging for intent, success, failure, and rollback outcomes
- [ ] Task 7: <Frontend> Implement hostname configuration UI with current value, validation feedback, impact confirmation modal, TLS certificate warning, service reload warning, and cluster-uniqueness messaging
- [ ] Task 8: <Test> Cover behavior parity with the existing `newurl` command, valid apply, invalid input, cluster-name conflict, stale update rejection, reload-not-restart behavior, interrupted apply reconciliation, rollback, and audit record creation
- [ ] Task 9: <Documentation> Update admin guide with replacement of the `newurl` command workflow, hostname change workflow, TLS certificate re-issuance guidance, service reload impact, and rollback behavior

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] UX mockups attached
- [ ] Existing `newurl` command behavior documented and mapped to ServerAdmin UI/API coverage
- [ ] List of hostname-dependent services confirmed
- [ ] TLS certificate impact documented
- [ ] Existing `newurl` storage strategy documented, including hostname/URL-related values and stores it updates
- [ ] Existing hostname storage contract confirmed; ServerAdmin will preserve the same storage strategy as `newurl` and will not create a new parallel source of truth
- [ ] Existing RSA apply/rollback interface confirmed
- [ ] Hostname validation and cluster-uniqueness rules confirmed
- [ ] Estimated by team

---

## STORY: BDCSPM-74950-S3 — Configure User Session Auto-Logout Timeout via ServerAdmin UI

**Parent Epic**: [BDCSPM-74950](https://eaton-corp.atlassian.net/browse/BDCSPM-74950) — ServerAdmin Network, Session Timeout, and Time Services Configuration
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As an Administrator,
I want to set the user session inactivity timeout through the ServerAdmin UI,
So that I can enforce security policies requiring automatic logout without running CLI commands.

**Context & notes**:
- Wraps existing `vdctools` session timeout functionality
- Applies to all BLSS web application user sessions
- Preset options: 5, 10, 15, 30, 60 minutes + custom entry
- **Bounds: 1–1440 minutes** (FR-5); UI warns when value `<5`
- **Existing storage must be preserved**: session timeout value is stored in DB today; implementation must continue to read/write the existing DB-backed source of truth 
- Hot-apply tier: **in-process** — DB remains the source of truth; the new value is propagated to in-process listeners through the existing runtime notification/broadcast mechanism; no service restart
- Changes apply to subsequent sessions; active sessions keep their current timeout until next login
- **Safety mechanism is mandatory**: configuration must enforce bounds, reject stale concurrent updates, restrict mutation to authorized admins, audit all changes, and avoid interrupting active user sessions unexpectedly
- See prototype: `Requirements/BDCSPM-73568/prototype-serveradmin-self-service-config.html` — "Session" section

**Acceptance Criteria**

AC1:
  Given: Admin is logged into ServerAdmin with admin role
  When: Admin navigates to Session Timeout configuration
  Then: Current timeout value is displayed from the existing DB-backed configuration source, along with active session count and last changed date when available

AC2:
  Given: Admin is on Session Timeout configuration
  When: Admin selects a new timeout (preset or custom) and saves
  Then: New timeout is validated, saved to the existing source of truth, and used by subsequent sessions without requiring admin CLI actions

AC3 (negative):
  Given: Admin enters out-of-bounds custom value (e.g., `<1` or `>1440` minutes)
  When: Admin attempts to save
  Then: Validation error shown with acceptable range **1–1440 minutes**; values `<5` are accepted but show a soft warning

AC4:
  Given: Session timeout configured to N minutes
  When: A user is inactive for N minutes
  Then: Session terminated; user redirected to login with "session expired" message

AC5:
  Given: Timeout change is saved
  When: Change completes
  Then: Audit log records the actor, old timeout, new timeout, timestamp, and outcome; revision history remains available through the existing session-timeout history strategy

AC6 (concurrency):
  Given: Two admins edit the session timeout concurrently
  When: The second admin submits based on stale configuration
  Then: System rejects the stale update and prompts the admin to refresh and retry with the latest value

AC7 (authorization):
  Given: A non-admin user attempts to view or change session timeout configuration
  When: The UI or API request is evaluated
  Then: The section is hidden or access is denied, and unauthorized mutation attempts are audited

**Tasks**
- [ ] Task 1: <Backend> Read and update session timeout through the existing DB-backed storage model
- [ ] Task 2: <Backend> Implement validation for bounds **1–1440 minutes** and preserve the soft warning for values `<5`
- [ ] Task 3: <Backend> Apply the updated timeout in-process using the existing runtime propagation mechanism so subsequent sessions use the new value without service restart
- [ ] Task 4: <Backend> Enforce admin RBAC, optimistic concurrency, revision history, and audit logging for intent, success, failure, and denied access outcomes
- [ ] Task 5: <Frontend> Implement Session Timeout section with current value, preset selector, custom input, validation feedback, soft warning for low values, active session count, and last changed date when available
- [ ] Task 6: <Test> Cover valid save, boundary values, out-of-range rejection, soft warning below 5 minutes, stale update rejection, in-process propagation to subsequent sessions, active-session behavior, and audit record creation
- [ ] Task 7: <Documentation> Update admin guide with session timeout configuration, supported range, low-value warning, and active-session impact

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] Bounds confirmed (**1–1440 minutes**)
- [ ] Existing DB-backed session timeout storage contract confirmed; no storage migration required for session timeout state
- [ ] Existing runtime propagation/listener contract documented
- [ ] vdctools session interface documented
- [ ] Impact on active sessions confirmed (next login only)
- [ ] Estimated by team

---

## STORY: BDCSPM-74950-S4 — Configure and Monitor NTP Time Services via ServerAdmin UI

**Parent Epic**: [BDCSPM-74950](https://eaton-corp.atlassian.net/browse/BDCSPM-74950) — ServerAdmin Network, Session Timeout, and Time Services Configuration
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As an IT Admin / Commissioning Engineer,
I want to configure and monitor BLSS NTP time services through the ServerAdmin UI,
So that the server and connected devices can maintain reliable time synchronization without requiring Linux CLI access.

**Context & notes**:
- **Existing behavior to replace**: NTP is not currently stored as ServerAdmin configuration; current manual time synchronization depends on `/usr/bin/rdate -u -s`. The new Time Services UI should replace this operational capability so admins can configure, sync, and monitor time services without CLI access. Implementation must not assume an existing DB-backed NTP source of truth.
- NTP upstream-source scope: configure primary/secondary upstream source, validate syntax/reachability, prevent unsafe empty-source behavior, and show unreachable-source warning/override where operationally required.
- NTP server-mode scope: enable/disable BLSS Application Server as an NTP server for connected devices, manage subnet allowlist, reject `0.0.0.0/0` or wildcard ACLs, apply firewall behavior for UDP 123, and expose connected-client visibility.
- NTP status/manual-sync scope: show synchronized/not synchronized state, offset, stratum, upstream source, last sync time, connected clients when server mode is enabled, drift warning, and manual "Sync Now".
- **Safety mechanism is mandatory**: enforce secure NTP server defaults, reject unsafe allowlists, restrict changes to authorized admins, handle conflicting updates safely, and audit all changes.
- **Safety mechanism is mandatory**: system must enforce secure behavior, reject unsafe configuration, restrict changes to authorized admins, ensure recoverable outcomes, and audit all changes.
- See prototype: `Requirements/BDCSPM-73568/prototype-serveradmin-self-service-config.html` — "Time Services" tab

**Acceptance Criteria**

AC1:
  Given: Admin is on Time Services section
  When: Page loads
  Then: System displays current NTP status where available (synchronization state, offset, stratum, upstream source, last sync time) and clearly supports replacing the legacy `/usr/bin/rdate -u -s` manual sync workflow through the UI

AC2:
  Given: Admin enters or updates an upstream NTP source (IP or hostname)
  When: Admin saves the change
  Then: System validates syntax and reachability, applies the requested time-service configuration, and records the change in audit without requiring CLI operations

AC3:
  Given: Admin enables BLSS Application Server NTP server mode for connected devices
  When: Admin provides a valid non-wildcard subnet allowlist and saves
  Then: System enables NTP server mode with secure behavior, allows NTP requests only from approved subnets, applies required network access behavior, and records the change in audit

AC3a (negative — unsafe ACL):
  Given: Admin enters `0.0.0.0/0` or any wildcard ACL for NTP server mode
  When: Admin attempts to save
  Then: System rejects the change with a security warning explaining NTP amplification risk; change is not submitted

AC3b (negative — disallowed subnet):
  Given: NTP server mode is enabled with a configured subnet allowlist
  When: A device outside the allowed subnet sends an NTP request
  Then: Request is dropped or receives no NTP response; behavior is verifiable in tests

AC4:
  Given: NTP server mode is enabled
  When: Admin disables NTP server mode and saves
  Then: System disables server mode, stops accepting NTP requests from connected devices, applies required network access behavior, and records the change in audit

AC5:
  Given: Admin views NTP status
  When: The status panel is refreshed
  Then: System shows synchronized/not synchronized state, current offset, stratum, upstream source, last sync time, and connected-client information when server mode is enabled

AC5a:
  Given: Current time offset exceeds the configured drift warning threshold
  When: NTP status is displayed
  Then: UI shows a warning indicator with guidance to check the upstream time source

AC5b:
  Given: Upstream NTP source is unreachable or the server time service is not synchronized
  When: NTP status is displayed
  Then: UI shows "Not Synchronized" with error detail and last known good sync time when available

AC6:
  Given: Admin clicks "Sync Now"
  When: System processes the request
  Then: The UI triggers time synchronization, updates the status panel after completion, and no CLI access is required from the admin

AC7 (negative — unsafe or invalid configuration):
  Given: Admin enters an unreachable source, removes the last source, enters an unsafe wildcard ACL, or configures an invalid subnet
  When: Admin attempts to save
  Then: System blocks unsafe changes where required, displays clear validation or warning messages, and audits any explicit override for an unreachable but intentionally future-available source

AC8 (authorization/concurrency):
  Given: A non-admin user attempts to change NTP configuration, or two admins submit conflicting NTP changes
  When: The request is evaluated
  Then: Unauthorized mutation is denied and audited; stale updates are rejected and the admin is prompted to refresh

**Tasks**
- [ ] Task 1: <Backend> Provide Time Services APIs for upstream source configuration, NTP server mode, manual sync, and status retrieval without assuming an existing DB-backed NTP source of truth
- [ ] Task 2: <Backend> Validate upstream sources and subnet allowlists, including reachability warnings, last-source guard, invalid subnet rejection, and unsafe wildcard allowlist rejection
- [ ] Task 3: <Backend> Support NTP server mode enable/disable, approved subnet access, connected-client visibility, secure defaults, and required UDP 123 firewall behavior
- [ ] Task 4: <Backend> Support status and "Sync Now" behavior, including synchronized/not synchronized state, offset, stratum, upstream source, last sync time, drift warning, and last known good sync time
- [ ] Task 5: <Backend> Enforce admin RBAC, conflicting-update handling, audit logging, and observability for NTP configuration, sync, and status operations
- [ ] Task 6: <Frontend> Implement Time Services UI for upstream source configuration, server-mode toggle, subnet allowlist, validation feedback, sync status, connected clients, drift warning, unreachable/unsynchronized details, last known good sync time, and "Sync Now"
- [ ] Task 7: <Test> Cover upstream source configuration, reachability warning/override, last-source guard, server-mode enable/disable, unsafe allowlist rejection, subnet access behavior, firewall behavior, manual sync, status display, drift warning, conflict handling, RBAC, and audit records
- [ ] Task 8: <Documentation> Update admin guide, firewall documentation, and troubleshooting guide for NTP client/server configuration, UDP 123 impact, status interpretation, and manual sync

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] Confirmed that NTP is not currently persisted in DB and legacy manual sync uses `/usr/bin/rdate -u -s`
- [ ] Target implementation approach confirmed during technical design
- [ ] Single vs. multiple upstream source support confirmed
- [ ] Last-source guard semantics confirmed
- [ ] Subnet allowlist and firewall management approach confirmed
- [ ] Security mitigation for NTP amplification confirmed
- [ ] Status refresh behavior and drift warning threshold confirmed
- [ ] Estimated by team

---

## STORY: BDCSPM-74950-S5 — Preserve Existing Configuration on Upgrade

**Parent Epic**: [BDCSPM-74950](https://eaton-corp.atlassian.net/browse/BDCSPM-74950) — ServerAdmin Network, Session Timeout, and Time Services Configuration
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As an On-Prem IT / System Admin upgrading an existing BLSS Application Server to the version that introduces ServerAdmin self-service configuration,
I want existing ServerAdmin-related configuration to remain available after upgrade,
So that the upgrade is non-disruptive and administrators can continue using the new UI without re-entering existing values.

**Context & notes**:
- Upgrade must preserve existing configuration managed by current command/storage strategies, including IP address, hostname, and session timeout.
- NTP can currently be changed through commands, but the changed value is not stored as ServerAdmin configuration. After upgrade, the current OS-level NTP configuration/status must be displayed correctly and continue to be used.
- If this Epic introduces new persisted NTP configuration, that storage must also be covered by upgrade.
- The story goal is to verify that upgrade works end to end; specific migration implementation details are owned by technical design.
- Upgrade must be non-disruptive: existing values should be visible in ServerAdmin after upgrade and should not require manual re-entry.
- Upgrade outcomes must follow the Epic-level RBAC and audit requirements.

**Acceptance Criteria**

AC1:
  Given: An existing node is upgraded to the version that includes BDCSPM-74950
  When: The system starts after upgrade
  Then: Existing IP address, hostname, and session timeout values managed by current command/storage strategies remain available and are shown correctly in ServerAdmin

AC2:
  Given: Existing NTP configuration or status is available at the OS level before upgrade
  When: The system is upgraded
  Then: The current NTP configuration/status is shown correctly in Time Services and continues to be used after upgrade

AC3:
  Given: New persisted NTP configuration is introduced by this Epic
  When: The system is upgraded
  Then: NTP configuration stored by the product remains available after upgrade and is shown correctly in Time Services

AC4:
  Given: Upgrade has completed successfully
  When: Admin opens ServerAdmin Network, Session, and Time Services sections
  Then: Existing applicable values are visible without manual re-entry and the administrator can continue managing them through the UI

AC5:
  Given: Upgrade cannot preserve or read one configuration domain correctly
  When: Admin opens the affected ServerAdmin section
  Then: The UI shows a clear warning and remediation guidance without blocking access to unaffected configuration domains

AC6:
  Given: Upgrade preserves, transforms, or reports a configuration issue
  When: The audit log is inspected
  Then: Upgrade-related outcomes are audit-logged according to the Epic-level common audit requirements

**Tasks**
- [ ] Task 1: <Backend> Ensure existing IP and hostname configuration remains available after upgrade according to the current `newip`/`newurl` storage strategies
- [ ] Task 2: <Backend> Ensure existing DB-backed session timeout configuration remains available after upgrade
- [ ] Task 3: <Backend> Ensure existing OS-level NTP configuration/status is correctly read, displayed, and used after upgrade even when it is not ServerAdmin-persisted
- [ ] Task 4: <Backend> Include any newly introduced persisted NTP configuration in upgrade coverage if such storage is added by this Epic
- [ ] Task 5: <Backend> Provide clear upgrade outcome reporting and audit logging for preserved, transformed, or failed configuration domains
- [ ] Task 6: <Frontend> Show existing upgraded values in the relevant ServerAdmin sections and display remediation guidance when a domain cannot be preserved or read
- [ ] Task 7: <Test> Cover upgrade scenarios for command-managed IP/hostname configuration, DB-backed session timeout configuration, existing OS-level NTP configuration/status, optional persisted NTP configuration, partial failure handling, UI visibility, and audit records
- [ ] Task 8: <Documentation> Document upgrade expectations, preserved configuration domains, OS-level NTP display/use behavior, optional NTP persistence behavior, and troubleshooting steps

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] Existing command-managed IP/hostname upgrade behavior confirmed
- [ ] Existing DB-backed configuration upgrade behavior confirmed
- [ ] Existing OS-level NTP configuration/status display behavior confirmed
- [ ] NTP persistence decision confirmed
- [ ] Upgrade failure/remediation UX agreed
- [ ] Estimated by team

---

---

## Validation Checklist (BDCSPM-74950)

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

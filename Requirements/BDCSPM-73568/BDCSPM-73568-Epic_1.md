# BDCSPM-73568 — ServerAdmin Self-Service Configuration for Network, Identity, and Time Services

## EPIC 1: UI-Based SAML SSO Configuration & LDAP Group Permission Mapping

**BLSS Product(s)**: Shared platform services (ServerAdmin → System Settings → Authentication Configuration)
**Summary** (1-2 sentences)
Enable administrators to configure SAML Single Sign-On and manage LDAP group-to-role permission mappings directly from the System Settings > Authentication Configuration page, eliminating CLI dependency for identity and access management.

---

### Description

- **Problem statement**: Currently, configuring SAML authentication and LDAP group-to-role permission mappings on the BLSS Application Server requires editing configuration files via the Linux CLI (or invoking `vdctools` commands). Customers in OT environments frequently lack Linux expertise, resulting in deployment delays, misconfiguration risk, and excessive support ticket volume for routine identity setup tasks.

- **Proposed solution**: Enhance the existing System Settings > Authentication Configuration area:
  1. **SAML Configuration page (new)**: A dedicated sub-page for configuring the BLSS server as a SAML 2.0 Service Provider — including IdP metadata import, SP Entity ID, attribute mappings, and enable/disable toggle with test-before-apply safety
  2. **LDAP Group Mapping table (enhancement to existing page)**: Add a group-to-role mapping table on the existing LDAP configuration page, allowing admins to search AD groups and assign BLSS application roles (Administrator, Operator, Viewer) via the UI — wrapping existing `vdctools` backend functionality

  Both configurations include a guaranteed local admin fallback login to prevent lockout from misconfiguration.

- **User personas**: On-Prem IT / System Admin, Commissioning Engineer, System Integrator, Support Teams

- **Business value**:
  - Eliminates CLI dependency for identity configuration during deployment
  - Reduces SAML/LDAP-related support tickets by enabling self-service
  - Accelerates commissioning: identity setup from hours (CLI + troubleshooting) to minutes (guided UI)
  - Prevents authentication lockout through built-in safety mechanisms
  - Standardizes identity configuration across customer deployments

- **Key workflows**:
  1. Admin navigates to System Settings → Authentication Configuration → SAML tab
  2. Admin imports IdP metadata (URL or XML file upload), configures SP Entity ID and attribute mappings
  3. Admin clicks "Test Configuration" to validate before enabling
  4. Admin enables SAML; SSO login option appears on login page
  5. Admin navigates to LDAP Configuration page → Group Permissions section
  6. Admin searches AD groups, selects group, assigns BLSS role, saves mapping
  7. All changes are audit-logged; local admin fallback remains always available

- **Cross-product touchpoints**: Authentication configuration affects login flow for all BLSS products (DCPM, EPMS Standard, Distributed IT, Brightlayer Power) since they share the same auth layer. LDAP group role mappings apply to all product access.

- **UX/UI references**: See prototype `Requirements/BDCSPM-73568/prototype-serveradmin-self-service-config.html` — "Identity & Access" tab. `[TBD — final UX mockups from design team]`

- **On-prem operational impact**:
  - Install/upgrade impact: New SAML configuration page added under System Settings. Upgrade must preserve any existing CLI-configured SAML/LDAP settings and surface them in the UI.
  - Sizing impact: Minimal — UI pages and validation logic only.
  - Network impact: SAML requires outbound HTTPS to IdP metadata URL (optional — can use file upload for air-gapped environments). LDAP requires existing connectivity to AD/LDAP server.
  - Backup impact: SAML configuration and LDAP group mappings must be included in backup scope. Secrets stored encrypted at rest.

---

### Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Performance | SAML configuration save response time | < 2s at P95 on reference hardware (Small tier) | P1 |
| Performance | LDAP group search response time | < 3s at P95 (depends on AD server performance) | P1 |
| Security | SAML certificates and LDAP bind credentials stored encrypted at rest | AES-256 or equivalent | P0 |
| Security | No secrets exposed in UI after initial entry (masked display) | 100% masked | P0 |
| Security | Configuration changes audit-logged (user, timestamp, action) | 100% of changes | P0 |
| Security | Secrets NOT written to audit log | Zero secret leakage | P0 |
| Recoverability | Local admin fallback always available even if SAML/LDAP misconfigured | Cannot be disabled | P0 |
| Recoverability | Test-before-apply for SAML (validate IdP reachability, metadata, certificate) | Pre-save validation | P0 |
| Usability | All identity configuration completable without CLI | Zero CLI requirement | P0 |
| Usability | Inline validation with clear error messages | Per-field validation | P1 |
| Compatibility | Browser support: Chrome v120+, Edge v120+, Firefox v115+, Safari v17+ | All listed browsers | P1 |
| Compatibility | OS support: RHEL 9, RHEL 10 | All supported OS versions | P0 |

---

### Acceptance Criteria (Epic-level)

- [ ] AC1: An admin can configure SAML SP settings (IdP metadata URL/file, Entity ID, attribute mappings) via System Settings > Authentication Configuration > SAML; SSO login functions after configuration.
- [ ] AC2: An admin can test SAML configuration (validate IdP metadata, certificate, connectivity) before enabling, with clear pass/fail result.
- [ ] AC3: An admin can enable/disable SAML SSO via toggle; when enabled, SSO option appears on login page.
- [ ] AC4: An admin can view, add, and remove LDAP group-to-role mappings in a table on the LDAP configuration page.
- [ ] AC5: An admin can search AD groups from the LDAP directory when adding a new mapping.
- [ ] AC6: LDAP group role changes take effect on next user login.
- [ ] AC7: A local admin fallback login (`/serveradmin/local-login`) remains always accessible regardless of SAML/LDAP state.
- [ ] AC8: All configuration changes are audit-logged (user, timestamp, action, non-secret values).
- [ ] AC9: Secrets (certificates, passwords) are stored encrypted and displayed masked in the UI.
- [ ] AC10: Authentication configuration pages enforce admin-only role-based access.

---

### Scope

- **In scope**:
  - New SAML configuration sub-page under System Settings > Authentication Configuration
  - SAML 2.0 SP configuration: IdP metadata import (URL or XML file), Entity ID, ACS URL, attribute mappings (username, email, groups)
  - SAML enable/disable toggle with test-before-apply
  - LDAP group permission mapping table on existing LDAP configuration page
  - LDAP group search (query AD for available groups)
  - Group-to-role mapping CRUD (Administrator, Operator, Viewer roles)
  - Local admin fallback login (always available, cannot be disabled)
  - Audit logging for all identity configuration changes
  - Admin-only RBAC enforcement on configuration pages

- **Out of scope**:
  - LDAP connection configuration (assumes already configured in existing System Settings)
  - User provisioning via LDAP (only group-to-role mapping)
  - Multi-IdP SAML support (single IdP per instance)
  - OIDC/OAuth2 support (future scope)
  - Certificate Authority management (beyond SAML certificate upload)
  - CLI-based SAML/LDAP configuration removal (existing CLI path remains as fallback)

- **Assumptions**:
  1. ⚠️ ASSUMPTION: SAML support targets generic SAML 2.0 SP configuration (IdP-agnostic: Azure AD, Okta, ADFS all supported)
  2. ⚠️ ASSUMPTION: LDAP group permissions use existing `vdctools` backend — UI wraps existing CLI functionality
  3. ⚠️ ASSUMPTION: LDAP connection is already configured on the existing LDAP page; this Epic adds group mapping only
  4. ⚠️ ASSUMPTION: A local admin fallback login is always preserved at `/serveradmin/local-login`
  5. ⚠️ ASSUMPTION: Available BLSS roles are: Administrator, Operator, Viewer `[TBD — confirm role list]`
  6. ⚠️ ASSUMPTION: LDAP group role changes take effect on next login (not retroactive to active sessions)
  7. ⚠️ ASSUMPTION: SAML library is `[TBD — confirm: mod_auth_mellon, Keycloak adapter, or custom]`

---

### Dependencies & Risks

| Item | Type (Dep/Risk) | Owner | Mitigation | Status |
|---|---|---|---|---|
| Existing LDAP configuration page (UI framework, layout) | Dep | Platform Engineering | Confirm page layout and extension points | `[TBD]` |
| vdctools LDAP group mapping CLI/API | Dep | Platform Engineering | Document vdctools interface for UI wrapping | `[TBD]` |
| SAML library selection (mod_auth_mellon, Keycloak, etc.) | Dep | Platform Engineering | Architecture decision required before Sprint 1 | `[TBD]` |
| Role-based access control framework | Dep | Platform Engineering | Confirm existing RBAC supports page-level access | `[TBD]` |
| SAML misconfiguration locks out all users | Risk | Platform Engineering | Mandatory local admin fallback; test-before-apply | Open |
| LDAP server unreachable → cannot search groups | Risk | Platform Engineering | Show connectivity error; existing mappings remain editable/viewable | Open |
| Encrypted secret storage mechanism | Dep | Platform Engineering | Confirm secret vault/encryption approach for on-prem | `[TBD]` |

---

### Operational Deliverables (required for on-prem)

- [ ] Installation/upgrade runbook updated (new SAML config page, LDAP mapping table migration)
- [ ] Backup/restore procedure updated (include SAML config and LDAP mappings in backup)
- [ ] Release notes entry drafted
- [ ] Lifecycle Management Services training/enablement updated (SAML setup workflow)
- [ ] Admin user guide: SAML configuration walkthrough
- [ ] Admin user guide: LDAP group mapping walkthrough
- [ ] Recovery procedure: authentication lockout recovery via local-login

---

### Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| S1 | Configure SAML SP Integration via System Settings UI | `[TBD]` | `[TBD]` |
| S2 | SAML Test-Before-Apply & Enable/Disable Toggle | `[TBD]` | `[TBD]` |
| S3 | LDAP Group-to-Role Mapping Table (View, Add, Remove) | `[TBD]` | `[TBD]` |
| S4 | LDAP Group Search from Active Directory | `[TBD]` | `[TBD]` |
| S5 | Authentication Lockout Prevention (Local Admin Fallback) | `[TBD]` | `[TBD]` |
| S6 | Audit Logging for Authentication Configuration Changes | `[TBD]` | `[TBD]` |

---

---

## STORY: S1 — Configure SAML SP Integration via System Settings UI

**Parent Epic**: Epic 1 — UI-Based SAML SSO Configuration & LDAP Group Permission Mapping
**BLSS Product**: Shared platform services (ServerAdmin > System Settings > Authentication Configuration)

**Description**
As an IT Admin,
I want to configure SAML Single Sign-On settings through the System Settings > Authentication Configuration > SAML page,
So that I can integrate BLSS with our corporate identity provider without requiring Linux CLI access or manual configuration file editing.

**Context & notes**:
- New sub-page/tab under existing Authentication Configuration in System Settings
- BLSS acts as SAML Service Provider (SP); customer provides IdP metadata
- Must support: IdP metadata upload (XML file or URL), Entity ID configuration, ACS URL, attribute mapping (username, email, groups)
- SAML library: `[TBD — confirm: mod_auth_mellon, Keycloak, or custom implementation]`
- Lockout prevention handled in Story S5

**Acceptance Criteria**

AC1:
  Given: Admin is logged into ServerAdmin with admin role
  When: Admin navigates to System Settings > Authentication Configuration > SAML
  Then: SAML configuration page is displayed showing current status (Enabled/Disabled, IdP name if configured)

AC2:
  Given: Admin is on SAML Configuration page
  When: Admin uploads IdP metadata (XML file) or enters IdP metadata URL
  Then: System parses metadata and displays extracted IdP information (Entity ID, SSO URL, certificate fingerprint)

AC3:
  Given: Admin has provided IdP metadata and configured SP Entity ID, ACS URL, and attribute mappings
  When: Admin clicks "Save"
  Then: SAML SP configuration is persisted; secrets stored encrypted at rest

AC4:
  Given: SAML is configured and enabled
  When: A user accesses BLSS login page
  Then: SSO login option is available; user is redirected to IdP and returned with valid assertion

AC5 (negative):
  Given: Admin uploads invalid or malformed IdP metadata
  When: System attempts to parse
  Then: Clear error message indicates the issue (malformed XML, missing required fields); configuration is not saved

AC6:
  Given: SAML configuration is saved
  When: Admin views the configuration page later
  Then: Certificates/secrets displayed masked; only metadata and non-secret fields are visible

**Tasks**
- [ ] Task 1: <Backend> Create API endpoints for SAML config CRUD: `GET/POST/PUT /api/serveradmin/settings/auth/saml`
- [ ] Task 2: <Backend> Implement IdP metadata parser (XML parsing, certificate extraction, field validation)
- [ ] Task 3: <Backend> Implement SAML SP configuration writer (integrate with underlying SAML library/service)
- [ ] Task 4: <Backend> Implement encrypted storage for SAML certificates and secrets
- [ ] Task 5: <Frontend> Implement SAML Configuration page under System Settings > Authentication Configuration: metadata upload/URL input, Entity ID, ACS URL, attribute mapping form
- [ ] Task 6: <Frontend> Implement metadata preview (show parsed IdP info before saving)
- [ ] Task 7: <Frontend> Implement secret masking (certificate fingerprint display only)
- [ ] Task 8: <Test> Integration tests: valid metadata upload, invalid metadata rejection, SSO login flow
- [ ] Task 9: <Security> Verify secrets encrypted at rest, not exposed in API responses
- [ ] Task 10: <Documentation> Update admin guide with SAML configuration workflow

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] UX mockups attached for SAML config page layout
- [ ] SAML library/implementation confirmed by architecture team
- [ ] Supported IdP list confirmed (Azure AD, Okta, ADFS, generic SAML 2.0)
- [ ] Attribute mapping requirements defined (username, email, groups)
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

## STORY: S2 — SAML Test-Before-Apply & Enable/Disable Toggle

**Parent Epic**: Epic 1 — UI-Based SAML SSO Configuration & LDAP Group Permission Mapping
**BLSS Product**: Shared platform services (ServerAdmin > System Settings > Authentication Configuration)

**Description**
As an IT Admin,
I want to test my SAML configuration before enabling it and toggle SAML on/off,
So that I can verify the integration works correctly without risking authentication lockout.

**Context & notes**:
- "Test Configuration" validates: IdP metadata reachable (if URL), certificate valid, SP config internally consistent
- Enable/disable toggle controls whether SSO login option appears on login page
- Disabling SAML does not delete configuration (can re-enable later)
- Critical safety feature — prevents lockout from untested SAML config

**Acceptance Criteria**

AC1:
  Given: SAML configuration has been saved (but not yet enabled)
  When: Admin clicks "Test Configuration"
  Then: System validates IdP metadata accessibility, certificate validity, and SP config consistency; displays pass/fail result with details

AC2:
  Given: SAML test passes
  When: Admin enables SAML via toggle
  Then: SSO login option appears on BLSS login page; users can authenticate via IdP

AC3:
  Given: SAML is enabled
  When: Admin disables SAML via toggle
  Then: SSO login option is removed from login page; saved configuration is preserved (not deleted)

AC4 (negative):
  Given: SAML test fails (e.g., IdP unreachable, certificate expired)
  When: Test result is displayed
  Then: Clear error details shown (specific failure reason); admin can still choose to enable at their own risk with explicit warning

AC5:
  Given: SAML enable/disable state changes
  When: Change is applied
  Then: Audit log records: user, timestamp, action (enabled/disabled)

**Tasks**
- [ ] Task 1: <Backend> Create test endpoint `POST /api/serveradmin/settings/auth/saml/test` (validate metadata URL reachability, cert validity, config consistency)
- [ ] Task 2: <Backend> Implement enable/disable endpoint `PUT /api/serveradmin/settings/auth/saml/status`
- [ ] Task 3: <Backend> Wire SAML status to login page SSO option (conditional rendering)
- [ ] Task 4: <Frontend> Implement "Test Configuration" button with loading state and result display (pass/fail with details)
- [ ] Task 5: <Frontend> Implement enable/disable toggle with confirmation for enabling
- [ ] Task 6: <Test> Integration tests: test pass scenario, test fail scenarios (unreachable, expired cert), toggle on/off
- [ ] Task 7: <Documentation> Document test and enable workflow

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] Test validation criteria confirmed (what constitutes pass/fail)
- [ ] Dependencies on S1 identified
- [ ] Estimated by team

---

## STORY: S3 — LDAP Group-to-Role Mapping Table (View, Add, Remove)

**Parent Epic**: Epic 1 — UI-Based SAML SSO Configuration & LDAP Group Permission Mapping
**BLSS Product**: Shared platform services (ServerAdmin > System Settings > LDAP Configuration)

**Description**
As an Administrator,
I want to view, add, and remove LDAP group-to-BLSS-role permission mappings in a table on the existing LDAP configuration page,
So that I can control access based on Active Directory groups without running CLI commands (vdctools).

**Context & notes**:
- Enhancement to the **existing** LDAP configuration page (add a "Group Permissions" section/table)
- Wraps existing `vdctools` CLI functionality in a UI layer
- Maps AD/LDAP groups → BLSS application roles (Administrator, Operator, Viewer)
- Changes take effect on next user login (not retroactive to active sessions) `[TBD — confirm]`

**Acceptance Criteria**

AC1:
  Given: Admin is logged in with admin role
  When: Admin navigates to System Settings > LDAP Configuration > Group Permissions section
  Then: Current group-to-role mappings are displayed in a table (columns: LDAP Group DN, BLSS Role, Actions)

AC2:
  Given: Admin is on Group Permissions section
  When: Admin clicks "Add Mapping", selects a group (from search — Story S4), and assigns a role
  Then: Mapping is persisted via vdctools; table updates to show new entry

AC3:
  Given: Admin views existing mappings
  When: Admin clicks "Remove" on a mapping row and confirms in modal
  Then: Mapping is deleted; affected users lose that role on next login

AC4:
  Given: Mapping table has entries
  When: Admin views the table
  Then: Each row shows: full group DN, assigned role (with color-coded badge), remove action button

AC5:
  Given: Any group mapping change (add/remove) is made
  When: Change is saved
  Then: Audit log records: user, timestamp, LDAP group, role, action (add/remove)

**Tasks**
- [ ] Task 1: <Backend> Create API endpoints: `GET/POST/DELETE /api/serveradmin/settings/ldap/mappings`
- [ ] Task 2: <Backend> Integrate with vdctools backend for reading/writing group-role mappings
- [ ] Task 3: <Frontend> Implement "Group Permissions" section on existing LDAP configuration page: mappings table with group DN, role badge, remove button
- [ ] Task 4: <Frontend> Implement "Add Mapping" flow: group selector + role dropdown + save
- [ ] Task 5: <Frontend> Implement removal confirmation modal
- [ ] Task 6: <Test> Integration tests: view mappings, add mapping, remove mapping, vdctools integration
- [ ] Task 7: <Documentation> Update admin guide with LDAP group mapping workflow

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] UX mockups for Group Permissions table on LDAP page
- [ ] vdctools API/CLI interface documented
- [ ] Available BLSS roles confirmed
- [ ] Session impact confirmed (next login vs. immediate)
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

## STORY: S4 — LDAP Group Search from Active Directory

**Parent Epic**: Epic 1 — UI-Based SAML SSO Configuration & LDAP Group Permission Mapping
**BLSS Product**: Shared platform services (ServerAdmin > System Settings > LDAP Configuration)

**Description**
As an Administrator,
I want to search for available LDAP/AD groups when adding a new group mapping,
So that I can find and select the correct group without manually typing the full distinguished name.

**Context & notes**:
- Uses existing LDAP connection settings from the LDAP configuration page
- Provides autocomplete/search-as-you-type for group names
- Must handle LDAP server unreachable gracefully
- Search should be scoped to the configured Base DN

**Acceptance Criteria**

AC1:
  Given: Admin is adding a new group mapping
  When: Admin types 2+ characters in the group search field
  Then: System queries LDAP directory and returns matching groups (filtered by name)

AC2:
  Given: Search results are displayed
  When: Admin clicks/selects a group from the results
  Then: Group DN is populated in the mapping form

AC3 (negative):
  Given: LDAP server is unreachable
  When: Admin attempts to search for groups
  Then: System displays connectivity error with troubleshooting guidance; admin can manually enter group DN as fallback

AC4:
  Given: Large AD environment with many groups
  When: Admin searches
  Then: Results are limited to first 50 matches with indication if more exist; search can be refined

**Tasks**
- [ ] Task 1: <Backend> Create LDAP group search endpoint: `GET /api/serveradmin/settings/ldap/groups?search=<query>&limit=50`
- [ ] Task 2: <Backend> Implement LDAP search logic using existing connection config (filter by CN, scoped to Base DN)
- [ ] Task 3: <Backend> Handle LDAP connectivity errors gracefully (timeout, unreachable, auth failure)
- [ ] Task 4: <Frontend> Implement search-as-you-type input with dropdown results
- [ ] Task 5: <Frontend> Implement fallback: manual DN entry when search unavailable
- [ ] Task 6: <Frontend> Implement LDAP connectivity status indicator
- [ ] Task 7: <Test> Integration tests: search returns results, empty results, LDAP unreachable, manual fallback

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] LDAP connection reuse approach confirmed (use existing page's connection config)
- [ ] Search scope confirmed (Base DN, subtree search depth)
- [ ] Dependencies on existing LDAP configuration page confirmed
- [ ] Estimated by team

---

## STORY: S5 — Authentication Lockout Prevention (Local Admin Fallback)

**Parent Epic**: Epic 1 — UI-Based SAML SSO Configuration & LDAP Group Permission Mapping
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As an On-Prem IT / System Admin,
I want a guaranteed fallback local admin login that works even if SAML or LDAP is misconfigured,
So that I can always recover access to ServerAdmin without requiring CLI intervention.

**Context & notes**:
- Critical safety mechanism — without this, a SAML/LDAP misconfiguration could permanently lock out all users
- Local admin account bypasses SAML/LDAP authentication
- Accessible via dedicated URL: `/serveradmin/local-login`
- Cannot be disabled by any user (always-on safety)

**Acceptance Criteria**

AC1:
  Given: SAML is enabled and IdP becomes unreachable
  When: Admin navigates to `/serveradmin/local-login`
  Then: Local admin login form is displayed; admin can authenticate with local credentials

AC2:
  Given: LDAP is configured but LDAP server is unreachable
  When: Local admin attempts to log in via fallback URL
  Then: Local admin can authenticate regardless of LDAP status

AC3:
  Given: Admin is logged in via local fallback
  When: Admin navigates to Authentication Configuration
  Then: Admin can reconfigure or disable SAML/LDAP to restore normal access

AC4:
  Given: Any user or admin
  When: Attempt is made to disable local admin fallback
  Then: System prevents this action (always-on); displays explanatory message

AC5:
  Given: Fallback login is used
  When: Admin authenticates via local-login
  Then: Audit log records the fallback login event

**Tasks**
- [ ] Task 1: <Backend> Implement local admin auth bypass for SAML/LDAP (dedicated endpoint `/serveradmin/local-login`)
- [ ] Task 2: <Backend> Ensure local admin cannot be disabled programmatically or via UI
- [ ] Task 3: <Frontend> Implement fallback login page at dedicated URL (simple login form, not dependent on SAML/LDAP)
- [ ] Task 4: <Security> Penetration test: verify fallback cannot be exploited to bypass SAML when SAML is functioning correctly
- [ ] Task 5: <Test> Integration tests: SAML failure → fallback works, LDAP unreachable → local admin works
- [ ] Task 6: <Documentation> Document recovery procedure for authentication lockout scenarios

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] Fallback URL path confirmed (`/serveradmin/local-login`)
- [ ] Security review of fallback mechanism completed
- [ ] Local admin password management approach confirmed (initial setup, reset procedure)
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

## STORY: S6 — Audit Logging for Authentication Configuration Changes

**Parent Epic**: Epic 1 — UI-Based SAML SSO Configuration & LDAP Group Permission Mapping
**BLSS Product**: Shared platform services (ServerAdmin)

**Description**
As an On-Prem IT / System Admin,
I want all authentication configuration changes to be recorded in an audit log,
So that I can track who changed SAML/LDAP settings, when, and support compliance requirements.

**Context & notes**:
- Applies to: SAML configuration changes, SAML enable/disable, LDAP group mapping add/remove
- Must log: user identity, timestamp (ISO 8601), action type, configuration category, result (success/failure)
- Secrets/credentials must NOT be logged
- Integrates with existing ServerAdmin audit infrastructure (if available) `[TBD — confirm]`

**Acceptance Criteria**

AC1:
  Given: Any authentication configuration change is made (SAML config, SAML toggle, LDAP mapping add/remove)
  When: Change is applied (success or failure)
  Then: Audit log entry created with: user, timestamp, category (SAML/LDAP), action, result, non-secret details

AC2:
  Given: A SAML certificate or LDAP bind password is changed
  When: Audit log entry is created
  Then: Entry records that a credential was changed but does NOT log the credential value

AC3:
  Given: Fallback local-login is used
  When: Authentication occurs
  Then: Audit log records the fallback login event (user, timestamp, method=local-fallback)

AC4:
  Given: Non-admin user attempts to access authentication configuration
  When: Access is denied
  Then: Unauthorized access attempt is logged

**Tasks**
- [ ] Task 1: <Backend> Implement audit logging middleware for all authentication configuration API endpoints
- [ ] Task 2: <Backend> Implement secret-scrubbing logic (prevent credentials from being logged)
- [ ] Task 3: <Backend> Log fallback login events and access denial events
- [ ] Task 4: <Test> Integration tests: verify all auth config endpoints produce audit entries, secrets are scrubbed
- [ ] Task 5: <Documentation> Document audit log schema for authentication events

**Definition of Ready checklist**
- [ ] AC reviewed and agreed by PO + dev team
- [ ] Audit log storage mechanism confirmed (existing infrastructure or new)
- [ ] Retention policy confirmed
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

---

## Validation Checklist (Epic 1)

- [x] All AC are testable
- [x] NFRs are quantified
- [x] BLSS product explicitly identified (Shared platform services / ServerAdmin > System Settings)
- [x] Scope boundaries are explicit
- [x] Cross-product dependencies identified (auth layer shared across all products)
- [x] On-prem operational impact assessed
- [x] Dependencies are identified (vdctools, SAML library, RBAC, LDAP connection)
- [x] No invented features, UI labels, device models, or protocol details
- [x] Terminology matches BLSS product glossary
- [x] Placeholders marked with `[TBD]`
- [x] Operational deliverables included
- [x] Service enablement (Lifecycle Management Services) considered

# BDCSPM-49336 — Apply Triggers to a Group of Devices

## EPIC 3: Rule-Based Bulk Trigger Application, Audit & Permissions

**BLSS Product(s)**: Shared platform services (Monitoring / Triggers subsystem) — cross-product (DCPM, Distributed IT Performance Management, Brightlayer Power)
**Summary** (1-2 sentences)
Provide a rule-based mechanism for applying triggers across multiple device groups simultaneously, implement comprehensive audit logging for all trigger configuration changes (group/model/device-level), and enforce role-based permission controls for group and model-level trigger management.

---

### Description

- **Problem statement**: While Epics 1 and 2 address single-group and single-model trigger management, customers with many device groups (e.g., multiple closets across sites) need a way to apply the same trigger to several groups at once without repeating the operation. Additionally, trigger changes affecting hundreds of devices must be auditable for compliance, and only authorized administrators should be able to modify triggers at group/model level (higher blast radius than device-level changes).

- **Proposed solution**:
  1. **Rule-based bulk application**: A "Trigger Rules" feature where an admin defines a trigger once and selects multiple device groups to apply it to. Rules can be managed (created, edited, deleted) from a dedicated "Trigger Rules" page.
  2. **Audit logging**: All trigger CRUD operations at device, group, model, and rule levels are logged with: timestamp, user identity, action, scope, before/after values.
  3. **Permission controls**: Only users with the "Trigger Admin" role (or equivalent) can create/modify/delete group-level, model-level, and rule-based triggers. Non-admin users see triggers in read-only mode. Device-level trigger management follows existing permission model.

- **User personas**: Data Center Facility Manager (rules), On-Prem IT / System Admin (audit, permissions), Eaton Field Service Engineer (audit review)

- **Business value**: Enables "configure once, apply to many groups" workflow for large multi-site deployments. Satisfies compliance requirements for auditable configuration changes. Reduces risk of accidental fleet-wide trigger misconfiguration by untrained users.

- **Key workflows**:
  1. Admin navigates to Triggers → Trigger Rules → Create Rule
  2. Admin defines a trigger (attribute, condition, threshold, severity)
  3. Admin selects target device groups: Closet A, Closet B, Closet C
  4. Admin saves the rule → trigger is applied to all devices in all selected groups
  5. Compliance officer reviews audit log for all trigger changes in the past 30 days
  6. Non-admin user views Device Group Triggers tab → sees triggers read-only → attempts Add → receives "Insufficient Permissions"

- **Cross-product touchpoints**: Audit logging integrates with the shared BLSS audit/logging infrastructure. Permission model integrates with existing RBAC (Active Directory / LDAP roles).

- **UX/UI references**: `[TBD — wireframes/mockups to be produced during Design Epic BDCSPM-66978]`

- **On-prem operational impact**:
  - Install/upgrade impact: New DB tables for trigger rules and audit log; new permission/role definition in RBAC configuration.
  - Sizing impact: Audit log table will grow with trigger activity; retention policy needed. Estimate `[TBD]` rows/day for typical deployment.
  - Network impact: None.
  - Backup impact: Audit log table included in backup; may increase backup size over time.

---

### Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Performance | Rule application across 5 groups (total 500 devices) | ≤ 30 seconds end-to-end | P1 |
| Performance | Audit log query (last 30 days, filtered by scope) | < 3s at P95 on Medium tier | P1 |
| Reliability | Audit log must never lose entries (write-ahead) | Zero audit log gaps | P0 |
| Security | Permission check latency (per API call) | < 50ms overhead | P1 |
| Security | Permission enforcement must be server-side (not UI-only) | API rejects unauthorized requests with 403 | P0 |
| Scalability | Audit log must support 1M+ entries without performance degradation | Query < 3s with proper indexing | P2 |
| Compliance | Audit log entries must be immutable (append-only, no delete/update by users) | Tamper-proof | P0 |
| Usability | Insufficient Permissions message must be clear and actionable | Shows required role and contact for access | P2 |

---

### Acceptance Criteria (Epic-level)

- [ ] AC1: An admin can create a trigger rule, select multiple device groups, and have the trigger applied to all devices in all selected groups.
- [ ] AC2: An admin can edit a trigger rule (change threshold or add/remove groups); changes propagate to affected devices.
- [ ] AC3: An admin can delete a trigger rule; the trigger is retracted from all devices in all affected groups.
- [ ] AC4: Every trigger creation, modification, deletion, override, and revert action (at device, group, model, or rule level) is recorded in the audit log with timestamp, user, action, scope, and before/after values.
- [ ] AC5: The audit log is queryable by time range, user, action type, scope (device/group/model/rule), and specific entity.
- [ ] AC6: Only users with the "Trigger Admin" role can create/edit/delete triggers at group, model, or rule level.
- [ ] AC7: Non-admin users viewing the Triggers tab on a group or model page see triggers in read-only mode; Add/Edit/Delete actions are not available.
- [ ] AC8: Non-admin users attempting to call group/model/rule trigger APIs directly receive a 403 Forbidden response with a clear error message.
- [ ] AC9: Existing device-level trigger permissions are unchanged (existing permission model applies).

---

### Scope

- **In scope**:
  - Trigger Rules feature: create/edit/delete rules that apply a trigger to multiple device groups
  - Trigger Rules management page (list, create, edit, delete rules)
  - Rule propagation (apply trigger to all devices in all selected groups)
  - Comprehensive audit logging for all trigger operations (CRUD at all levels)
  - Audit log storage (append-only DB table)
  - Audit log query API (filtered by time, user, action, scope)
  - Audit log viewer UI (admin-only)
  - Role-based permission enforcement for group/model/rule trigger management
  - "Insufficient Permissions" UX for unauthorized users
  - Server-side permission validation (API layer)

- **Out of scope**:
  - Rule-based application by criteria other than device group (e.g., by location, by custom tag) — future enhancement
  - Audit log export to external SIEM/syslog — future enhancement
  - Audit log retention/purge policy automation — future enhancement (manual DB maintenance for now)
  - Custom role creation (uses existing RBAC roles from AD/LDAP)
  - Changes to device-level trigger permissions

- **Assumptions**:
  1. ⚠️ ASSUMPTION: BLSS has an existing RBAC system integrated with Active Directory / LDAP that supports role-based permission checks.
  2. ⚠️ ASSUMPTION: A "Trigger Admin" role (or equivalent admin-level role) already exists in the RBAC system, or can be added as a new role with minimal effort.
  3. ⚠️ ASSUMPTION: The audit log is stored in PostgreSQL (same DB as application data). External audit log forwarding is out of scope for this Epic.
  4. ⚠️ ASSUMPTION: "Trigger Rules" are a new first-class entity (not to be confused with monitoring templates or alarm policies).
  5. ⚠️ ASSUMPTION: A trigger rule can target device groups only (not individual devices or models). Model-level triggers are managed directly on the Model page (Epic 1).

---

### Dependencies & Risks

| Item | Type | BLSS Product | Owner | Mitigation | Status |
|---|---|---|---|---|---|
| Epic 1 must be complete (group-level triggers exist) | Dep | Shared platform | Dev team | Sequence: Epic 1 → Epic 2 → Epic 3 | Planned |
| Epic 2 priority resolution must be complete (rules use same propagation) | Dep | Shared platform | Dev team | Rules leverage S1.2 propagation service | Planned |
| RBAC system must support new permission check | Dep | Shared platform | `[TBD]` | Confirm RBAC extensibility with platform team | ⚠️ NEEDS VALIDATION |
| Audit log table growth (disk space) | Risk | Shared platform | Operations | Define retention guidance in sizing guide; index properly | Open |
| Rule propagation failure across multiple groups (partial success) | Risk | Shared platform | Architecture | Implement per-group status tracking; allow retry for failed groups | Open |

---

### Operational Deliverables (required for on-prem)

- [ ] Installation/upgrade runbook updated (audit log table, rule table, RBAC role configuration)
- [ ] Sizing guide updated (audit log disk usage projections by deployment size)
- [ ] Backup/restore procedure updated (audit log table in backup scope)
- [ ] Firewall/port documentation: N/A
- [ ] Release notes entry drafted (rules feature, audit, permissions)
- [ ] Lifecycle Services training/enablement updated (rules workflow, audit log review, permission setup)
- [ ] RBAC configuration guide updated (how to assign "Trigger Admin" role)

---

### Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| S3.1 | Backend: Rule engine for bulk trigger application across multiple groups | 8 | `[TBD]` |
| S3.2 | UI: Create/edit/delete trigger rules with group selection | 5 | `[TBD]` |
| S3.3 | UI: Rule summary view with applied groups and trigger definitions | 3 | `[TBD]` |
| S3.4 | Backend: Audit logging for all group/model/rule trigger changes | 5 | `[TBD]` |
| S3.5 | Backend: Role-based permission check for group/model trigger management | 5 | `[TBD]` |
| S3.6 | UI: "Insufficient Permissions" UX for non-admin users | 3 | `[TBD]` |
| S3.7 | Backend & UI: Audit log query API and viewer | 3 | `[TBD]` |

---

## Stories

---

### STORY: S3.1 — Backend: Rule Engine for Bulk Trigger Application Across Multiple Groups

**Parent Epic**: BDCSPM-49336 Epic 3 — Rule-Based Bulk Trigger Application, Audit & Permissions
**BLSS Product**: Shared platform services (Triggers subsystem)

**Description**
As a Data Center Facility Manager,
I want to create a trigger rule that applies a trigger to multiple device groups at once,
So that I can configure the same threshold for devices across several closets/locations without repeating the operation per group.

**Context & notes**:
- A "Trigger Rule" is a new entity: {rule_id, trigger_definition (attribute, condition, threshold, severity), target_group_ids[], enabled, created_by, created_at}
- On rule creation, the system iterates over all target groups and applies the trigger using the existing propagation service (S1.2)
- On rule edit (threshold change), all affected devices are updated
- On rule delete, the trigger is retracted from all devices in all target groups
- On group addition to an existing rule, the trigger is applied to the new group's devices
- On group removal from a rule, the trigger is retracted from that group's devices
- Matches Scenario 4 from BDCSPM-49336: "Bulk Apply via Rule Selection"

**Acceptance Criteria**

AC1:
  Given: An admin creates a rule: "Humidity > 70%" applied to groups [Closet A, Closet B]
  When: The rule is saved
  Then: The Humidity trigger is applied to all applicable devices in both Closet A and Closet B; a summary shows results per group.

AC2:
  Given: An existing rule applies "Humidity > 70%" to [Closet A, Closet B]
  When: The admin adds "Closet C" to the rule
  Then: The trigger is applied to devices in Closet C; Closet A and B devices are unaffected.

AC3:
  Given: An existing rule applies a trigger to 3 groups
  When: The admin updates the threshold from 70% to 75%
  Then: All devices in all 3 groups are updated to the new threshold value.

AC4:
  Given: An existing rule applies a trigger to [Closet A, Closet B, Closet C]
  When: The admin removes "Closet B" from the rule
  Then: The trigger is retracted from Closet B devices only; Closet A and C devices retain the trigger.

AC5:
  Given: An admin deletes a rule entirely
  When: Deletion is confirmed
  Then: The trigger is retracted from all devices in all groups previously associated with the rule.

AC6 (edge case):
  Given: Rule application to 5 groups, one group fails (e.g., DB error during propagation to that group)
  When: The partial failure occurs
  Then: Successfully applied groups retain their triggers; the failed group is reported; the admin can retry the failed group.

AC7 (edge case):
  Given: A device in one of the target groups already has the same trigger (from a different source: model or device-level)
  When: The rule is applied
  Then: The rule-applied trigger is treated as a group-level trigger for priority resolution purposes (Device > Group/Rule > Model).

**Tasks**
- [ ] Task 1: <Data/schema> Design trigger_rule table (id, attribute, condition, threshold, severity, enabled, created_by, created_at, updated_at)
- [ ] Task 2: <Data/schema> Design trigger_rule_group junction table (rule_id, group_id)
- [ ] Task 3: <Data/schema> Write DB migration scripts (forward + rollback)
- [ ] Task 4: <Backend> Implement TriggerRuleService (create, update, delete, addGroup, removeGroup)
- [ ] Task 5: <Backend> Implement bulk propagation orchestrator (iterate groups, call S1.2 per group, aggregate results)
- [ ] Task 6: <Backend> Implement per-group status tracking for partial failure handling
- [ ] Task 7: <Backend> Create REST API endpoints: POST/GET/PUT/DELETE /api/v2/trigger-rules, PUT /api/v2/trigger-rules/{ruleId}/groups
- [ ] Task 8: <Test> Unit tests for rule CRUD, bulk propagation, partial failure
- [ ] Task 9: <Test> Integration test: create rule with 3 groups → verify propagation → update threshold → verify update → delete rule → verify retraction
- [ ] Task 10: <Test> Performance test: apply rule to 5 groups × 100 devices each (500 total)

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] API contract defined (rule CRUD + group management endpoints)
- [ ] Cross-product dependencies: S1.2 propagation service
- [ ] On-prem install/upgrade impact: new DB tables
- [ ] Sizing impact: minimal metadata
- [ ] Dependencies: Epic 1 (S1.1, S1.2), Epic 2 (S2.1 for priority resolution of rule-applied triggers)
- [ ] NFRs applicable: performance (30s for 500 devices), reliability (partial failure handling)
- [ ] Estimated by team

---

### STORY: S3.2 — UI: Create/Edit/Delete Trigger Rules with Group Selection

**Parent Epic**: BDCSPM-49336 Epic 3 — Rule-Based Bulk Trigger Application, Audit & Permissions
**BLSS Product**: Shared platform services (UI layer)

**Description**
As a Data Center Facility Manager,
I want a UI to create, edit, and delete trigger rules with a multi-select for target device groups,
So that I can manage bulk trigger configurations visually without using the API directly.

**Context & notes**:
- New navigation item: Triggers → Trigger Rules (or integrated into existing Triggers page)
- "Create Rule" form: Define trigger (attribute, condition, threshold, severity) + multi-select device groups
- Group selector shows all available device groups with device count
- Edit flow allows changing trigger values and adding/removing groups
- Delete flow shows confirmation with count of affected groups and devices
- Link to UX mockup: `[TBD — pending Design Epic BDCSPM-66978]`

**Acceptance Criteria**

AC1:
  Given: An admin navigates to Triggers → Trigger Rules → Create Rule
  When: The creation form opens
  Then: The form has two sections: (1) Trigger definition (Attribute, Condition, Threshold, Severity) and (2) Target groups (multi-select with search/filter, showing group names and device counts).

AC2:
  Given: The admin fills in trigger details and selects 3 groups
  When: They click "Save Rule"
  Then: The rule is created; propagation begins; a progress/result summary is shown.

AC3:
  Given: An existing rule is displayed in the rules list
  When: The admin clicks "Edit"
  Then: The edit form is pre-populated; the admin can modify trigger values and add/remove groups.

AC4:
  Given: The admin clicks "Delete" on a rule
  When: The confirmation dialog appears
  Then: It shows "This will remove the trigger from X devices across Y groups. Are you sure?"

AC5:
  Given: The group selector is displayed
  When: The admin types a filter query
  Then: Groups are filtered by name; each shows (N devices) count.

**Tasks**
- [ ] Task 1: <Frontend> Add "Trigger Rules" navigation item and list page
- [ ] Task 2: <Frontend> Implement "Create Rule" form with trigger definition section
- [ ] Task 3: <Frontend> Implement multi-select device group picker (search, filter, device count display)
- [ ] Task 4: <Frontend> Implement Edit Rule flow (pre-populate, modify, save)
- [ ] Task 5: <Frontend> Implement Delete Rule flow (confirmation with impact summary)
- [ ] Task 6: <Frontend> Wire to API: POST/PUT/DELETE /api/v2/trigger-rules
- [ ] Task 7: <Frontend> Display propagation progress/result summary after save
- [ ] Task 8: <Test> Component tests for form, group picker, confirmation dialog
- [ ] Task 9: <Test> E2E test: create rule → verify → edit → verify → delete → verify

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — `[TBD]`
- [ ] API contract defined (S3.1 endpoints)
- [ ] Dependencies: S3.1 (backend rule engine)
- [ ] NFRs applicable: usability
- [ ] Estimated by team

---

### STORY: S3.3 — UI: Rule Summary View with Applied Groups and Trigger Definitions

**Parent Epic**: BDCSPM-49336 Epic 3 — Rule-Based Bulk Trigger Application, Audit & Permissions
**BLSS Product**: Shared platform services (UI layer)

**Description**
As a Data Center Facility Manager,
I want a summary view of all trigger rules showing which groups each rule targets and the trigger definition,
So that I can quickly review the bulk trigger configuration across my environment.

**Context & notes**:
- List/table view of all trigger rules
- Columns: Rule name/ID, Attribute, Condition, Threshold, Severity, Target Groups (count + names), Total Devices Affected, Enabled/Disabled, Last Modified
- Expandable row or detail panel showing individual group application status
- Link to UX mockup: `[TBD]`

**Acceptance Criteria**

AC1:
  Given: 5 trigger rules exist in the system
  When: The admin navigates to the Trigger Rules page
  Then: All 5 rules are displayed in a table with key information visible.

AC2:
  Given: A rule targets 3 groups
  When: The admin views the rule row
  Then: The "Target Groups" column shows the count (3) and a tooltip/expandable list with group names.

AC3:
  Given: A rule is disabled
  When: Displayed in the list
  Then: The rule row shows a "Disabled" status badge; the trigger is not actively applied but the definition is retained.

AC4:
  Given: The admin clicks on a rule row
  When: The detail panel/page opens
  Then: Full details are shown: trigger definition, all target groups with per-group device count and propagation status (applied/partial/failed).

**Tasks**
- [ ] Task 1: <Frontend> Implement Trigger Rules list page with data table
- [ ] Task 2: <Frontend> Implement rule detail panel (expand or navigate to detail page)
- [ ] Task 3: <Frontend> Show per-group propagation status in detail view
- [ ] Task 4: <Frontend> Wire to API: GET /api/v2/trigger-rules (list) and GET /api/v2/trigger-rules/{ruleId} (detail)
- [ ] Task 5: <Test> Component tests for list and detail views
- [ ] Task 6: <Test> E2E test: navigate to rules → verify list → expand detail → verify per-group info

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — `[TBD]`
- [ ] API contract defined
- [ ] Dependencies: S3.1 (backend), S3.2 (rules exist to display)
- [ ] NFRs applicable: usability, performance (list load < 2s)
- [ ] Estimated by team

---

### STORY: S3.4 — Backend: Audit Logging for All Group/Model/Rule Trigger Changes

**Parent Epic**: BDCSPM-49336 Epic 3 — Rule-Based Bulk Trigger Application, Audit & Permissions
**BLSS Product**: Shared platform services (Audit subsystem)

**Description**
As an On-Prem IT / System Admin,
I want all trigger configuration changes (create, update, delete, override, revert) at every level (device, group, model, rule) to be recorded in an immutable audit log,
So that I can satisfy compliance requirements and troubleshoot configuration issues.

**Context & notes**:
- Audit log table: trigger_audit_log (id, timestamp, user_id, user_name, action, scope, scope_id, entity_type, entity_id, before_value JSON, after_value JSON, metadata JSON)
- Actions: CREATE, UPDATE, DELETE, OVERRIDE, REVERT, PROPAGATE
- Scopes: DEVICE, GROUP, MODEL, RULE
- Append-only table (no UPDATE/DELETE operations by application)
- Must log the initiating user (from session/auth context)
- Matches Capability NFR: "Changes should be logged for compliance"
- ⚠️ ASSUMPTION: Audit log is stored in the same PostgreSQL database. External forwarding is out of scope.

**Acceptance Criteria**

AC1:
  Given: An admin creates a group-level trigger "Temperature > 85°F" on group "Closet A"
  When: The create operation succeeds
  Then: An audit log entry is written: {action: CREATE, scope: GROUP, scope_id: closet_a_id, entity_type: TRIGGER, after_value: {attribute: Temperature, condition: >, threshold: 85, severity: ...}, user: admin_name, timestamp: ...}.

AC2:
  Given: An admin updates a model-level trigger threshold from 80°F to 82°F
  When: The update operation succeeds
  Then: An audit log entry is written with both before_value and after_value showing the change.

AC3:
  Given: An admin overrides a group trigger at device level
  When: The override is saved
  Then: An audit log entry is written: {action: OVERRIDE, scope: DEVICE, metadata: {overridden_source: GROUP, overridden_source_id: group_id, overridden_value: ...}}.

AC4:
  Given: An admin deletes a trigger rule affecting 3 groups
  When: The delete operation succeeds
  Then: An audit log entry is written: {action: DELETE, scope: RULE, entity_id: rule_id, before_value: {full rule definition}, after_value: null, metadata: {affected_groups: [...], affected_device_count: N}}.

AC5:
  Given: Application code attempts to UPDATE or DELETE a row in the audit log table
  When: The operation is attempted
  Then: The operation fails (DB constraints or application-level guard prevent modification of audit entries).

AC6 (edge case):
  Given: A bulk propagation operation affects 500 devices
  When: Propagation completes
  Then: A single summary audit entry is logged (not 500 individual entries); metadata includes affected_device_count and skipped_device_count.

**Tasks**
- [ ] Task 1: <Data/schema> Design trigger_audit_log table (append-only, with indexes on timestamp, user_id, scope, action)
- [ ] Task 2: <Data/schema> Write DB migration script; add DB trigger/policy to prevent UPDATE/DELETE on audit table
- [ ] Task 3: <Backend> Implement AuditLogService (logTriggerEvent method, accepts structured audit entry)
- [ ] Task 4: <Backend> Integrate audit logging into all trigger CRUD operations (device, group, model, rule)
- [ ] Task 5: <Backend> Integrate audit logging into override and revert operations
- [ ] Task 6: <Backend> Integrate audit logging into propagation operations (summary entry)
- [ ] Task 7: <Backend> Ensure audit log captures user identity from authentication context
- [ ] Task 8: <Test> Unit tests for audit log service (all action types, immutability)
- [ ] Task 9: <Test> Integration test: perform trigger operations → verify corresponding audit entries

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Audit log schema confirmed by architecture
- [ ] Immutability enforcement mechanism confirmed (DB trigger vs. application guard)
- [ ] On-prem install/upgrade impact: new table, DB migration
- [ ] Sizing impact: audit log growth estimates needed
- [ ] Dependencies: Epic 1 (trigger CRUD), Epic 2 (override/revert)
- [ ] NFRs applicable: reliability (zero loss), compliance (immutable), scalability (1M+ entries)
- [ ] Estimated by team

---

### STORY: S3.5 — Backend: Role-Based Permission Check for Group/Model Trigger Management

**Parent Epic**: BDCSPM-49336 Epic 3 — Rule-Based Bulk Trigger Application, Audit & Permissions
**BLSS Product**: Shared platform services (Security / RBAC)

**Description**
As an On-Prem IT / System Admin,
I want only authorized users (with "Trigger Admin" role or equivalent) to create, modify, or delete triggers at the group, model, or rule level,
So that high-impact trigger changes (affecting many devices) are restricted to qualified personnel.

**Context & notes**:
- Enforce at API layer (server-side) — UI enforcement alone is insufficient
- Integrates with existing BLSS RBAC system (Active Directory / LDAP role mapping)
- Unauthorized API calls return 403 Forbidden with descriptive error message
- Read access (viewing triggers) remains open to all authenticated users
- Device-level trigger permissions follow existing model (no change)
- Matches Scenario 6 from BDCSPM-49336: Permissions
- ⚠️ ASSUMPTION: BLSS has an existing permission checking mechanism (e.g., Spring Security, middleware filter) that can be extended with new permission rules.

**Acceptance Criteria**

AC1:
  Given: A user with "Trigger Admin" role makes a POST request to /api/v2/device-groups/{groupId}/triggers
  When: The permission check runs
  Then: The request proceeds normally (200/201 response).

AC2:
  Given: A user WITHOUT "Trigger Admin" role makes a POST request to /api/v2/device-groups/{groupId}/triggers
  When: The permission check runs
  Then: The request is rejected with 403 Forbidden: {error: "Insufficient permissions", required_role: "Trigger Admin", message: "Group-level trigger management requires the Trigger Admin role. Contact your system administrator."}.

AC3:
  Given: A user WITHOUT "Trigger Admin" role makes a GET request to /api/v2/device-groups/{groupId}/triggers
  When: The permission check runs
  Then: The request proceeds normally (read access allowed for all authenticated users).

AC4:
  Given: Permission is checked for model-level and rule-level trigger write operations
  When: An unauthorized user attempts any write (POST/PUT/DELETE) on model triggers or trigger rules
  Then: All are rejected with 403 Forbidden.

AC5:
  Given: A user has "Trigger Admin" role in Active Directory
  When: Their BLSS session is established
  Then: The role is correctly mapped and recognized by the trigger permission check.

AC6 (edge case):
  Given: Device-level trigger creation by a non-admin user
  When: They create a trigger on their own device
  Then: Existing device-level permissions apply (this story does NOT restrict device-level trigger management).

**Tasks**
- [ ] Task 1: <Backend> Define "TRIGGER_ADMIN" permission constant and map to AD/LDAP role
- [ ] Task 2: <Backend> Implement permission check interceptor/filter for group trigger endpoints
- [ ] Task 3: <Backend> Implement permission check for model trigger endpoints
- [ ] Task 4: <Backend> Implement permission check for trigger rule endpoints
- [ ] Task 5: <Backend> Implement structured 403 error response (with required_role, message)
- [ ] Task 6: <Backend> Ensure GET endpoints remain accessible to all authenticated users
- [ ] Task 7: <Documentation> Update RBAC configuration guide with new "Trigger Admin" role setup instructions
- [ ] Task 8: <Test> Unit tests for permission checks (authorized, unauthorized, read vs. write)
- [ ] Task 9: <Test> Integration test: attempt group trigger CRUD as admin → success; as non-admin → 403

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] RBAC system capability confirmed (can add new permission/role)
- [ ] AD/LDAP role mapping approach confirmed
- [ ] On-prem install/upgrade impact: new role definition in RBAC config
- [ ] Dependencies: existing RBAC infrastructure
- [ ] NFRs applicable: security (server-side enforcement), performance (< 50ms overhead)
- [ ] Estimated by team

---

### STORY: S3.6 — UI: "Insufficient Permissions" UX for Non-Admin Users

**Parent Epic**: BDCSPM-49336 Epic 3 — Rule-Based Bulk Trigger Application, Audit & Permissions
**BLSS Product**: Shared platform services (UI layer)

**Description**
As a non-admin user (e.g., Operations / NOC Engineer with read-only access),
I want to see a clear "Insufficient Permissions" message when I attempt to modify group/model/rule triggers,
So that I understand why I cannot make changes and know who to contact for access.

**Context & notes**:
- UI checks user role on page load and adjusts available actions
- Add/Edit/Delete buttons hidden or disabled for non-admin users on group/model/rule trigger pages
- If a non-admin somehow triggers a write API call (e.g., direct URL), the 403 response is caught and displayed
- Error message includes: what permission is needed, who to contact (system administrator)
- Matches Scenario 6 from BDCSPM-49336

**Acceptance Criteria**

AC1:
  Given: A non-admin user navigates to the Triggers tab of a Device Group
  When: The page loads
  Then: The "Add Trigger", "Edit", and "Delete" action buttons are not visible (or visibly disabled with tooltip "Requires Trigger Admin role").

AC2:
  Given: A non-admin user navigates to the Trigger Rules page
  When: The page loads
  Then: The "Create Rule" button is not visible (or disabled); existing rules are displayed read-only.

AC3:
  Given: A non-admin user's API call is rejected with 403
  When: The frontend receives the error response
  Then: A user-friendly message is displayed: "Insufficient Permissions — Group-level trigger management requires the Trigger Admin role. Contact your system administrator for access."

AC4:
  Given: An admin user navigates to the same pages
  When: The page loads
  Then: All action buttons are visible and functional (no restrictions).

**Tasks**
- [ ] Task 1: <Frontend> Implement role check on component mount (query user permissions from session/API)
- [ ] Task 2: <Frontend> Conditionally render/disable Add/Edit/Delete buttons based on permission
- [ ] Task 3: <Frontend> Implement permission-denied message component (reusable)
- [ ] Task 4: <Frontend> Handle 403 API responses gracefully (catch and display friendly message)
- [ ] Task 5: <Test> Component tests for admin vs. non-admin rendering
- [ ] Task 6: <Test> E2E test: login as non-admin → navigate to group triggers → verify read-only; login as admin → verify full access

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups for permission-denied state — `[TBD]`
- [ ] User role/permission API confirmed (how frontend queries current user's roles)
- [ ] Dependencies: S3.5 (backend permission enforcement)
- [ ] NFRs applicable: usability, security (UI complements server-side enforcement)
- [ ] Estimated by team

---

### STORY: S3.7 — Backend & UI: Audit Log Query API and Viewer

**Parent Epic**: BDCSPM-49336 Epic 3 — Rule-Based Bulk Trigger Application, Audit & Permissions
**BLSS Product**: Shared platform services (Audit subsystem + UI)

**Description**
As an On-Prem IT / System Admin,
I want to query and view the trigger audit log through a UI,
So that I can review what trigger changes were made, by whom, and when — for compliance and troubleshooting purposes.

**Context & notes**:
- New page: Triggers → Audit Log (or integrated into existing audit/activity log if one exists)
- Filter by: date range, user, action type, scope (device/group/model/rule), specific entity
- Sortable by timestamp (newest first by default)
- Paginated results (50 per page)
- Each entry shows: timestamp, user, action, scope, entity, summary of change
- Expandable row shows full before/after JSON diff
- Admin-only access (same "Trigger Admin" role or broader "Audit Viewer" role)
- ⚠️ ASSUMPTION: No existing audit log viewer exists; this is new UI.

**Acceptance Criteria**

AC1:
  Given: The admin navigates to Triggers → Audit Log
  When: The page loads
  Then: The most recent 50 audit entries are displayed, sorted by timestamp descending.

AC2:
  Given: The admin applies filters (date range: last 7 days, action: CREATE)
  When: The filter is applied
  Then: Only matching entries are displayed; pagination updates accordingly.

AC3:
  Given: An audit entry shows "CREATE GROUP TRIGGER on Closet A by admin@company.com"
  When: The admin expands the entry
  Then: Full details are shown including after_value (the trigger definition created).

AC4:
  Given: An audit entry for an UPDATE action
  When: The admin expands the entry
  Then: Both before_value and after_value are shown with the changed fields highlighted.

AC5:
  Given: The audit log has 10,000+ entries
  When: The admin queries with no filters
  Then: Results are paginated (50 per page); page load is < 3 seconds.

**Tasks**
- [ ] Task 1: <Backend> Create REST API endpoint: GET /api/v2/trigger-audit-log (paginated, filterable)
- [ ] Task 2: <Backend> Implement query filters (dateFrom, dateTo, userId, action, scope, entityId)
- [ ] Task 3: <Backend> Implement pagination (cursor-based or offset)
- [ ] Task 4: <Backend> Apply permission check (only "Trigger Admin" or "Audit Viewer" can access)
- [ ] Task 5: <Frontend> Implement Audit Log page with filter panel and data table
- [ ] Task 6: <Frontend> Implement expandable row for full before/after details
- [ ] Task 7: <Frontend> Implement pagination controls
- [ ] Task 8: <Frontend> Implement date range picker, user filter, action/scope dropdowns
- [ ] Task 9: <Test> Unit tests for query API (filters, pagination, permission)
- [ ] Task 10: <Test> E2E test: perform trigger operations → navigate to audit log → verify entries appear with correct details

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups for audit log viewer — `[TBD]`
- [ ] API contract defined (query parameters, response schema, pagination format)
- [ ] Dependencies: S3.4 (audit log entries exist), S3.5 (permission enforcement)
- [ ] NFRs applicable: performance (< 3s for paginated query), scalability (1M+ entries)
- [ ] Estimated by team

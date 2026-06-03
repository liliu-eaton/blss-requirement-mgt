# BDCSPM-49336 — Apply Triggers to a Group of Devices

## EPIC 1: Group-Level & Model-Level Trigger Configuration

**BLSS Product(s)**: Shared platform services (Monitoring / Triggers subsystem) — cross-product (DCPM, Distributed IT Performance Management, Brightlayer Power)
**Summary** (1-2 sentences)
Enable Operations Engineers and Facility Managers to define and apply triggers at the device group or device model level, so that trigger configuration for large device fleets is streamlined from per-device manual work to a single group/model-level action.

---

### Description

- **Problem statement**: Currently, triggers can only be applied at the individual device level. For customers managing large fleets (e.g., 200+ M2 card devices), adjusting a temperature trigger for 25 UPS Rackmount units requires manual per-device updates or cloning monitoring templates for each device — adding significant complexity, time, and risk of configuration drift.

- **Proposed solution**: Extend the trigger management engine to support two new trigger scopes beyond the existing device-level:
  1. **Device Group level**: A "Triggers" tab on the Device Group detail page allowing admins to define triggers that automatically apply to all devices in the group.
  2. **Model level**: A "Triggers" tab on the Model detail page allowing admins to define triggers that automatically apply to all devices of that model.
  3. **Inheritance propagation**: When a device is added to a group or a new device of a model is discovered, applicable triggers are automatically inherited.

- **User personas**: Data Center Facility Manager, Operations / NOC Engineer, On-Prem IT / System Admin, Eaton Field Service Engineer

- **Business value**: Reduces trigger configuration time by an estimated 80%+ for large device fleets. Eliminates configuration drift risk when the same threshold should apply to multiple devices. Reduces Lifecycle Services deployment and reconfiguration effort.

- **Key workflows**:
  1. Admin navigates to Device Groups → selects a group → opens Triggers tab
  2. Admin adds a trigger (e.g., Temperature > 85°F) at the group level
  3. System propagates the trigger to all devices in the group that have the matching attribute
  4. Devices without the attribute are silently skipped (logged)
  5. Alternatively: Admin navigates to Models → selects a model → opens Triggers tab
  6. Admin adds a trigger at model level; all devices of that model inherit it
  7. When a new device is added to the group or discovered as that model, triggers are auto-applied

- **Cross-product touchpoints**: The trigger engine is shared infrastructure. This Epic affects the trigger subsystem used by DCPM (facility device monitoring), Distributed IT (edge device monitoring), and Brightlayer Power (power device monitoring). No product-specific API changes; the trigger scope extension is at the platform layer.

- **UX/UI references**: `[TBD — wireframes/mockups to be produced during Design Epic BDCSPM-66978]`

- **On-prem operational impact**:
  - Install/upgrade impact: New DB tables for group-level and model-level trigger definitions; migration script must preserve existing device-level triggers unchanged.
  - Sizing impact: Minimal — trigger definitions are lightweight metadata. For very large fleets (10,000+ devices), propagation job may require slightly more CPU during trigger save operations.
  - Network impact: None (all local to Application Server).
  - Backup impact: New trigger tables must be included in backup scope; schema migration must be forward/backward compatible for rollback.

---

### Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Performance | Applying a group-level trigger to N devices (propagation completion) | ≤ 5 seconds for N ≤ 100 devices on Medium tier hardware | P0 |
| Performance | Applying a group-level trigger to N devices (propagation completion) | ≤ 15 seconds for N ≤ 500 devices on Medium tier hardware | P1 |
| Performance | Applying a model-level trigger (propagation completion) | ≤ 30 seconds for N ≤ 1,000 devices on Large tier hardware | P1 |
| Performance | Triggers tab page load time (group or model) | < 2s at P95 | P1 |
| Reliability | Propagation must be atomic per-device (partial failure must not leave inconsistent state) | No partial/corrupt trigger states | P0 |
| Reliability | If server restarts during propagation, pending propagations must resume | Eventual consistency within 60s of restart | P1 |
| Data integrity | Existing device-level triggers must not be affected by group/model trigger operations | Zero regression on existing device triggers | P0 |
| Compatibility | Must support PostgreSQL 14+ on RHEL 9/10 | All supported platform combinations | P0 |
| Scalability | System must support up to 500 device groups, each with up to 1,000 devices | `[TBD — confirm with architecture team]` | P1 |
| Scalability | System must support up to 200 device models with model-level triggers | `[TBD — confirm with architecture team]` | P2 |

---

### Acceptance Criteria (Epic-level)

- [ ] AC1: An admin can navigate to a Device Group detail page and see a "Triggers" tab where group-level triggers can be added, edited, and deleted.
- [ ] AC2: When a group-level trigger is saved, all devices in the group that have the matching attribute receive the trigger within the performance target (≤ 5s for 100 devices).
- [ ] AC3: Devices in the group that lack the specified attribute are silently skipped; a log entry records which devices were skipped and why.
- [ ] AC4: An admin can navigate to a Model detail page and see a "Triggers" tab where model-level triggers can be added, edited, and deleted.
- [ ] AC5: When a model-level trigger is saved, all devices of that model that have the matching attribute receive the trigger.
- [ ] AC6: When a new device is added to a group that has group-level triggers, the device automatically inherits applicable triggers.
- [ ] AC7: When a new device of a model with model-level triggers is discovered/added, the device automatically inherits applicable model triggers.
- [ ] AC8: When a device is removed from a group, inherited group triggers are removed from that device (unless overridden at device level — see Epic 2).
- [ ] AC9: Existing device-level trigger functionality is fully preserved with zero regression.
- [ ] AC10: Schema migration from previous version preserves all existing device-level triggers; upgrade is reversible.

---

### Scope

- **In scope**:
  - New DB schema for group-level and model-level trigger definitions
  - Trigger propagation service (apply/remove inherited triggers on group membership changes)
  - "Triggers" tab on Device Group detail page (CRUD operations)
  - "Triggers" tab on Model detail page (CRUD operations)
  - Auto-inheritance on device addition to group
  - Auto-inheritance on device discovery for model-level triggers
  - Removal of inherited triggers on device removal from group
  - Silent skip for devices lacking the specified attribute (with logging)
  - DB migration script with rollback support

- **Out of scope**:
  - Priority resolution when device has triggers from multiple sources (Epic 2)
  - Device-level override UX (Epic 2)
  - Rule-based bulk application across multiple groups (Epic 3)
  - Audit trail / compliance logging of trigger changes (Epic 3)
  - Permission / RBAC for group-level trigger management (Epic 3)
  - Changes to existing device-level trigger creation UX
  - Alarm notification or escalation changes
  - Monitoring template redesign

- **Assumptions**:
  1. ⚠️ ASSUMPTION: "Device Groups" already exist as a first-class entity in BLSS with device membership management (add/remove devices from groups).
  2. ⚠️ ASSUMPTION: A "Model" entity exists in the device inventory; devices have a model association field.
  3. ⚠️ ASSUMPTION: The current trigger system stores trigger definitions per-device in PostgreSQL with a well-defined schema (trigger_id, device_id, attribute, condition, threshold, etc.).
  4. ⚠️ ASSUMPTION: The "Triggers" tab UI will follow the same UX patterns as existing trigger management on the device detail page.
  5. ⚠️ ASSUMPTION: Group/model trigger propagation is synchronous (user waits for completion confirmation) for groups ≤ 100 devices, and asynchronous with progress indicator for larger groups.
  6. ⚠️ ASSUMPTION: Device "attributes" are discoverable from the device model/driver metadata (system knows which attributes a device supports).

---

### Dependencies & Risks

| Item | Type | BLSS Product | Owner | Mitigation | Status |
|---|---|---|---|---|---|
| Device Group entity must exist with membership API | Dep | Shared platform | `[TBD]` | Confirm Device Groups feature is complete and stable | ⚠️ NEEDS VALIDATION |
| Model entity must exist with device association | Dep | Shared platform | `[TBD]` | Confirm Model entity and device-model relationship | ⚠️ NEEDS VALIDATION |
| SA Evaluation (BDCSPM-66977) must define technical approach | Dep | Shared platform | Chen, Peter | SA Evaluation is "In Planning" | In Progress |
| Design mockups (BDCSPM-66978) needed for UI stories | Dep | UX/Design | `[TBD]` | Blocked until SA Evaluation completes | Funnel |
| Large fleet propagation performance | Risk | Shared platform | Architecture | Load test with 500+ devices early; consider async propagation with progress | Open |
| Schema migration complexity | Risk | Shared platform | Architecture | Design forward/backward compatible schema; test rollback extensively | Open |

---

### Operational Deliverables (required for on-prem)

- [ ] Installation/upgrade runbook updated (new DB tables, migration script execution)
- [ ] Sizing guide updated (trigger propagation CPU/memory impact for large fleets)
- [ ] Backup/restore procedure updated (new trigger tables in backup scope)
- [ ] Firewall/port documentation: N/A (no new network requirements)
- [ ] Release notes entry drafted
- [ ] Lifecycle Services training/enablement updated (new trigger configuration workflow)

---

### Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| S1.1 | Backend: Trigger scope data model (group-level & model-level trigger persistence) | 8 | `[TBD]` |
| S1.2 | Backend: Trigger propagation service (apply group/model triggers to devices) | 8 | `[TBD]` |
| S1.3 | UI: Add "Triggers" tab to Device Group detail page | 5 | `[TBD]` |
| S1.4 | UI: Add/edit/delete triggers on Device Group "Triggers" tab | 8 | `[TBD]` |
| S1.5 | UI: Add "Triggers" tab to Model detail page | 5 | `[TBD]` |
| S1.6 | UI: Add/edit/delete triggers on Model "Triggers" tab | 5 | `[TBD]` |
| S1.7 | Backend: Auto-apply inherited triggers on device addition/removal from group | 5 | `[TBD]` |
| S1.8 | Backend: Auto-apply model triggers on device discovery | 5 | `[TBD]` |

---

## Stories

---

### STORY: S1.1 — Backend: Trigger Scope Data Model

**Parent Epic**: BDCSPM-49336 Epic 1 — Group-Level & Model-Level Trigger Configuration
**BLSS Product**: Shared platform services (Triggers subsystem)

**Description**
As an On-Prem IT / System Admin,
I want the system to persist trigger definitions at the device group and model levels (in addition to device level),
So that triggers can be managed at higher scopes without duplicating definitions per-device.

**Context & notes**:
- Extends existing trigger table schema to support `scope` (device, group, model) and `scope_id` (device_id, group_id, model_id)
- Alternatively: new tables `group_trigger` and `model_trigger` with FK relationships
- Must include DB migration script with rollback (down migration)
- Must preserve existing device-level trigger data with zero modification
- ⚠️ ASSUMPTION: Current schema has a `device_trigger` table or equivalent. Confirm exact table/column names with SA team.

**Acceptance Criteria**

AC1:
  Given: The BLSS Application Server is upgraded to the new version
  When: The DB migration script executes
  Then: New tables/columns for group-level and model-level trigger definitions are created; existing device-level trigger data is unchanged.

AC2:
  Given: A group-level trigger definition is saved via API
  When: The system queries triggers for that group
  Then: The group-level trigger is returned with all fields (attribute, condition, threshold, severity, enabled flag).

AC3:
  Given: A model-level trigger definition is saved via API
  When: The system queries triggers for that model
  Then: The model-level trigger is returned with all fields.

AC4:
  Given: The DB migration has been applied
  When: A rollback (down migration) is executed
  Then: The new tables/columns are removed; existing device-level triggers remain intact.

AC5 (edge case):
  Given: A group or model trigger references an attribute name
  When: The attribute name is validated
  Then: The system accepts any valid attribute name from the device attribute catalog (no FK constraint to specific devices at definition time).

**Tasks**
- [ ] Task 1: <Data/schema> Design group_trigger and model_trigger table schemas (columns: id, scope_id, attribute_name, condition_type, threshold_value, severity, enabled, created_by, created_at, updated_at)
- [ ] Task 2: <Data/schema> Write forward DB migration script (PostgreSQL 14+)
- [ ] Task 3: <Data/schema> Write rollback DB migration script
- [ ] Task 4: <Backend> Implement repository/DAO layer for group_trigger CRUD
- [ ] Task 5: <Backend> Implement repository/DAO layer for model_trigger CRUD
- [ ] Task 6: <Backend> Create REST API endpoints: POST/GET/PUT/DELETE /api/v2/device-groups/{groupId}/triggers
- [ ] Task 7: <Backend> Create REST API endpoints: POST/GET/PUT/DELETE /api/v2/models/{modelId}/triggers
- [ ] Task 8: <Test> Unit tests for repository layer (CRUD operations, constraint validation)
- [ ] Task 9: <Test> Integration tests for API endpoints (happy path + validation errors)
- [ ] Task 10: <Installer/config> Update installer to execute new migration script; update schema version registry

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (if UI-impacting) — N/A (backend story)
- [ ] API contract defined (endpoint paths, request/response schemas)
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified (existing trigger table schema confirmed)
- [ ] On-prem install/upgrade impact assessed (migration script)
- [ ] Sizing impact assessed — minimal (metadata tables)
- [ ] Dependencies identified and unblocked (current schema documented)
- [ ] NFRs applicable: data integrity, compatibility
- [ ] Estimated by team

---

### STORY: S1.2 — Backend: Trigger Propagation Service

**Parent Epic**: BDCSPM-49336 Epic 1 — Group-Level & Model-Level Trigger Configuration
**BLSS Product**: Shared platform services (Triggers subsystem)

**Description**
As an Operations / NOC Engineer,
I want the system to automatically propagate group-level and model-level triggers to all applicable devices when a trigger is created or modified at those scopes,
So that I don't need to manually apply triggers device-by-device after defining them at the group/model level.

**Context & notes**:
- This service is the core engine that "fans out" a group/model trigger to individual devices
- Must skip devices that don't have the specified attribute (silent skip with log entry)
- Must handle partial failure gracefully (some devices may fail to receive trigger due to transient issues)
- Performance target: ≤ 5s for 100 devices, ≤ 15s for 500 devices
- Consider async execution with progress tracking for groups > 100 devices
- Must be idempotent (re-running propagation produces same result)

**Acceptance Criteria**

AC1:
  Given: A group-level trigger is created for attribute "Temperature" with condition "> 85°F" on a group with 50 devices (all having Temperature attribute)
  When: The trigger is saved
  Then: All 50 devices have the trigger applied within 5 seconds; a success confirmation is returned.

AC2:
  Given: A group-level trigger is created for attribute "Temperature" on a mixed group (30 devices have Temperature, 20 do not)
  When: The trigger is saved
  Then: 30 devices receive the trigger; 20 are skipped; a log entry records skipped devices with reason "attribute not available".

AC3:
  Given: A group-level trigger is updated (threshold changed from 85°F to 90°F)
  When: The update is saved
  Then: All devices with the inherited trigger are updated to the new threshold value.

AC4:
  Given: A group-level trigger is deleted
  When: The deletion is confirmed
  Then: The inherited trigger is removed from all devices in the group (unless overridden at device level — handled in Epic 2).

AC5:
  Given: A model-level trigger is created for attribute "Voltage" on model "M2 Card" with 200 devices
  When: The trigger is saved
  Then: All 200 devices of that model receive the trigger within the performance target.

AC6 (edge case):
  Given: The Application Server restarts during a propagation operation
  When: The server comes back online
  Then: The propagation resumes or re-executes, achieving eventual consistency within 60 seconds.

AC7 (edge case):
  Given: Propagation is triggered for a group with 500 devices
  When: The operation takes longer than 5 seconds
  Then: The system provides an asynchronous progress indicator (API returns job ID; job status is queryable).

**Tasks**
- [ ] Task 1: <Backend> Design propagation service interface (TriggerPropagationService with methods: propagateToGroup, propagateToModel, retractFromGroup, retractFromModel)
- [ ] Task 2: <Backend> Implement synchronous propagation for groups ≤ 100 devices
- [ ] Task 3: <Backend> Implement asynchronous propagation with job tracking for groups > 100 devices
- [ ] Task 4: <Backend> Implement attribute availability check per device (query device attribute catalog)
- [ ] Task 5: <Backend> Implement skip logic with structured logging (device_id, attribute, reason)
- [ ] Task 6: <Backend> Implement idempotency (re-propagation produces same state)
- [ ] Task 7: <Backend> Implement restart recovery (persist propagation state; resume on startup)
- [ ] Task 8: <Backend> Create REST API endpoint: GET /api/v2/trigger-propagation-jobs/{jobId}/status
- [ ] Task 9: <Test> Unit tests for propagation logic (happy path, skip, partial failure, idempotency)
- [ ] Task 10: <Test> Performance test: propagate trigger to 100, 500, 1000 devices; measure elapsed time
- [ ] Task 11: <Test> Integration test: restart recovery scenario

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — N/A (backend service)
- [ ] API contract defined (propagation job status endpoint)
- [ ] Device attribute catalog API confirmed available
- [ ] Cross-product dependencies identified (device attribute metadata source)
- [ ] On-prem install/upgrade impact assessed — new service component
- [ ] Sizing impact assessed (CPU/memory for large propagations)
- [ ] Dependencies identified: S1.1 (data model) must be complete
- [ ] NFRs applicable: performance (5s/100 devices), reliability (restart recovery)
- [ ] Estimated by team

---

### STORY: S1.3 — UI: Add "Triggers" Tab to Device Group Detail Page

**Parent Epic**: BDCSPM-49336 Epic 1 — Group-Level & Model-Level Trigger Configuration
**BLSS Product**: Shared platform services (UI layer)

**Description**
As a Data Center Facility Manager,
I want to see a "Triggers" tab on the Device Group detail page,
So that I can access and manage triggers that apply to all devices in that group.

**Context & notes**:
- New tab added to existing Device Group detail page
- Tab shows list of group-level triggers (table/grid view)
- Empty state: "No group-level triggers defined. Add a trigger to apply it to all devices in this group."
- Must follow Brightlayer Design System patterns for tab navigation and data tables
- Link to UX mockup: `[TBD — pending BDCSPM-66978 Design Epic]`

**Acceptance Criteria**

AC1:
  Given: A user navigates to a Device Group detail page
  When: The page loads
  Then: A "Triggers" tab is visible alongside existing tabs (e.g., Devices, Properties).

AC2:
  Given: The user clicks the "Triggers" tab on a group with 3 defined triggers
  When: The tab content loads
  Then: A table displays all 3 triggers with columns: Attribute, Condition, Threshold, Severity, Enabled, Actions.

AC3:
  Given: The user clicks the "Triggers" tab on a group with no triggers
  When: The tab content loads
  Then: An empty state message is displayed with an "Add Trigger" action button.

AC4:
  Given: The group has triggers and the table is displayed
  When: The user views the table
  Then: Each row shows the trigger's current status (enabled/disabled) and provides Edit/Delete action buttons.

AC5 (edge case):
  Given: The user does not have admin permissions
  When: They view the Triggers tab
  Then: The triggers are visible (read-only) but Add/Edit/Delete actions are hidden or disabled. `[TBD — confirm with Epic 3 permission design]`

**Tasks**
- [ ] Task 1: <Frontend> Add "Triggers" tab to Device Group detail page component
- [ ] Task 2: <Frontend> Implement trigger list table component (reusable for Model page)
- [ ] Task 3: <Frontend> Implement empty state component for no-triggers scenario
- [ ] Task 4: <Frontend> Wire tab to API: GET /api/v2/device-groups/{groupId}/triggers
- [ ] Task 5: <Frontend> Implement loading state (skeleton/spinner) during API call
- [ ] Task 6: <Test> Component tests for Triggers tab rendering (with data, empty state)
- [ ] Task 7: <Test> E2E test: navigate to Device Group → Triggers tab → verify data display

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — `[TBD — pending Design Epic]`
- [ ] API contract defined (GET endpoint from S1.1)
- [ ] Cross-product dependencies: none
- [ ] On-prem install/upgrade impact: frontend bundle update (standard)
- [ ] Sizing impact: none
- [ ] Dependencies: S1.1 (API endpoints available)
- [ ] NFRs applicable: page load < 2s
- [ ] Estimated by team

---

### STORY: S1.4 — UI: Add/Edit/Delete Triggers on Device Group "Triggers" Tab

**Parent Epic**: BDCSPM-49336 Epic 1 — Group-Level & Model-Level Trigger Configuration
**BLSS Product**: Shared platform services (UI layer)

**Description**
As a Data Center Facility Manager,
I want to add, edit, and delete triggers on the Device Group "Triggers" tab,
So that I can manage which triggers apply to all devices in the group.

**Context & notes**:
- "Add Trigger" opens a form/dialog with: Attribute selector, Condition type, Threshold value, Severity, Enabled toggle
- Attribute selector should show attributes available across devices in the group (union of all device attributes)
- On save, trigger propagation is initiated (calls S1.2 service)
- Confirmation dialog on delete with count of affected devices
- Success/error toast notifications
- Link to UX mockup: `[TBD — pending BDCSPM-66978 Design Epic]`

**Acceptance Criteria**

AC1:
  Given: The user is on the Triggers tab of a Device Group
  When: They click "Add Trigger"
  Then: A form/dialog opens with fields: Attribute (dropdown), Condition (dropdown: >, <, >=, <=, =, !=), Threshold (numeric input), Severity (dropdown), Enabled (toggle, default: on).

AC2:
  Given: The user fills in all required fields and clicks "Save"
  When: The trigger is valid
  Then: The trigger is saved, propagation is initiated, and a success notification displays the count of devices affected (e.g., "Trigger applied to 25 of 30 devices. 5 devices skipped (attribute not available).").

AC3:
  Given: The user clicks "Edit" on an existing group trigger
  When: The edit form opens
  Then: All fields are pre-populated with current values; the user can modify and save.

AC4:
  Given: The user modifies a trigger threshold and saves
  When: The update completes
  Then: The updated threshold is propagated to all applicable devices; success notification shown.

AC5:
  Given: The user clicks "Delete" on a group trigger
  When: The confirmation dialog appears showing "This will remove the trigger from X devices"
  Then: On confirm, the trigger is deleted and retracted from all devices; on cancel, no action.

AC6 (edge case):
  Given: The user attempts to add a trigger with an empty threshold
  When: They click Save
  Then: Validation error is displayed; the form is not submitted.

AC7 (edge case):
  Given: Propagation takes longer than 5 seconds (large group)
  When: The async propagation starts
  Then: The UI shows a progress indicator; the user can navigate away and the propagation continues in the background.

**Tasks**
- [ ] Task 1: <Frontend> Implement "Add Trigger" form/dialog component
- [ ] Task 2: <Frontend> Implement attribute selector (fetch available attributes for group via API)
- [ ] Task 3: <Frontend> Implement form validation (required fields, numeric threshold)
- [ ] Task 4: <Frontend> Wire Save action to POST /api/v2/device-groups/{groupId}/triggers
- [ ] Task 5: <Frontend> Implement Edit flow (pre-populate form, PUT on save)
- [ ] Task 6: <Frontend> Implement Delete flow (confirmation dialog with device count, DELETE on confirm)
- [ ] Task 7: <Frontend> Implement success/error toast notifications with propagation summary
- [ ] Task 8: <Frontend> Implement async propagation progress indicator (poll job status)
- [ ] Task 9: <Backend> Create API endpoint: GET /api/v2/device-groups/{groupId}/available-attributes (union of attributes across group devices)
- [ ] Task 10: <Test> Component tests for form validation, submit, error states
- [ ] Task 11: <Test> E2E test: full Add → propagation → verify → Edit → Delete cycle

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — `[TBD — pending Design Epic]`
- [ ] API contract defined (CRUD + available-attributes endpoints)
- [ ] Cross-product dependencies: none
- [ ] On-prem install/upgrade impact: frontend bundle update (standard)
- [ ] Sizing impact: none
- [ ] Dependencies: S1.1 (data model), S1.2 (propagation service), S1.3 (tab exists)
- [ ] NFRs applicable: performance (propagation feedback within 5s or async)
- [ ] Estimated by team

---

### STORY: S1.5 — UI: Add "Triggers" Tab to Model Detail Page

**Parent Epic**: BDCSPM-49336 Epic 1 — Group-Level & Model-Level Trigger Configuration
**BLSS Product**: Shared platform services (UI layer)

**Description**
As an Operations / NOC Engineer,
I want to see a "Triggers" tab on the Model detail page,
So that I can access and manage triggers that apply to all devices of that model.

**Context & notes**:
- Mirrors the pattern from S1.3 (Device Group Triggers tab) but on the Model detail page
- Tab shows list of model-level triggers
- Reuses the trigger list table component from S1.3
- Link to UX mockup: `[TBD — pending BDCSPM-66978 Design Epic]`

**Acceptance Criteria**

AC1:
  Given: A user navigates to a Model detail page
  When: The page loads
  Then: A "Triggers" tab is visible alongside existing tabs.

AC2:
  Given: The user clicks the "Triggers" tab on a model with defined triggers
  When: The tab content loads
  Then: A table displays all model-level triggers with columns: Attribute, Condition, Threshold, Severity, Enabled, Actions.

AC3:
  Given: The user clicks the "Triggers" tab on a model with no triggers
  When: The tab content loads
  Then: An empty state message is displayed with an "Add Trigger" action button.

AC4:
  Given: The table displays model triggers
  When: The user views the device count
  Then: Each trigger row shows how many devices of this model currently have the trigger applied.

**Tasks**
- [ ] Task 1: <Frontend> Add "Triggers" tab to Model detail page component
- [ ] Task 2: <Frontend> Reuse trigger list table component from S1.3
- [ ] Task 3: <Frontend> Wire tab to API: GET /api/v2/models/{modelId}/triggers
- [ ] Task 4: <Frontend> Show device count per trigger (API includes affected_device_count)
- [ ] Task 5: <Test> Component tests for Model Triggers tab rendering
- [ ] Task 6: <Test> E2E test: navigate to Model → Triggers tab → verify data display

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — `[TBD — pending Design Epic]`
- [ ] API contract defined (GET endpoint from S1.1)
- [ ] Cross-product dependencies: none
- [ ] On-prem install/upgrade impact: frontend bundle update (standard)
- [ ] Sizing impact: none
- [ ] Dependencies: S1.1 (API endpoints), S1.3 (reusable components)
- [ ] NFRs applicable: page load < 2s
- [ ] Estimated by team

---

### STORY: S1.6 — UI: Add/Edit/Delete Triggers on Model "Triggers" Tab

**Parent Epic**: BDCSPM-49336 Epic 1 — Group-Level & Model-Level Trigger Configuration
**BLSS Product**: Shared platform services (UI layer)

**Description**
As an Operations / NOC Engineer,
I want to add, edit, and delete triggers on the Model "Triggers" tab,
So that I can define triggers that automatically apply to all devices of a specific model.

**Context & notes**:
- Mirrors the pattern from S1.4 (Device Group CRUD) but for model context
- Attribute selector shows attributes available for the model (from model definition/driver metadata)
- On save, trigger propagation applies to all devices of that model
- Link to UX mockup: `[TBD — pending BDCSPM-66978 Design Epic]`

**Acceptance Criteria**

AC1:
  Given: The user is on the Triggers tab of a Model page
  When: They click "Add Trigger"
  Then: A form/dialog opens with Attribute (populated from model's known attributes), Condition, Threshold, Severity, Enabled fields.

AC2:
  Given: The user saves a valid model-level trigger
  When: The trigger is saved
  Then: Propagation applies the trigger to all devices of that model; success notification shows affected device count.

AC3:
  Given: The user edits an existing model-level trigger
  When: The update is saved
  Then: All devices of that model are updated with the new trigger values.

AC4:
  Given: The user deletes a model-level trigger
  When: Deletion is confirmed
  Then: The trigger is removed from all devices of that model (unless overridden at device level).

**Tasks**
- [ ] Task 1: <Frontend> Implement Add/Edit/Delete flows for model triggers (reuse group trigger form component with model context)
- [ ] Task 2: <Frontend> Wire to POST/PUT/DELETE /api/v2/models/{modelId}/triggers
- [ ] Task 3: <Backend> Create API endpoint: GET /api/v2/models/{modelId}/available-attributes (from model driver metadata)
- [ ] Task 4: <Test> Component tests for model trigger CRUD
- [ ] Task 5: <Test> E2E test: Add model trigger → verify propagation to devices of that model

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — `[TBD]`
- [ ] API contract defined
- [ ] Dependencies: S1.1, S1.2, S1.4 (reusable form component), S1.5
- [ ] NFRs applicable: performance
- [ ] Estimated by team

---

### STORY: S1.7 — Backend: Auto-Apply Inherited Triggers on Device Addition/Removal from Group

**Parent Epic**: BDCSPM-49336 Epic 1 — Group-Level & Model-Level Trigger Configuration
**BLSS Product**: Shared platform services (Triggers subsystem)

**Description**
As an On-Prem IT / System Admin,
I want newly added devices to a group to automatically inherit the group's triggers (and have them removed when the device leaves the group),
So that trigger configuration stays consistent without manual intervention when group membership changes.

**Context & notes**:
- Triggered by device group membership change events (add device to group, remove device from group)
- On add: evaluate all group-level triggers; apply those whose attribute exists on the device
- On remove: retract all group-inherited triggers from the device (unless overridden at device level — Epic 2)
- Must integrate with existing device group membership management API/event system
- ⚠️ ASSUMPTION: Device group membership changes emit an event/hook that this service can subscribe to.

**Acceptance Criteria**

AC1:
  Given: A device group has 2 triggers defined (Temperature > 85°F, Humidity > 70%)
  When: A new device (with both Temperature and Humidity attributes) is added to the group
  Then: Both triggers are automatically applied to the new device.

AC2:
  Given: A device group has a Temperature trigger
  When: A device without a Temperature attribute is added to the group
  Then: The trigger is not applied to that device; a log entry records the skip.

AC3:
  Given: A device has inherited group triggers
  When: The device is removed from the group
  Then: All group-inherited triggers are removed from the device.

AC4 (edge case):
  Given: A device belongs to two groups, both defining a Temperature trigger
  When: The device is removed from one group
  Then: Only that group's trigger is removed; the other group's trigger remains. `[TBD — confirm multi-group membership behavior]`

AC5 (edge case):
  Given: Multiple devices are added to a group in a batch operation
  When: The batch completes
  Then: All devices receive applicable group triggers; performance is proportional to batch size.

**Tasks**
- [ ] Task 1: <Backend> Subscribe to device group membership change events (add/remove)
- [ ] Task 2: <Backend> Implement "on device added to group" handler (apply applicable group triggers)
- [ ] Task 3: <Backend> Implement "on device removed from group" handler (retract group triggers)
- [ ] Task 4: <Backend> Handle batch add/remove operations efficiently
- [ ] Task 5: <Test> Unit tests for add/remove handlers (single device, batch, skip logic)
- [ ] Task 6: <Test> Integration test: add device to group → verify trigger inheritance → remove → verify retraction

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Device group membership event system confirmed available
- [ ] Cross-product dependencies: device group management feature
- [ ] On-prem install/upgrade impact: event subscription registration
- [ ] Dependencies: S1.1 (data model), S1.2 (propagation service)
- [ ] NFRs applicable: reliability (no missed events)
- [ ] Estimated by team

---

### STORY: S1.8 — Backend: Auto-Apply Model Triggers on Device Discovery

**Parent Epic**: BDCSPM-49336 Epic 1 — Group-Level & Model-Level Trigger Configuration
**BLSS Product**: Shared platform services (Triggers subsystem)

**Description**
As an Eaton Field Service Engineer,
I want newly discovered devices to automatically inherit their model's triggers,
So that new devices are immediately configured with the standard trigger thresholds defined for their model without manual setup.

**Context & notes**:
- Triggered by device discovery/registration events
- When a new device is discovered and its model is identified, check if model-level triggers exist
- Apply applicable model triggers to the new device
- ⚠️ ASSUMPTION: Device discovery emits an event containing the device model identifier

**Acceptance Criteria**

AC1:
  Given: Model "M2 Card" has a Voltage trigger (< 110V) defined
  When: A new M2 Card device is discovered/added to the system
  Then: The Voltage trigger is automatically applied to the new device.

AC2:
  Given: Model "M2 Card" has a trigger for attribute "X" that the specific device variant doesn't support
  When: The device is discovered
  Then: The trigger for attribute "X" is skipped; a log entry records the skip.

AC3:
  Given: A model has no triggers defined
  When: A new device of that model is discovered
  Then: No triggers are applied; no errors or warnings generated.

AC4 (edge case):
  Given: A device is re-discovered (already exists in the system)
  When: The discovery event fires
  Then: Model triggers are not duplicated; the system is idempotent (existing inherited triggers remain unchanged).

**Tasks**
- [ ] Task 1: <Backend> Subscribe to device discovery/registration event
- [ ] Task 2: <Backend> Implement "on device discovered" handler (lookup model, apply model triggers)
- [ ] Task 3: <Backend> Implement idempotency check (skip if trigger already inherited)
- [ ] Task 4: <Test> Unit tests for discovery handler (new device, re-discovery, no model triggers)
- [ ] Task 5: <Test> Integration test: discover device → verify model trigger inheritance

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Device discovery event system confirmed available
- [ ] Model identification confirmed in discovery event payload
- [ ] On-prem install/upgrade impact: event subscription registration
- [ ] Dependencies: S1.1 (data model), S1.2 (propagation service)
- [ ] NFRs applicable: reliability (no missed events), idempotency
- [ ] Estimated by team

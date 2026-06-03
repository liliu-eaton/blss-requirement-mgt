# BDCSPM-49336 — Apply Triggers to a Group of Devices

## EPIC 2: Trigger Priority Resolution & Device-Level Override

**BLSS Product(s)**: Shared platform services (Monitoring / Triggers subsystem) — cross-product (DCPM, Distributed IT Performance Management, Brightlayer Power)
**Summary** (1-2 sentences)
Implement the trigger priority resolution engine that enforces the inheritance hierarchy (Device-level > Group-level > Model-level) and provide a UX for viewing trigger inheritance sources, overriding inherited triggers at the device level, and reverting overrides back to inherited values.

---

### Description

- **Problem statement**: When triggers can be defined at device, group, and model levels simultaneously, conflicts will arise (e.g., a device may have a Temperature threshold from its model at 80°F, from its group at 85°F, and a device-level override at 90°F). Without a clear, deterministic priority resolution mechanism and user-facing visibility into trigger sources, administrators cannot understand which trigger is effective or confidently manage overrides.

- **Proposed solution**: Implement a three-tier priority resolution engine:
  1. **Highest priority**: Device-level trigger definition (explicit override)
  2. **High priority**: Device Group-level trigger definition
  3. **Medium priority**: Model-level trigger definition
  
  The system resolves the "effective trigger" for each attribute on each device by applying this priority hierarchy. The UI clearly indicates the source of each trigger (inherited from model, inherited from group, or device-level override) and provides UX to override or revert.

- **User personas**: Data Center Facility Manager, Operations / NOC Engineer, On-Prem IT / System Admin

- **Business value**: Provides flexibility for "configure once at group/model level, override where needed" workflows. Prevents administrator confusion about which trigger is active on a device. Supports the common pattern of fleet-wide defaults with per-device exceptions.

- **Key workflows**:
  1. Admin views a device's Triggers tab → sees all effective triggers with source indicators (Model, Group, Device)
  2. Admin identifies one device that needs a different threshold → overrides group trigger at device level
  3. Admin later decides the override is no longer needed → reverts to inherited value
  4. System always uses the highest-priority trigger for alarm evaluation

- **Cross-product touchpoints**: Same as Epic 1 — shared triggers subsystem.

- **UX/UI references**: `[TBD — wireframes/mockups to be produced during Design Epic BDCSPM-66978]`

- **On-prem operational impact**:
  - Install/upgrade impact: New column or table to track trigger source/scope per device; migration must classify existing device-level triggers as "device-level" source.
  - Sizing impact: Minimal — additional metadata per trigger record.
  - Network impact: None.
  - Backup impact: Source/scope metadata included in trigger backup.

---

### Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Performance | Priority resolution for a single device (compute effective triggers) | < 100ms at P95 | P0 |
| Performance | Device Triggers tab load (showing all effective triggers with sources) | < 2s at P95 on Medium tier | P1 |
| Performance | Override save (device-level) | < 1s at P95 | P1 |
| Reliability | Priority resolution must be deterministic and consistent | Same input → same effective trigger, always | P0 |
| Usability | Trigger source must be immediately visually identifiable | Icon/badge/color per source type | P1 |
| Data integrity | Override at device level must not modify or delete the group/model definition | Group/model triggers unchanged | P0 |
| Data integrity | Revert must restore to current group/model value (not historical) | Live inheritance after revert | P0 |

---

### Acceptance Criteria (Epic-level)

- [ ] AC1: When a device has triggers from device-level, group-level, and model-level for the same attribute, the device-level trigger takes precedence for alarm evaluation.
- [ ] AC2: When a device has triggers from group-level and model-level (no device override), the group-level trigger takes precedence.
- [ ] AC3: The device Triggers tab displays all effective triggers with a clear source indicator (icon/badge: "Device", "Group: [name]", "Model: [name]").
- [ ] AC4: An admin can override a group-inherited trigger at the device level; the effective trigger immediately switches to the device-level value.
- [ ] AC5: An admin can revert a device-level override; the effective trigger reverts to the next highest-priority source (group or model).
- [ ] AC6: Group/model trigger definitions are never modified or deleted by device-level override or revert operations.
- [ ] AC7: The alarm engine uses the effective (resolved) trigger value for alarm evaluation on every device.

---

### Scope

- **In scope**:
  - Three-tier priority resolution engine (Device > Group > Model)
  - Effective trigger computation per device
  - Source indicator display on device Triggers tab
  - Device-level override of inherited triggers
  - Revert device-level override to inherited value
  - Integration with alarm engine (use effective trigger value)
  - Handling devices in multiple groups (if a device is in two groups with same-attribute triggers, `[TBD — define resolution: first group wins? most recent? error?]`)

- **Out of scope**:
  - Group-level trigger CRUD (Epic 1)
  - Model-level trigger CRUD (Epic 1)
  - Audit logging of override/revert actions (Epic 3)
  - Permission control for override actions (Epic 3)
  - UI for bulk override across multiple devices
  - History/changelog of trigger value changes

- **Assumptions**:
  1. ⚠️ ASSUMPTION: A device can belong to only ONE device group at a time (simplifies priority resolution). If multi-group membership is supported, define conflict resolution rule: `[TBD — confirm with architecture team]`
  2. ⚠️ ASSUMPTION: The alarm engine can be updated to query "effective trigger" instead of directly reading the device_trigger table.
  3. ⚠️ ASSUMPTION: "Override" creates a device-level trigger record that shadows the inherited trigger; it does not modify the group/model definition.
  4. ⚠️ ASSUMPTION: "Revert" deletes the device-level override record, causing the system to fall through to the next priority level.

---

### Dependencies & Risks

| Item | Type | BLSS Product | Owner | Mitigation | Status |
|---|---|---|---|---|---|
| Epic 1 must be complete (group/model triggers exist) | Dep | Shared platform | Dev team | Sequence Epics: 1 → 2 → 3 | Planned |
| Alarm engine must support "effective trigger" resolution | Dep | Shared platform | `[TBD]` | Early spike to assess alarm engine modification effort | ⚠️ NEEDS VALIDATION |
| Multi-group membership conflict resolution | Risk | Shared platform | Architecture | Define rule early; if complex, defer multi-group to future iteration | Open |
| Performance of priority resolution at scale (10,000 devices) | Risk | Shared platform | Architecture | Cache effective triggers; invalidate on change | Open |

---

### Operational Deliverables (required for on-prem)

- [ ] Installation/upgrade runbook updated (trigger source metadata migration)
- [ ] Sizing guide: N/A (minimal additional metadata)
- [ ] Backup/restore procedure: no change (triggers already in backup scope)
- [ ] Release notes entry drafted (inheritance behavior explanation)
- [ ] Lifecycle Services training/enablement updated (override/revert workflow)

---

### Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| S2.1 | Backend: Trigger priority resolution engine (Device > Group > Model) | 8 | `[TBD]` |
| S2.2 | UI: Display inherited vs. overridden triggers on Device "Triggers" tab | 5 | `[TBD]` |
| S2.3 | UI: Override inherited trigger at device level | 5 | `[TBD]` |
| S2.4 | UI: Revert device-level override to inherit from group/model | 3 | `[TBD]` |
| S2.5 | Backend: Skip devices without matching attribute (silent skip + log) | 5 | `[TBD]` |
| S2.6 | UI: Display trigger inheritance source indicator (group/model/device) | 3 | `[TBD]` |

---

## Stories

---

### STORY: S2.1 — Backend: Trigger Priority Resolution Engine

**Parent Epic**: BDCSPM-49336 Epic 2 — Trigger Priority Resolution & Device-Level Override
**BLSS Product**: Shared platform services (Triggers subsystem)

**Description**
As the alarm evaluation engine,
I want a deterministic priority resolution function that computes the "effective trigger" for a given device and attribute,
So that the correct trigger threshold is always used for alarm evaluation regardless of how many levels define it.

**Context & notes**:
- Core algorithm: For each (device, attribute) pair, resolve effective trigger by checking in order: device_trigger → group_trigger (via device's group membership) → model_trigger (via device's model)
- First match wins (highest priority)
- Must be highly performant (called frequently by alarm engine during polling cycles)
- Consider caching effective triggers per device with invalidation on trigger CRUD or group membership changes
- ⚠️ NEEDS VALIDATION: If a device belongs to multiple groups, define deterministic tie-breaking (e.g., most specific group, most recently defined trigger, or error/warning)

**Acceptance Criteria**

AC1:
  Given: Device "UPS-001" has a device-level Temperature trigger (> 90°F), its group "Closet A" has Temperature (> 85°F), and its model "UPS Rackmount" has Temperature (> 80°F)
  When: The effective trigger is resolved for UPS-001 / Temperature
  Then: The effective trigger is "> 90°F" (device-level wins).

AC2:
  Given: Device "UPS-002" has NO device-level Temperature trigger, its group has Temperature (> 85°F), and its model has Temperature (> 80°F)
  When: The effective trigger is resolved for UPS-002 / Temperature
  Then: The effective trigger is "> 85°F" (group-level wins over model).

AC3:
  Given: Device "UPS-003" has NO device-level or group-level Temperature trigger, but its model has Temperature (> 80°F)
  When: The effective trigger is resolved
  Then: The effective trigger is "> 80°F" (model-level applies).

AC4:
  Given: Device "UPS-004" has no triggers at any level for attribute "Humidity"
  When: The effective trigger is resolved for UPS-004 / Humidity
  Then: No effective trigger exists; no alarm evaluation occurs for that attribute.

AC5:
  Given: A group-level trigger is updated from 85°F to 88°F
  When: The effective trigger is re-resolved for a device in that group (no device-level override)
  Then: The new effective trigger is "> 88°F" (reflects the update).

AC6 (performance):
  Given: 5,000 devices, each with up to 10 attributes
  When: Effective triggers are resolved for all devices (full resolution pass)
  Then: Resolution completes within `[TBD — confirm: < 5s? < 10s?]` on Medium tier hardware.

**Tasks**
- [ ] Task 1: <Backend> Design EffectiveTriggerResolver service interface (resolveForDevice, resolveForDeviceAttribute, resolveAll)
- [ ] Task 2: <Backend> Implement priority resolution algorithm (device → group → model lookup chain)
- [ ] Task 3: <Backend> Implement effective trigger cache (per device/attribute) with TTL or invalidation
- [ ] Task 4: <Backend> Implement cache invalidation triggers (on trigger CRUD, group membership change, device model change)
- [ ] Task 5: <Backend> Integrate with alarm engine (replace direct trigger lookup with effective trigger resolution)
- [ ] Task 6: <Backend> Create REST API endpoint: GET /api/v2/devices/{deviceId}/effective-triggers (returns resolved triggers with source metadata)
- [ ] Task 7: <Test> Unit tests for all priority scenarios (device wins, group wins, model wins, no trigger)
- [ ] Task 8: <Test> Performance test: resolve triggers for 5,000 devices × 10 attributes
- [ ] Task 9: <Test> Integration test: update group trigger → verify effective trigger changes for group devices

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Multi-group conflict resolution rule defined by architecture
- [ ] Alarm engine integration approach confirmed by SA team
- [ ] On-prem install/upgrade impact: alarm engine update
- [ ] Dependencies: Epic 1 data model (S1.1)
- [ ] NFRs applicable: performance (< 100ms single resolution), reliability (deterministic)
- [ ] Estimated by team

---

### STORY: S2.2 — UI: Display Inherited vs. Overridden Triggers on Device Triggers Tab

**Parent Epic**: BDCSPM-49336 Epic 2 — Trigger Priority Resolution & Device-Level Override
**BLSS Product**: Shared platform services (UI layer)

**Description**
As a Data Center Facility Manager,
I want the device Triggers tab to show all effective triggers with clear visual indicators of their source (device-level, inherited from group, inherited from model),
So that I can immediately understand where each trigger comes from and which ones can be overridden.

**Context & notes**:
- Enhance existing device Triggers tab to show "effective triggers" instead of only device-level triggers
- Each trigger row includes a source badge/icon: "Device" (direct), "Group: Closet A" (inherited), "Model: M2 Card" (inherited)
- Inherited triggers are visually distinct (e.g., lighter styling, inheritance icon)
- Overridden triggers show both the effective value and a hint of the inherited value being overridden
- Link to UX mockup: `[TBD — pending Design Epic BDCSPM-66978]`

**Acceptance Criteria**

AC1:
  Given: Device "UPS-001" has effective triggers from mixed sources (1 device-level, 2 group-inherited, 1 model-inherited)
  When: The user views the Triggers tab for UPS-001
  Then: All 4 effective triggers are listed, each with a source indicator badge.

AC2:
  Given: A trigger is inherited from group "Closet A"
  When: The user views the trigger row
  Then: The source badge shows "Group: Closet A" with a link/tooltip to the group's Triggers tab.

AC3:
  Given: A device-level trigger overrides a group-level trigger for the same attribute
  When: The user views the trigger row
  Then: The row shows the device-level value as effective, with a visual hint (e.g., "Overrides Group: Closet A at 85°F").

AC4:
  Given: A trigger is inherited from model "M2 Card"
  When: The user views the trigger row
  Then: The source badge shows "Model: M2 Card" with a link to the model's Triggers tab.

AC5:
  Given: The device has no triggers at any level
  When: The user views the Triggers tab
  Then: An empty state is shown: "No triggers configured for this device. Triggers can be defined at device, group, or model level."

**Tasks**
- [ ] Task 1: <Frontend> Update device Triggers tab to call GET /api/v2/devices/{deviceId}/effective-triggers
- [ ] Task 2: <Frontend> Implement source indicator badge component (Device/Group/Model with name)
- [ ] Task 3: <Frontend> Implement visual differentiation for inherited vs. device-level triggers
- [ ] Task 4: <Frontend> Implement "overrides" hint for device-level triggers that shadow a group/model trigger
- [ ] Task 5: <Frontend> Implement navigation links from source badges to group/model Triggers tabs
- [ ] Task 6: <Test> Component tests for source badge rendering (all 3 types)
- [ ] Task 7: <Test> E2E test: device with mixed-source triggers → verify correct display

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — `[TBD]`
- [ ] API contract defined (effective-triggers endpoint with source metadata)
- [ ] Dependencies: S2.1 (effective trigger resolver), Epic 1 (group/model triggers exist)
- [ ] NFRs applicable: usability (source immediately identifiable), performance (< 2s load)
- [ ] Estimated by team

---

### STORY: S2.3 — UI: Override Inherited Trigger at Device Level

**Parent Epic**: BDCSPM-49336 Epic 2 — Trigger Priority Resolution & Device-Level Override
**BLSS Product**: Shared platform services (UI layer)

**Description**
As a Data Center Facility Manager,
I want to override an inherited trigger (from group or model) with a device-specific value,
So that I can set a different threshold for one device that has different operational requirements than the rest of its group/model.

**Context & notes**:
- User clicks "Override" action on an inherited trigger row
- A form opens pre-populated with the inherited trigger values
- User modifies the threshold (or other fields) and saves
- System creates a device-level trigger that shadows the inherited trigger
- The inherited trigger definition at group/model level is NOT modified
- Matches Scenario 3 from BDCSPM-49336: "One device needs a different temperature threshold"

**Acceptance Criteria**

AC1:
  Given: Device "UPS-005" has an inherited group trigger "Temperature > 85°F"
  When: The admin clicks "Override" on that trigger row
  Then: A form opens pre-populated with: Attribute=Temperature, Condition=>, Threshold=85, Severity=[current], Enabled=on.

AC2:
  Given: The admin changes the threshold to 90°F and saves
  When: The override is saved
  Then: A device-level trigger is created (Temperature > 90°F); the effective trigger immediately shows the device-level value; the source badge changes to "Device (overrides Group: Closet A at 85°F)".

AC3:
  Given: The override is saved
  When: The admin navigates to the group's Triggers tab
  Then: The group trigger still shows 85°F unchanged; the device count may show "24 of 25 devices" (indicating one device has an override).

AC4 (edge case):
  Given: The admin attempts to override with the same value as the inherited trigger
  When: They save
  Then: The system accepts the override (explicit device-level declaration); `[TBD — or show warning "same as inherited value"?]`

**Tasks**
- [ ] Task 1: <Frontend> Add "Override" action button on inherited trigger rows
- [ ] Task 2: <Frontend> Implement override form (pre-populated from inherited values, editable)
- [ ] Task 3: <Frontend> Wire save to POST /api/v2/devices/{deviceId}/triggers (creates device-level trigger)
- [ ] Task 4: <Backend> Ensure device-level trigger creation invalidates effective trigger cache for that device
- [ ] Task 5: <Frontend> Update source badge after override to show "Device (overrides [source])"
- [ ] Task 6: <Test> E2E test: override group trigger at device level → verify effective trigger changes → verify group trigger unchanged

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — `[TBD]`
- [ ] API contract defined
- [ ] Dependencies: S2.1 (resolver), S2.2 (display with source)
- [ ] NFRs applicable: data integrity (group/model unaffected)
- [ ] Estimated by team

---

### STORY: S2.4 — UI: Revert Device-Level Override to Inherit from Group/Model

**Parent Epic**: BDCSPM-49336 Epic 2 — Trigger Priority Resolution & Device-Level Override
**BLSS Product**: Shared platform services (UI layer)

**Description**
As a Data Center Facility Manager,
I want to revert a device-level override back to inheriting from the group or model,
So that the device returns to using the standard threshold defined for its group/model.

**Context & notes**:
- "Revert to inherited" action on device-level triggers that shadow a group/model trigger
- Deletes the device-level trigger record
- Effective trigger falls back to group-level (or model-level if no group trigger)
- Confirmation dialog: "Revert to [source] value of [threshold]? This device will use the group/model trigger."

**Acceptance Criteria**

AC1:
  Given: Device "UPS-005" has a device-level override (Temperature > 90°F) that shadows group trigger (Temperature > 85°F)
  When: The admin clicks "Revert to inherited"
  Then: A confirmation dialog shows: "Revert to Group 'Closet A' value: Temperature > 85°F?"

AC2:
  Given: The admin confirms the revert
  When: The revert completes
  Then: The device-level trigger is deleted; the effective trigger is now "> 85°F" from group; the source badge shows "Group: Closet A".

AC3:
  Given: The device-level trigger does NOT shadow any group/model trigger (standalone device trigger)
  When: The admin views that trigger
  Then: No "Revert to inherited" action is available (only standard Delete is shown).

AC4 (edge case):
  Given: The group trigger has been updated since the device override was created (was 85°F, now 88°F)
  When: The admin reverts
  Then: The effective trigger becomes the CURRENT group value (88°F), not the value at the time of override.

**Tasks**
- [ ] Task 1: <Frontend> Add "Revert to inherited" action button on device-level triggers that shadow inherited triggers
- [ ] Task 2: <Frontend> Implement confirmation dialog showing target inherited value
- [ ] Task 3: <Frontend> Wire revert to DELETE /api/v2/devices/{deviceId}/triggers/{triggerId}
- [ ] Task 4: <Backend> Determine if a device trigger shadows an inherited trigger (API metadata)
- [ ] Task 5: <Test> E2E test: override → revert → verify effective trigger returns to group/model value

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — `[TBD]`
- [ ] Dependencies: S2.1 (resolver), S2.3 (override exists)
- [ ] NFRs applicable: data integrity (correct fallback value)
- [ ] Estimated by team

---

### STORY: S2.5 — Backend: Skip Devices Without Matching Attribute (Silent Skip + Log)

**Parent Epic**: BDCSPM-49336 Epic 2 — Trigger Priority Resolution & Device-Level Override
**BLSS Product**: Shared platform services (Triggers subsystem)

**Description**
As a system operator,
I want the trigger propagation to silently skip devices that don't have the specified attribute and log the skip for troubleshooting,
So that applying a trigger to a mixed group doesn't fail or display errors for devices that simply don't support that measurement.

**Context & notes**:
- When a group trigger for "Temperature" is applied to a group containing both UPS devices (have Temperature) and network switches (no Temperature), the switches should be skipped
- No error displayed to the user in the UI (just a summary: "Applied to X of Y devices. Z skipped.")
- Detailed skip log: structured log entry per skipped device with device_id, attribute, group_id, reason
- Log must be queryable for compliance/troubleshooting
- Matches Scenario 5 from BDCSPM-49336

**Acceptance Criteria**

AC1:
  Given: A group trigger "Temperature > 85°F" is applied to a group with 30 devices (20 have Temperature, 10 do not)
  When: Propagation completes
  Then: 20 devices receive the trigger; 10 are skipped; the API response includes: {applied: 20, skipped: 10, total: 30}.

AC2:
  Given: Propagation skips devices
  When: The admin checks the propagation summary
  Then: The UI shows "Applied to 20 of 30 devices. 10 devices skipped (attribute not available)." with an option to view details.

AC3:
  Given: Devices are skipped during propagation
  When: The system log is queried
  Then: Each skipped device has a structured log entry: {timestamp, level: INFO, event: "trigger_propagation_skip", device_id, device_name, group_id, attribute, reason: "attribute_not_available"}.

AC4:
  Given: ALL devices in a group lack the specified attribute
  When: The trigger is saved
  Then: The trigger is saved at group level (valid definition); propagation reports {applied: 0, skipped: N, total: N}; no error.

AC5 (edge case):
  Given: A device initially lacked the attribute (was skipped) but later acquires it (e.g., sensor added)
  When: The device attribute set changes
  Then: `[TBD — define: should the system retroactively apply the group trigger? Or require manual re-propagation?]`

**Tasks**
- [ ] Task 1: <Backend> Implement attribute availability check for each device during propagation
- [ ] Task 2: <Backend> Implement skip logic (continue propagation, don't fail)
- [ ] Task 3: <Backend> Return structured propagation result (applied_count, skipped_count, skipped_devices list)
- [ ] Task 4: <Backend> Implement structured logging for skipped devices (INFO level, queryable fields)
- [ ] Task 5: <Frontend> Display propagation summary in success notification (applied/skipped counts)
- [ ] Task 6: <Frontend> Optional: "View skipped devices" expandable list in propagation result
- [ ] Task 7: <Test> Unit tests for skip logic (all have attribute, none have, mixed)
- [ ] Task 8: <Test> Integration test: apply trigger to mixed group → verify correct skip behavior and log entries

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Device attribute catalog API confirmed (how to check if device has attribute)
- [ ] Log format/destination confirmed (application log file, centralized logging, or DB table?)
- [ ] Dependencies: S1.2 (propagation service)
- [ ] NFRs applicable: error handling (no errors shown for expected skips)
- [ ] Estimated by team

---

### STORY: S2.6 — UI: Display Trigger Inheritance Source Indicator

**Parent Epic**: BDCSPM-49336 Epic 2 — Trigger Priority Resolution & Device-Level Override
**BLSS Product**: Shared platform services (UI layer)

**Description**
As a Data Center Facility Manager,
I want each trigger displayed on a device to have a clear visual source indicator,
So that I can immediately distinguish between device-level, group-inherited, and model-inherited triggers.

**Context & notes**:
- Reusable UI component (badge/chip) that displays trigger source
- Three source types: "Device" (direct), "Group: [name]" (inherited), "Model: [name]" (inherited)
- Visual differentiation: different icon or color per source type
- Must be accessible (color is not the only differentiator; icon + text label)
- Follows Brightlayer Design System patterns for chips/badges

**Acceptance Criteria**

AC1:
  Given: A trigger is defined directly on the device
  When: Displayed in the trigger list
  Then: Source indicator shows icon + "Device" badge (e.g., solid blue chip).

AC2:
  Given: A trigger is inherited from device group "Closet A"
  When: Displayed in the trigger list
  Then: Source indicator shows icon + "Group: Closet A" badge (e.g., outlined green chip with group icon).

AC3:
  Given: A trigger is inherited from model "M2 Card"
  When: Displayed in the trigger list
  Then: Source indicator shows icon + "Model: M2 Card" badge (e.g., outlined purple chip with model icon).

AC4:
  Given: A screen reader user is navigating the trigger list
  When: Focus reaches the source indicator
  Then: The screen reader announces the full source (e.g., "Inherited from group Closet A").

**Tasks**
- [ ] Task 1: <Frontend> Create TriggerSourceBadge component (props: sourceType, sourceName, sourceId)
- [ ] Task 2: <Frontend> Define visual styling per source type (icon + color + text; accessible)
- [ ] Task 3: <Frontend> Integrate TriggerSourceBadge into trigger list table rows
- [ ] Task 4: <Frontend> Add aria-label for accessibility
- [ ] Task 5: <Test> Component test: render all 3 source types; verify accessibility attributes
- [ ] Task 6: <Test> Visual regression test (if applicable)

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached with badge design — `[TBD]`
- [ ] Brightlayer Design System component guidance confirmed
- [ ] Dependencies: S2.1 (effective trigger API returns source metadata)
- [ ] NFRs applicable: usability, accessibility
- [ ] Estimated by team

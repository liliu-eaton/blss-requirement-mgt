# BDCSPM-49336 — Test Scenario Matrix

## Traceability: Capability Use Cases → Epic/Story Coverage

| # | Capability Use Case (from BDCSPM-49336) | Epic | Stories | Status |
|---|---|---|---|---|
| 1 | Scenario 1: Apply Trigger to an Entire Device Group | Epic 1 | S1.3, S1.4, S1.2 | Covered |
| 2 | Scenario 2: Apply Trigger by Model | Epic 1 | S1.5, S1.6, S1.2 | Covered |
| 3 | Scenario 3: Override Group Trigger at Device Level | Epic 2 | S2.1, S2.2, S2.3, S2.4 | Covered |
| 4 | Scenario 4: Bulk Apply via Rule Selection | Epic 3 | S3.1, S3.2, S3.3 | Covered |
| 5 | Scenario 5: Error Handling (skip missing attribute) | Epic 2 | S2.5 | Covered |
| 6 | Scenario 6: Permissions | Epic 3 | S3.5, S3.6 | Covered |

## NFR Traceability

| # | Capability NFR | Epic | Stories | Target |
|---|---|---|---|---|
| 1 | Performance: ≤ 5s for 100 devices | Epic 1 | S1.2 | ≤ 5s (sync), ≤ 15s for 500 | 
| 2 | Error Handling: Skip device without attribute, no error | Epic 2 | S2.5 | Silent skip + log |
| 3 | Auditability: Changes logged for compliance | Epic 3 | S3.4, S3.7 | Append-only audit log |

## Priority Hierarchy Validation Scenarios

| # | Scenario | Device Trigger | Group Trigger | Model Trigger | Expected Effective | Epic/Story |
|---|---|---|---|---|---|---|
| 1 | All three levels defined | Temp > 90°F | Temp > 85°F | Temp > 80°F | Temp > 90°F (Device wins) | S2.1 AC1 |
| 2 | Group + Model (no device) | — | Temp > 85°F | Temp > 80°F | Temp > 85°F (Group wins) | S2.1 AC2 |
| 3 | Model only | — | — | Temp > 80°F | Temp > 80°F (Model applies) | S2.1 AC3 |
| 4 | No trigger at any level | — | — | — | No trigger (no alarm) | S2.1 AC4 |
| 5 | Override then revert | Temp > 90°F → revert | Temp > 85°F | Temp > 80°F | After revert: Temp > 85°F | S2.3, S2.4 |
| 6 | Group updated after override | Temp > 90°F | 85°F → 88°F | — | Still Temp > 90°F (device override unchanged) | S2.1 |
| 7 | Group updated, no device override | — | 85°F → 88°F | — | Temp > 88°F (inherits update) | S2.1 AC5 |

## Propagation Behavior Matrix

| # | Event | Expected Behavior | Epic/Story |
|---|---|---|---|
| 1 | Group trigger created | Apply to all group devices with matching attribute | S1.2, S1.4 |
| 2 | Group trigger updated | Update on all inheriting devices (no device override) | S1.2 |
| 3 | Group trigger deleted | Remove from all inheriting devices | S1.2, S1.4 |
| 4 | Device added to group | Inherit all applicable group triggers | S1.7 |
| 5 | Device removed from group | Retract all group-inherited triggers | S1.7 |
| 6 | New device discovered (model has triggers) | Inherit all applicable model triggers | S1.8 |
| 7 | Model trigger created | Apply to all devices of that model | S1.2, S1.6 |
| 8 | Model trigger updated | Update on all inheriting devices (no device/group override) | S1.2 |
| 9 | Model trigger deleted | Remove from all inheriting devices | S1.2, S1.6 |
| 10 | Rule created (multiple groups) | Apply trigger to all devices in all target groups | S3.1 |
| 11 | Rule updated (threshold change) | Update trigger on all devices across all target groups | S3.1 |
| 12 | Rule deleted | Retract trigger from all devices in all target groups | S3.1 |
| 13 | Group added to existing rule | Apply trigger to new group's devices | S3.1 |
| 14 | Group removed from rule | Retract trigger from that group's devices only | S3.1 |

## Permission Scenario Matrix

| # | User Role | Action | Scope | Expected Result | Story |
|---|---|---|---|---|---|
| 1 | Trigger Admin | Create trigger | Group | Allowed (201) | S3.5 AC1 |
| 2 | Trigger Admin | Create trigger | Model | Allowed (201) | S3.5 |
| 3 | Trigger Admin | Create rule | Rule | Allowed (201) | S3.5 |
| 4 | Non-admin | Create trigger | Group | Denied (403) | S3.5 AC2, S3.6 |
| 5 | Non-admin | Create trigger | Model | Denied (403) | S3.5 AC4 |
| 6 | Non-admin | Create rule | Rule | Denied (403) | S3.5 AC4 |
| 7 | Non-admin | View triggers | Group | Allowed (200, read-only) | S3.5 AC3 |
| 8 | Non-admin | Create trigger | Device | Existing permissions apply | S3.5 AC6 |
| 9 | Trigger Admin | View audit log | — | Allowed (200) | S3.7 |
| 10 | Non-admin | View audit log | — | `[TBD — confirm if read-only audit access for non-admins]` | S3.7 |

## Validation Checklist

- [x] All 6 Capability use cases covered by Epic/Story assignments
- [x] All 3 Capability NFRs have quantified targets and Story assignments
- [x] Priority hierarchy fully specified with test scenarios
- [x] Propagation behavior defined for all state change events
- [x] Permission matrix covers admin/non-admin for all scopes
- [x] Edge cases identified (partial failure, restart recovery, attribute mismatch, multi-group)
- [ ] `[TBD]` items tracked for SME validation:
  - Multi-group membership conflict resolution
  - Retroactive trigger application when device acquires attribute
  - Audit log access for non-admin users
  - Exact performance targets for 1,000+ device propagation

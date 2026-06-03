# Dynamic Analytics Quota Reporting — Zero-Code Extension for MSP & License Compliance Reports

## EPIC: Data-Driven Analytics Quota Reporting Framework

**BLSS Product(s)**: BLSS Core Platform (License Management, Reporting, MSP)  
**Parent Capability**: BDCSPM-52680 — PHASE 2 - UPS Analytics - Support for License Quota  
**Reference**: BDCSPM-62315 — Support for UPS Analytics License Quota in BLSS  
**Summary** (1-2 sentences)  
Refactor the BLSS license quota reporting architecture to a metadata-driven framework, enabling MSP quota distribution pages and License Compliance Reports to dynamically reflect any newly registered analytics algorithm (e.g., Motor Analytics, UPS Analytics, future algorithms) without requiring code changes to reports, PDF/Excel generators, or MSP UI.

---

### Background & Problem Statement

BDCSPM-62315 successfully delivered UPS Analytics license quota support. However, the implementation followed the same pattern as Motor Analytics (LF39): every new analytics algorithm requires modifications across **10+ code artifacts** (per Confluence page *"How to Create the Quota in BLSS"*):

| Layer | Files/Functions Requiring Change |
|-------|----------------------------------|
| Database types | `rpt.t_sys_license_allocation`, `rpt.t_sys_license_compliance_vcom` |
| Database functions | `rpt.f_get_rptdata_sys_license_allocation()`, `rpt.f_get_rptdata_sys_license_compliance_vcom_report()`, `rpt.f_get_rptdata_sys_license_device()` |
| Java (PDF reports) | `LicenseAllocation.java`, `LicenseComplianceVCOM.java` |
| Java (Excel reports) | `ExportColumn.java`, `RunReportDao#exportVCOMLicenseReport`, `RunReportDao#getLicenseReportFiter`, `RunReportDao#setLicenseAllocationData` |
| Department management | `SysComponent#exportDepartments`, `UserDao#checkDeparmentFRL`, `UserDao#saveDepartment`, `UserDao#getDepartmentList` |
| UI config | `department.json` |

Each time a new analytics algorithm is introduced (e.g., Battery Health Analytics, PDU Analytics, Transformer Analytics), the team must repeat this entire chain of changes for MSP and compliance reporting alone — adding significant lead time, regression risk, and tight coupling between quota definition and report rendering.

---

### Proposed Solution

Introduce a **Quota Type Registry** and refactor the reporting/MSP layers to consume quota metadata dynamically:

#### 1. Quota Type Registry (Database)

Create a master registry table that defines all analytics quota types:

```
sys.quota_type_registry (
    quota_type_id       SERIAL PRIMARY KEY,
    quota_code          VARCHAR(50) NOT NULL UNIQUE,  -- e.g., 'MOTOR_ANALYTICS', 'UPS_ANALYTICS'
    display_name        VARCHAR(100) NOT NULL,        -- e.g., 'Motor Analytics', 'UPS Analytics'
    license_feature_id  VARCHAR(10) NOT NULL,         -- e.g., 'LF39', 'LF43'
    device_category     VARCHAR(50) NOT NULL,         -- e.g., 'motor', 'ups'
    sort_order          INT DEFAULT 0,
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMP DEFAULT NOW()
)
```

When a new analytics algorithm is approved, a **single row insert** to this registry enables end-to-end reporting.

#### 2. Dynamic Report Data Functions

Refactor `rpt.f_get_rptdata_sys_license_allocation()` and `rpt.f_get_rptdata_sys_license_compliance_vcom_report()` to:
- Query `sys.quota_type_registry` for all active quota types
- Dynamically pivot quota data (total, used, available) per registered type
- Return a flexible result set (row-per-quota-type rather than column-per-quota-type)

#### 3. Dynamic PDF/Excel Report Rendering

Refactor `LicenseAllocation.java`, `LicenseComplianceVCOM.java`, and `ExportColumn.java` to:
- Read active quota types from the registry at report-generation time
- Dynamically add columns/rows for each registered quota type
- Use `display_name` from the registry as column headers

#### 4. Dynamic MSP Quota Distribution

Refactor the MSP department management layer to:
- Dynamically render quota allocation columns in the department management grid based on active registry entries
- Support filtering, sorting, and import/export for any registered quota type
- Dynamically generate quota assignment fields in the department edit form

#### 5. Dynamic Department Import/Export

Refactor `SysComponent#exportDepartments` and related import logic to:
- Include all active quota types in export templates (headers generated from registry)
- Accept and validate imported quota values for any registered type

---

### User Personas

- **Product Manager / License Administrator**: Registers a new analytics algorithm quota via registry entry; expects reports to reflect it immediately
- **Compliance Auditor**: Views License Compliance Reports that dynamically include all registered quota types without waiting for a software release
- **MSP Customer**: Distributes any analytics quota across departments via the same existing UI
- **Development Team**: No longer needs to modify report code when a new quota type is added

---

### Business Value

- **Zero-code extension**: Adding a new analytics algorithm quota requires only a database registry entry — no code deployment, no regression testing of report code
- **Reduced lead time**: New quota visible in reports/MSP within minutes (registry insert) vs. weeks (development cycle)
- **Reduced regression risk**: Report generation code is algorithm-agnostic; no risk of breaking existing quotas when adding new ones
- **Future-proof scalability**: Supports unlimited quota types without architectural change
- **Consistent experience**: All quota types appear uniformly in reports, exports, and MSP distribution

---

### Key Workflows

1. **Register New Quota Type**: Admin/DevOps inserts a row into `sys.quota_type_registry` (via migration script or admin tool) specifying `quota_code`, `display_name`, `license_feature_id`, `device_category`
2. **License Compliance Report**: Report generator queries registry → dynamically builds compliance rows for all active quota types → PDF/Excel output includes the new quota automatically
3. **License Allocation Report**: Allocation report dynamically includes total/assigned/available for all registered quota types
4. **MSP Department Distribution**: Department management grid automatically shows a column for the new quota type; admin can allocate quota to departments
5. **Department Export/Import**: Export template dynamically includes all registered quota columns; import validates against registry

---

### Acceptance Criteria

- [ ] AC1: A `sys.quota_type_registry` table exists and contains entries for Motor Analytics (LF39) and UPS Analytics (LF43) as seed data
- [ ] AC2: Adding a new row to `sys.quota_type_registry` (with valid `license_feature_id`) causes the License Compliance Report to include that quota type's usage (total, used, available, compliance status) — **without any code changes or redeployment**
- [ ] AC3: The License Allocation Report dynamically includes all active quota types from the registry with correct total/assigned/available values
- [ ] AC4: MSP Department Management grid dynamically displays quota distribution columns for all active registry entries
- [ ] AC5: Department export file includes columns for all active quota types; department import file accepts and validates quota values for all active quota types
- [ ] AC6: Setting `is_active = FALSE` on a registry entry hides that quota from reports and MSP UI (soft disable)
- [ ] AC7: Report PDF and Excel outputs use `display_name` from the registry as column headers (localization-ready)
- [ ] AC8: Existing Motor Analytics and UPS Analytics reporting behavior is fully preserved (backward compatible)
- [ ] AC9: Performance: License Compliance Report generation time does not degrade by more than 10% with up to 20 registered quota types

---

### Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|----------|-------------|--------|----------|
| Performance | Report generation latency with N quota types (N ≤ 20) | < 10% degradation vs. current | P1 |
| Performance | Registry lookup cached per report generation session | Single query per report run | P2 |
| Scalability | Number of supported quota types | ≥ 20 simultaneous active types | P1 |
| Backward Compatibility | Existing Motor/UPS Analytics data and reports | 100% preserved | P0 |
| Maintainability | Lines of code changed to add a new quota type | 0 (registry insert only) | P0 |
| Data Integrity | Quota totals in dynamic reports match current hard-coded reports | Exact numerical match | P0 |
| Operability | Registry entry validation (reject invalid license_feature_id) | Enforced via FK or check constraint | P1 |
| Testability | Automated regression test for report output per quota type | Per-type assertion | P1 |

---

### Technical Design Considerations

1. **Row-based vs. Column-based reports**: Current reports use fixed columns per quota type. The refactored approach should use row-based data internally (one row per quota type per entity) and pivot dynamically for presentation — this avoids schema changes when quota types are added.

2. **Caching**: The quota type registry is low-cardinality and rarely changes. Cache it in application memory with TTL-based invalidation (e.g., 5 minutes) to avoid per-request DB calls.

3. **Migration strategy**: 
   - Phase 1: Create registry table, seed with Motor Analytics + UPS Analytics entries
   - Phase 2: Refactor report functions to read from registry (dual-path with feature flag)
   - Phase 3: Refactor Java report renderers and MSP UI
   - Phase 4: Remove legacy hard-coded paths after validation

4. **Admin tooling**: Consider a lightweight admin UI or CLI command to manage the quota type registry (CRUD operations) in addition to direct DB inserts.

---

### Dependencies

| Dependency | Team | Description |
|-----------|------|-------------|
| License Feature ID allocation | Optimus (License team) | New LF codes must be allocated before registry entry |
| Quota charging rules | Database team | `sys.f_check_quota.sql` and related device-type procedures still need per-type logic for quota enforcement (out of scope for this epic — only reporting is dynamic) |
| MSP UI framework | Frontend team | Department management grid must support dynamic column rendering |

---

### Scope Clarification

**In scope**:
- Dynamic reporting (License Compliance, License Allocation, License Detail)
- Dynamic MSP department quota distribution display and management
- Dynamic department export/import
- Quota type registry design and seeding

**Out of scope** (require separate epics):
- Dynamic quota **enforcement** rules (the `sys.f_check_quota` logic per device category still needs device-type-specific SQL — see "BLSS Charge Quota Rule Update for New Device Types")
- Dynamic device management UI (Monitor Config tab slider per algorithm type)
- License Portal changes (Optimus team manages LF code generation independently)
- Quota charge rules for new device categories

---

### Story Decomposition (Suggested)

| # | Story | Description |
|---|-------|-------------|
| 1 | Design & create `sys.quota_type_registry` table | Schema design, constraints, seed data for LF39/LF43 |
| 2 | Refactor `rpt.f_get_rptdata_sys_license_compliance_vcom_report()` | Dynamic pivot from registry; backward-compatible output |
| 3 | Refactor `rpt.f_get_rptdata_sys_license_allocation()` | Dynamic allocation data from registry |
| 4 | Refactor License Compliance PDF renderer | `LicenseComplianceVCOM.java` reads registry for dynamic columns |
| 5 | Refactor License Allocation PDF renderer | `LicenseAllocation.java` reads registry for dynamic columns |
| 6 | Refactor License Report Excel export | `ExportColumn`, `RunReportDao` methods made registry-aware |
| 7 | Refactor MSP Department Management grid | Dynamic quota columns from registry |
| 8 | Refactor Department Export/Import | Dynamic template headers and validation |
| 9 | Validation & regression testing | Verify Motor/UPS Analytics parity; add new test quota type |
| 10 | Documentation & runbook | How to register a new quota type (operational guide) |

---

### Success Metrics

- A new analytics algorithm quota type can be reported in MSP and License Compliance Report within **1 hour** of registry entry (vs. current ~2-sprint development cycle)
- **Zero** code changes required in reporting/MSP layer for future quota type additions
- **Zero** regression in existing Motor Analytics / UPS Analytics report data after migration

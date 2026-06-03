# BDCSPM-74129 — Flexible License Management for Decoupled BLSS, Plugin Quota, and Algorithm Feature Control

## EPIC 1: Data-Driven Licensing & Quota Enforcement Engine for Dynamic Algorithm Registration

**BLSS Product(s)**: BLSS Core Platform (License Engine, Quota Enforcement, Plugin Framework)  
**Parent Capability**: BDCSPM-66856 — BLSS Analytics Plugin Platform Foundation (DAC Framework)  
**Summary** (1-2 sentences)  
Build a metadata-driven licensing and quota enforcement engine that enables BLSS to automatically validate license entitlement, enforce quota limits, and control algorithm feature access for any newly registered analytics algorithm — through configuration-only registration, without requiring additional code development.

---

### Description

- **Problem statement**:  
  Currently, each new analytics algorithm (Motor Analytics LF39, UPS Analytics LF43) requires bespoke code changes across multiple layers to support licensing and quota enforcement:

  | Enforcement Aspect | Current Code Changes Required |
  |---|---|
  | License feature activation | Hard-coded LF flag checks in `sys.f_check_quota.sql` |
  | Quota counting rules | Per-algorithm SQL procedures (e.g., `sys.f_get_quota41_core_device_entity.sql`, `sys.f_get_quota43_remote_alarm.sql`) |
  | Device enablement validation | Custom logic in Monitor Config, Manage Devices, bulk import |
  | Quota decrement/enforcement | Separate enforcement logic per algorithm (BDCSPM-67839 pattern) |
  | Feature enable/disable in UI | Hard-coded slider per algorithm type |
  | Backend API validation | Per-algorithm entitlement checks |
  | MSP department quota distribution | Per-algorithm column in department management |
  | License compliance reporting | Per-algorithm column in reports |

  Per Confluence ["BLSS Charge Quota Rule Update for New Device Types"](https://eaton-corp.atlassian.net/wiki/spaces/BLDCS/pages/702677993), adding a new device/algorithm type requires updating 3-6 database procedures per category. Per ["How to Create the Quota in BLSS"](https://eaton-corp.atlassian.net/wiki/spaces/BLDCS/pages/554405305), the reporting layer alone requires 10+ artifact changes.

  BDCSPM-74129 envisions a plugin-based architecture where new analytics algorithms can be installed and enabled without modifying the core BLSS platform. However, the current tightly-coupled license enforcement mechanism is the primary blocker — every new algorithm still requires a dedicated development cycle for licensing/quota integration.

- **Proposed solution**:  
  Design and implement a **Rule-Driven License & Quota Enforcement Engine** that:

  1. **Algorithm Quota Registry** — A centralized metadata store where each analytics algorithm declares its licensing and quota rules
  2. **Generic Quota Enforcement Engine** — A single, parameterized enforcement pathway that reads rules from the registry and applies them universally
  3. **Dynamic Feature Gating** — License feature activation/deactivation driven by registry metadata rather than hard-coded flags
  4. **Generic Quota Charging Rules** — A rule-based quota calculation engine that evaluates device-to-quota-type mappings from the registry
  5. **Dynamic UI Feature Controls** — Monitor Config and Device Management pages dynamically render algorithm-specific sliders/toggles based on registered capabilities
  6. **Backend API Entitlement Guard** — A generic interceptor that validates any algorithm operation against the registry before execution

  When a new analytics algorithm is registered (via plugin installation or configuration), it provides a **Quota Rule Descriptor** specifying:
  - Which license feature (LF code) gates the algorithm
  - Quota counting rule (e.g., "1 device = 1 quota unit", or "1 extension + 1 analytics")
  - Device types eligible for this algorithm
  - Enable/disable control behavior
  - MSP distribution granularity

  The enforcement engine consumes this descriptor and automatically provides full license enforcement without any new code.

- **User personas**:
  - **Product Manager**: Defines and registers new analytics algorithms with licensing rules
  - **System Administrator**: Installs plugins / enables algorithms — quota enforcement happens automatically
  - **BLSS User**: Enables analytics on devices — system blocks if quota exceeded, without knowing which algorithm is new vs. legacy
  - **MSP Customer**: Distributes quota for any algorithm across departments — UI dynamically adapts
  - **Development Team**: Delivers new algorithms without touching license/quota enforcement code

- **Business value**:
  - **Zero enforcement code for new algorithms**: New analytics algorithm licensed and quota-enforced through registration only
  - **Plugin-ready architecture**: Aligns with BDCSPM-66856 DAC Framework goal — plugins can be developed independently
  - **Reduced TTM**: New algorithm from "defined" to "enforceable" in hours (config) vs. weeks (development)
  - **Consistent enforcement behavior**: All algorithms share the same validation logic — no per-algorithm bugs
  - **Future scalability**: Supports unlimited algorithm types (Battery Health, PDU Analytics, Transformer Analytics, etc.)
  - **Backward compatible**: Existing Motor Analytics (LF39) and UPS Analytics (LF43) enforcement fully preserved

- **Key workflows**:

  **Workflow 1: Register a New Analytics Algorithm**
  1. Algorithm developer/PM registers quota rule descriptor (via plugin manifest or admin registry entry)
  2. System validates the descriptor (LF code exists in license, device types valid, quota rule parseable)
  3. Registration complete — enforcement is immediately active

  **Workflow 2: Enable Algorithm on Device (Manual)**
  1. User navigates to device Monitor Config tab
  2. UI dynamically renders algorithm sliders based on all registered algorithms applicable to this device type
  3. User toggles new algorithm ON
  4. Enforcement engine: checks LF feature enabled → checks available quota → decrements quota → enables
  5. If quota insufficient: blocks operation, shows "Quota exceeded for [Algorithm Display Name]" warning

  **Workflow 3: Enable Algorithm via Bulk Import**
  1. User exports device template — template dynamically includes columns for all registered algorithms
  2. User enables algorithm for multiple devices in spreadsheet
  3. On import: enforcement engine validates total quota impact across all modified rows
  4. If non-compliant: rejects import, reports which algorithm quota(s) would be exceeded

  **Workflow 4: Disable Algorithm / Release Quota**
  1. User disables algorithm on device (manual or bulk)
  2. Enforcement engine increments available quota per registered charging rule
  3. Quota availability updated in real-time

  **Workflow 5: License Feature Not Present**
  1. Algorithm registered but LF code not present in installed license
  2. UI hides algorithm slider / shows "Not Licensed" badge
  3. Backend API rejects any enable attempt with 403 + descriptive message

  **Workflow 6: MSP Quota Distribution**
  1. MSP admin views department management — all registered algorithm quotas shown dynamically
  2. Admin allocates quota for new algorithm to departments
  3. Department-level enforcement: department users cannot exceed their allocated share

- **Cross-product touchpoints**:
  - License Portal (Optimus team): Must generate LF codes for new algorithms — but no BLSS-side code change needed
  - Plugin Framework (DAC): Plugins declare their quota rule descriptor in manifest
  - Device Management: Monitor Config and Manage Devices consume registry for dynamic rendering
  - Reporting: License Compliance/Allocation reports consume registry (complement to dynamic reporting epic)
  - Backup/Restore: Quota state and registry included in backup scope

- **On-prem operational impact**:
  - Install/upgrade impact: Engine deployed as platform service; existing Motor/UPS enforcement migrated to registry-driven mode
  - Sizing impact: Minimal — rule evaluation is lightweight (in-memory registry lookup + single DB quota check)
  - Network impact: None — all enforcement is local
  - Backup impact: `sys.algorithm_quota_registry` and quota state tables included in backup scope
  - Plugin installation: New algorithms available after plugin install + service restart (no DB migration required)

---

### Technical Design

#### 1. Algorithm Quota Registry Schema

```sql
sys.algorithm_quota_registry (
    algorithm_id        SERIAL PRIMARY KEY,
    algorithm_code      VARCHAR(50) NOT NULL UNIQUE,     -- 'MOTOR_ANALYTICS', 'UPS_ANALYTICS', 'BATTERY_HEALTH'
    display_name        VARCHAR(100) NOT NULL,           -- 'Motor Analytics', 'UPS Analytics'
    license_feature_id  VARCHAR(10) NOT NULL,            -- 'LF39', 'LF43'
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMP DEFAULT NOW()
);

sys.algorithm_quota_rules (
    rule_id             SERIAL PRIMARY KEY,
    algorithm_id        INT REFERENCES sys.algorithm_quota_registry(algorithm_id),
    rule_type           VARCHAR(30) NOT NULL,            -- 'DEVICE_ENABLE', 'BULK_IMPORT', 'CONVERSION'
    eligible_device_types   VARCHAR[] NOT NULL,          -- '{MHM,UPS,PDU}'
    quota_charge_per_device INT DEFAULT 1,              -- units consumed per device enabled
    prerequisite_quota_type VARCHAR(50),                 -- e.g., 'EXTENSION_MONITORING' (consumes 1 ext + 1 algo)
    enforcement_mode    VARCHAR(20) DEFAULT 'STRICT',   -- 'STRICT' (block), 'WARN' (allow with warning)
    max_devices_per_operation INT,                       -- optional bulk limit
    description         TEXT
);

sys.algorithm_feature_control (
    control_id          SERIAL PRIMARY KEY,
    algorithm_id        INT REFERENCES sys.algorithm_quota_registry(algorithm_id),
    ui_control_type     VARCHAR(30) DEFAULT 'SLIDER',   -- 'SLIDER', 'CHECKBOX', 'HIDDEN'
    ui_placement        VARCHAR(50) DEFAULT 'MONITOR_CONFIG', -- where in UI to render
    api_endpoint_pattern VARCHAR(200),                   -- API path pattern to guard
    sort_order          INT DEFAULT 0
);
```

#### 2. Generic Quota Enforcement Engine (Pseudocode)

```
function enforceQuota(deviceId, algorithmCode, operation):
    registry = loadAlgorithmRegistry(algorithmCode)
    if registry == null: return REJECT("Unknown algorithm")
    
    // 1. License feature check
    if not isLicenseFeatureActive(registry.license_feature_id):
        return REJECT("Feature not licensed")
    
    // 2. Load applicable rule
    rule = loadQuotaRule(registry.algorithm_id, operation)
    if rule == null: return REJECT("No rule defined for this operation")
    
    // 3. Device type eligibility
    deviceType = getDeviceType(deviceId)
    if deviceType not in rule.eligible_device_types:
        return REJECT("Device type not eligible for this algorithm")
    
    // 4. Quota availability check
    currentUsed = getQuotaUsed(registry.algorithm_id, departmentScope)
    totalCapacity = getQuotaCapacity(registry.algorithm_id, departmentScope)
    chargeUnits = rule.quota_charge_per_device
    
    if (currentUsed + chargeUnits) > totalCapacity:
        if rule.enforcement_mode == 'STRICT':
            return REJECT("Quota exceeded: {used}/{total} for {display_name}")
        else:
            return WARN("Quota will be exceeded")
    
    // 5. Prerequisite check (e.g., Motor Analytics requires Extension Monitoring)
    if rule.prerequisite_quota_type:
        if not hasPrerequisiteQuota(deviceId, rule.prerequisite_quota_type):
            return REJECT("Prerequisite quota not available")
    
    // 6. Execute
    chargeQuota(registry.algorithm_id, chargeUnits, deviceId)
    return ALLOW
```

#### 3. Dynamic UI Rendering

```
function renderMonitorConfigSliders(deviceId):
    deviceType = getDeviceType(deviceId)
    registeredAlgorithms = getAlgorithmsForDeviceType(deviceType)
    
    for each algorithm in registeredAlgorithms:
        control = loadFeatureControl(algorithm.algorithm_id)
        licenseActive = isLicenseFeatureActive(algorithm.license_feature_id)
        
        render control.ui_control_type:
            label = algorithm.display_name
            enabled = licenseActive AND hasAvailableQuota(algorithm)
            tooltip = licenseActive ? "Quota: {used}/{total}" : "Not Licensed"
```

#### 4. Migration Strategy

| Phase | Scope | Risk |
|-------|-------|------|
| Phase 1 | Create registry tables, seed Motor Analytics (LF39) + UPS Analytics (LF43) descriptors | Low — data only |
| Phase 2 | Implement generic enforcement engine with feature flag (dual-path: old + new) | Medium — logic change |
| Phase 3 | Migrate Monitor Config & Manage Devices to dynamic rendering | Medium — UI change |
| Phase 4 | Migrate bulk import/export to registry-driven validation | Medium — import logic |
| Phase 5 | Migrate MSP department quota distribution to dynamic mode | Medium — MSP logic |
| Phase 6 | Remove legacy hard-coded enforcement after full regression | Low — cleanup |
| Phase 7 | Plugin manifest integration (algorithm self-registration on install) | Low — extension |

---

### Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|----------|-------------|--------|----------|
| Performance | Quota enforcement check latency (single device enable) | < 50ms at P95 | P0 |
| Performance | Bulk import quota validation (1000 devices) | < 5s at P95 | P1 |
| Performance | Registry lookup cached in memory | Single DB hit per session | P1 |
| Scalability | Number of registered algorithms supported simultaneously | ≥ 50 | P1 |
| Scalability | Quota enforcement under concurrent requests | Thread-safe, serializable per algorithm | P0 |
| Backward Compatibility | Motor Analytics (LF39) enforcement behavior | 100% identical to current | P0 |
| Backward Compatibility | UPS Analytics (LF43) enforcement behavior | 100% identical to current | P0 |
| Reliability | Quota count consistency (charge/release atomicity) | Zero drift under concurrent operations | P0 |
| Reliability | Enforcement engine available when plugin service down | Graceful degradation — reject new enables, existing devices unaffected | P1 |
| Security | Unauthorized API calls to licensed endpoints | 403 Forbidden with descriptive message | P0 |
| Security | UI hides unlicensed algorithm controls | No visual leakage of unlicensed capabilities | P0 |
| Maintainability | Code changes required to add a new algorithm | 0 lines (registry entry + plugin manifest only) | P0 |
| Testability | Enforcement engine testable with mock registry entries | Unit-testable without real license | P1 |
| Operability | Registry modification audit-logged | Every insert/update/delete logged | P1 |

---

### Acceptance Criteria (Epic-level)

- [ ] AC1: An `sys.algorithm_quota_registry` table exists with Motor Analytics (LF39) and UPS Analytics (LF43) seeded as baseline entries, including their respective quota rules.
- [ ] AC2: Registering a new algorithm (e.g., "Battery Health Analytics" / LF99) via a single registry entry + quota rule causes the enforcement engine to automatically validate entitlement and enforce quota for that algorithm — **without any code changes or redeployment**.
- [ ] AC3: When a user enables a registered algorithm on an eligible device and quota is available, the system decrements quota and enables the feature — identical behavior to current Motor/UPS Analytics.
- [ ] AC4: When enabling an algorithm would exceed available quota, the system blocks the operation (STRICT mode) with a meaningful warning message that includes the algorithm display name and quota counts.
- [ ] AC5: Bulk import validates quota for all registered algorithms; import is rejected if any algorithm quota would be exceeded, with a report identifying the violating algorithm(s).
- [ ] AC6: Device Monitor Config page dynamically renders enable/disable controls for all registered algorithms applicable to the device type, respecting license feature presence.
- [ ] AC7: If a license feature (LF code) is not present in the installed license, the corresponding algorithm's UI controls are hidden and backend API calls return 403.
- [ ] AC8: MSP department quota distribution page dynamically includes allocation fields for all registered algorithms.
- [ ] AC9: Disabling an algorithm on a device correctly releases quota (increments available) per the charging rule.
- [ ] AC10: Concurrent enable operations for the same algorithm maintain quota count consistency (no over-provisioning due to race conditions).
- [ ] AC11: Existing Motor Analytics and UPS Analytics enforcement behavior is fully preserved after migration (zero regression).
- [ ] AC12: A plugin manifest can declare its algorithm quota descriptor, triggering automatic registration upon plugin installation.

---

### Scope Clarification

**In scope**:
- Generic quota enforcement engine (license validation, quota check, charge/release)
- Algorithm quota registry schema and API
- Dynamic UI rendering for device enable/disable controls (Monitor Config, Manage Devices)
- Dynamic bulk import/export with enforcement
- MSP department quota distribution (dynamic)
- Plugin manifest integration for self-registration
- Migration of Motor Analytics and UPS Analytics to registry-driven enforcement
- Backend API entitlement guard (generic interceptor)

**Out of scope** (covered by companion epics or separate efforts):
- Dynamic reporting (License Compliance, License Allocation reports) — see "Dynamic Quota Reporting" epic
- License Portal changes (Optimus team generates LF codes independently)
- Algorithm execution logic (data collection, analytics processing) — plugin responsibility
- License generation and purchase workflow
- New algorithm business logic / domain-specific rules

---

### Dependencies

| Dependency | Team | Description |
|-----------|------|-------------|
| License Feature ID allocation | Optimus (License team) | New LF codes must be provisioned before algorithm registration |
| DAC Plugin Framework | Platform team (BDCSPM-66856) | Plugin manifest format must support quota rule descriptor |
| Device Type registry | Device Management team | Eligible device types must be queryable for rule matching |
| Monitor Config UI framework | Frontend team | Must support dynamic control rendering from registry |
| Existing quota enforcement | Titan/Database team | Migration requires understanding of current `sys.f_check_quota.sql` logic |

---

### Relationship to BDCSPM-62315 & BDCSPM-74129

| Aspect | BDCSPM-62315 (UPS Analytics Quota) | BDCSPM-74129 (This Epic - Dynamic Enforcement) |
|--------|------|------|
| Approach | Hard-coded per-algorithm | Registry-driven, algorithm-agnostic |
| New algorithm effort | ~2 sprints of development | ~1 hour (registry entry) |
| Enforcement logic | Algorithm-specific SQL + Java | Generic engine with parameterized rules |
| UI controls | Hard-coded sliders per algorithm | Dynamically rendered from registry |
| Testing scope | Full regression per new algorithm | Engine test once; new algorithms covered automatically |
| Scalability | Linear cost growth per algorithm | Constant cost regardless of algorithm count |

---

### Story Decomposition (Suggested)

| # | Story | Description | Estimate |
|---|-------|-------------|----------|
| 1 | Design `sys.algorithm_quota_registry` & rule schema | Schema, constraints, seed data for LF39/LF43 | S |
| 2 | Implement Generic Quota Enforcement Engine | Core enforcement logic: license check → quota check → charge/release | L |
| 3 | Migrate Motor Analytics (LF39) to registry-driven enforcement | Move from hard-coded `sys.f_check_quota.sql` to generic engine | M |
| 4 | Migrate UPS Analytics (LF43) to registry-driven enforcement | Validate UPS charging rule through generic engine | M |
| 5 | Dynamic Monitor Config rendering | Device config page renders algorithm sliders from registry | M |
| 6 | Dynamic Manage Devices form | Manage Devices enables algorithms from registry | M |
| 7 | Dynamic bulk import/export enforcement | Import template + validation from registry | L |
| 8 | Backend API entitlement interceptor | Generic guard for algorithm API endpoints | M |
| 9 | MSP department dynamic quota distribution | Department management dynamically includes all algorithms | M |
| 10 | Plugin manifest integration | Self-registration from plugin manifest on install | S |
| 11 | Concurrency & atomicity hardening | Race condition prevention for quota charge/release | M |
| 12 | Regression testing & validation | Verify Motor/UPS parity; test with mock new algorithm | L |

---

### Success Metrics

- A new analytics algorithm achieves full license enforcement + quota management within **1 hour** of registration (vs. current ~2 sprint development cycle)
- **Zero** lines of enforcement code changed for future algorithm additions
- **Zero** regression in existing Motor Analytics / UPS Analytics quota enforcement after migration
- Enforcement engine handles **≥ 100 concurrent enable requests** without quota drift
- Time-to-market for new analytics algorithms reduced by **80%** (licensing no longer on critical path)

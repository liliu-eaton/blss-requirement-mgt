# BDCSPM-37662 — Multiple SNMPv3 Trap Usernames in BLSS

## EPIC 1: Protocol-Qualified SNMPv3 Trap Receiver Configuration and Processing

**BLSS Product(s)**: Shared platform services (SNMP trap receiver configuration, persistence, and Probe Server / RDG-based SNMPv3 processing)  
⚠️ NEEDS VALIDATION: Confirm whether this capability should remain under Shared platform services only, or also be mapped to specific BLSS products that expose SNMP trap workflows.

**Summary** (1-2 sentences)  
Enable BLSS administrators and integrators to configure and process multiple SNMPv3 trap receiver entries that share the same username when the authentication and/or privacy protocol differs, so customers do not need to rename field-device users solely to satisfy BLSS validation rules and BLSS still resolves traps deterministically at runtime.

---

### Description

- **Problem statement**: The current SNMP V3 Trap Receiver Settings flow assumes usernames must be globally unique. Customers deploying mixed network card types or phased SNMPv3 security migrations are forced to change device-side usernames even when the devices legitimately use the same username with different supported security profiles. Even if configuration is relaxed, runtime processing can still mis-handle inbound SNMPv3 traps if matching logic assumes username uniqueness.

- **Proposed solution**: Update the SNMP V3 Trap Receiver Settings UX, persistence rules, validation logic, and Probe Server / RDG-based SNMPv3 processing so uniqueness and matching are enforced by the credential tuple of username + authentication protocol + privacy protocol, instead of username alone. The end-to-end flow must continue to protect passwords at rest, fail closed on unsupported or unmatched tuples, and ship with regression, performance, and rollout readiness coverage.

- **User personas**: System Integrator / Partner, On-Prem IT / System Admin, Eaton Field Service Engineer, Operations / NOC Engineer, Eaton Support Team

- **Business value**: Reduces deployment friction, lowers support effort tied to forced device reconfiguration, preserves deterministic trap handling for mixed-security customer deployments, and aligns BLSS credential modeling more closely with customer SNMPv3 deployment practices.

- **Key workflows**:
  1. Admin opens SNMP V3 Trap Receiver Settings and adds a new entry for a device family using an existing username.
  2. Admin selects a supported authentication protocol and privacy protocol combination that differs from an existing entry.
  3. System accepts the save when the tuple is unique, masks the saved secret, and persists the entry securely.
  4. Probe Server receives an inbound SNMPv3 trap and resolves the correct credential entry by the full protocol-qualified tuple.
  5. If an incoming trap does not map to a supported configured tuple, BLSS fails closed and emits sanitized diagnostics without selecting an incorrect credential.
  6. Support, QA, and Lifecycle Services teams use documented regression evidence and rollout guidance to validate successful on-prem adoption.

- **Cross-product touchpoints**: The configuration is consumed by shared SNMP trap processing services used across BLSS deployments that ingest SNMPv3 traps, and shared SNMPv3 trap ingestion may feed downstream alarming or monitoring workflows in multiple BLSS products.  
  ⚠️ NEEDS VALIDATION: Confirm whether DCPM, Distributed IT Performance Management, and Brightlayer Power all rely on the same trap receiver configuration path and which downstream regression suites must be included.

- **UX/UI references**: Jira issue BDCSPM-37662 and related BAT Epic BDCSPM-67948 reference the existing “SNMP V3 Trap Receiver Settings” workflow. `[TBD — attach approved UI mockup, annotated screenshot, or runtime sequence diagram]`

- **On-prem operational impact**:
  - Install/upgrade impact: Existing persisted SNMPv3 trap receiver configurations must remain valid after upgrade; any schema/index or runtime component change requires rollback coverage and documented upgrade order.
  - Sizing impact: Negligible CPU/RAM impact is expected for configuration storage, and trap lookup overhead must remain within the current supported baseline. `[TBD — confirm with architecture team]`
  - Network impact: None. This Epic does not add ports, protocols, or external dependencies.
  - Backup impact: Configuration backup/restore must preserve multiple entries sharing the same username when their protocol tuple differs, and restored configurations must continue to resolve traps correctly at runtime.

---

### Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Security | Stored SNMPv3 passwords remain encrypted at rest and are never redisplayed in cleartext after save | 100% of saved credentials | P0 |
| Performance | Create/edit validation for an SNMPv3 trap receiver entry completes within acceptable UI latency on Medium reference hardware | < 2s at P95 with up to 100 configured SNMPv3 trap receiver entries `[TBD — confirm baseline]` | P1 |
| Performance | SNMPv3 trap credential resolution adds minimal overhead relative to the current baseline | <= 5% increase at P95 versus current production baseline on Medium reference hardware `[TBD — confirm baseline]` | P0 |
| Performance | Credential lookup scales predictably as the number of configured SNMPv3 entries grows | Lookup time remains linear from 1 to 100 configured SNMPv3 entries in benchmark coverage | P1 |
| Reliability | Duplicate detection uses the full tuple of username + authentication protocol + privacy protocol | 100% enforcement in create and edit flows | P0 |
| Reliability | Runtime always resolves the correct credential tuple or fails closed | 100% deterministic outcome in positive and negative test cases | P0 |
| Reliability | Existing unique-username deployments show no regression in SNMPv3 trap reception after the change | 0 Sev1/Sev2 regressions in release qualification `[TBD — confirm release gate]` | P0 |
| Compatibility | Existing valid SNMPv3 trap receiver configurations remain loadable and editable after upgrade | 100% backward compatibility for current supported releases `[TBD — confirm version floor]` | P0 |
| Compatibility | Behavior is consistent across all supported on-premises architectures that use Probe Server / RDG-based SNMPv3 processing | 100% pass rate on the supported architecture matrix `[TBD — confirm matrix]` | P1 |
| Security | Secrets are not written to application logs, audit logs, runtime diagnostics, or UI notifications in plaintext | 0 plaintext secret exposure events in regression coverage | P0 |
| Usability | Users can distinguish saved entries that share the same username by their protocol metadata in the configuration view | 100% of saved entries display auth/privacy metadata alongside the username `[TBD — confirm UI treatment]` | P2 |

---

### Acceptance Criteria (Epic-level)

- [ ] AC1: An administrator can save multiple SNMPv3 trap receiver entries with the same username when each saved entry has a different supported authentication and/or privacy protocol combination.
- [ ] AC2: The system rejects creation or editing of a second entry when the username + authentication protocol + privacy protocol tuple matches an existing entry, regardless of password value.
- [ ] AC3: Saved SNMPv3 passwords remain masked in the UI, encrypted at rest, and excluded from logs and diagnostics in plaintext.
- [ ] AC4: Incoming SNMPv3 traps are matched using the full username + authentication protocol + privacy protocol tuple rather than username alone.
- [ ] AC5: BLSS correctly processes supported mixed-protocol deployments where multiple configured entries share the same username.
- [ ] AC6: If an incoming trap does not match a supported configured tuple, BLSS does not map it to the wrong credential and emits sanitized diagnostics.
- [ ] AC7: Existing supported deployments remain available after upgrade without requiring device-side username changes or introducing runtime regression.
- [ ] AC8: Release qualification includes documented regression, performance, security, and operational evidence suitable for on-premises deployment teams.

---

### Scope

- **In scope**:
  - SNMP V3 Trap Receiver Settings UI validation changes
  - Shared-platform persistence rules for tuple-qualified credential uniqueness
  - Save/edit behavior for entries with duplicate usernames but distinct supported protocol combinations
  - Password masking and encrypted-at-rest handling for newly saved or edited entries
  - Tuple-based runtime credential lookup for inbound SNMPv3 traps
  - Consistent behavior across Probe Server / RDG-based processing paths
  - Sanitized runtime diagnostics for unsupported or unmatched tuples
  - Upgrade validation, regression, performance, and operational readiness coverage

- **Out of scope**:
  - Device-side SNMP agent configuration changes on customer hardware
  - SNMPv1 or SNMPv2c trap handling
  - New cryptographic algorithms beyond the currently supported BLSS SNMPv3 options
  - Automatic migration of field-device credentials
  - Changes to customer password policy
  - New downstream alarm semantics unrelated to trap credential resolution

- **Assumptions**:
  1. ⚠️ ASSUMPTION / ⚠️ NEEDS VALIDATION: The supported authentication and privacy protocol matrix remains the current BLSS-supported SNMPv3 set; this Epic does not introduce new algorithms.
  2. ⚠️ ASSUMPTION / ⚠️ NEEDS VALIDATION: Uniqueness is enforced by username + authentication protocol + privacy protocol only. Password differences alone do not make two entries distinct.
  3. ⚠️ ASSUMPTION / ⚠️ NEEDS VALIDATION: Inbound trap metadata provides enough information for the runtime component to reliably distinguish the supported authentication/privacy tuple.
  4. ⚠️ ASSUMPTION / ⚠️ NEEDS VALIDATION: Existing persisted entries are keyed or can be migrated without customer-visible re-entry of credentials.
  5. ⚠️ ASSUMPTION / ⚠️ NEEDS VALIDATION: Probe Server and any RDG-based processing paths share equivalent credential resolution logic or can be brought into parity in the same release.
  6. ⚠️ ASSUMPTION / ⚠️ NEEDS VALIDATION: The configuration workflow is owned by a shared BLSS administration surface rather than a product-specific plugin.
  7. ⚠️ ASSUMPTION / ⚠️ NEEDS VALIDATION: Current release qualification already includes an SNMPv3 regression suite that can be extended rather than created from scratch.

---

### Dependencies & Risks

| Item | Type (Dep/Risk) | BLSS Product | Owner | Mitigation | Status |
|---|---|---|---|---|---|
| Existing UI validation rules for SNMP V3 Trap Receiver Settings | Dep | Shared platform services | UI team `[TBD]` | Review current client-side and server-side validation paths before implementation | In Evaluation |
| Persistence model and uniqueness constraint changes | Dep | Shared platform services | Platform team `[TBD]` | Define tuple-based uniqueness and add rollback-tested migration if schema/index changes are required | In Evaluation |
| Probe Server / RDG ownership clarity | Dep | Shared platform services | Architecture team `[TBD]` | Confirm all runtime components that participate in SNMPv3 trap processing | Open |
| Supported protocol matrix confirmation | Dep | Shared platform services | SNMP SME `[TBD]` | Lock supported combinations in validation and runtime test coverage before implementation | Open |
| Incorrect credential selection under shared usernames | Risk | Shared platform services | Development team `[TBD]` | Add deterministic positive/negative test matrix for each supported protocol tuple | Open |
| Password handling or logging regression during save, edit, or runtime failure flows | Risk | Shared platform services | Security team `[TBD]` | Reuse existing secret storage pattern and add regression tests for masked display and sanitized logging | Open |
| Regression in existing unique-username deployments | Risk | Shared platform services | QA team `[TBD]` | Run legacy baseline scenarios unchanged in release qualification | Open |
| Product ownership boundary and impacted downstream products | Risk | Shared platform / Cross-product | Product management `[TBD]` | Confirm owning backlog and impacted product list before PI commitment | Open |

---

### Operational deliverables (required for on-prem)

- [ ] Installation/upgrade runbook updated — describe any persistence/index change, runtime rollout order, restart expectations, and rollback steps
- [ ] Sizing guide updated — N/A unless architecture confirms measurable resource change or benchmark results show trap-processing impact
- [ ] Backup/restore procedure updated — confirm restored configurations preserve tuple-qualified entries and runtime compatibility
- [ ] Firewall/port documentation updated — N/A
- [ ] Release notes entry drafted — describe the removal of username-only uniqueness constraint and runtime trap-matching change
- [ ] Lifecycle Services training/enablement updated — explain when duplicate usernames are allowed, how protocol pairing is validated, and how to verify mixed-protocol deployments after upgrade

---

### Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| S1.1 | Allow duplicate usernames when the protocol tuple differs | 5 | `[TBD]` |
| S1.2 | Persist SNMPv3 trap credentials with tuple-qualified uniqueness | 8 | `[TBD]` |
| S1.3 | Enforce supported protocol combinations and duplicate rejection rules | 5 | `[TBD]` |
| S1.4 | Resolve inbound SNMPv3 traps using tuple-qualified credential lookup | 8 | `[TBD]` |
| S1.5 | Support mixed network-card and phased security migration scenarios | 5 | `[TBD]` |
| S1.6 | Add sanitized diagnostics for unmatched or unsupported SNMPv3 tuples | 5 | `[TBD]` |
| S1.7 | Preserve upgrade compatibility for trap receiver settings and runtime components | 5 | `[TBD]` |
| S1.8 | Qualify the release with regression, performance, and security coverage | 8 | `[TBD]` |
| S1.9 | Publish rollout guidance and services enablement for on-prem deployments | 3 | `[TBD]` |

---
---

## Stories for Epic 1

---

### STORY: S1.1 — Allow Duplicate Usernames When the Protocol Tuple Differs

**Parent Epic**: Epic 1 — Protocol-Qualified SNMPv3 Trap Receiver Configuration and Processing  
**BLSS Product**: Shared platform services

**Description**  
As a System Integrator / Partner,  
I want to create multiple SNMPv3 trap receiver entries that share the same username when the supported authentication and/or privacy protocol differs,  
So that I can deploy mixed device security profiles without renaming users on customer hardware.

**Context & notes**:
- The current issue specifically cites mixed network card deployments such as M2 and MS cards using different SNMPv3 security profiles.
- This story changes configuration behavior only; it does not alter device-side credentials.
- `[TBD — attach approved UI mockup or annotated screenshot for SNMP V3 Trap Receiver Settings]`
- On-prem considerations: saved configuration must remain local to the customer-managed BLSS environment and continue to be included in standard backup/restore.

**Acceptance Criteria**

AC1:
  Given: An existing SNMPv3 trap receiver entry uses username `trapuser` with one supported authentication/privacy protocol combination
  When: The administrator creates a second entry using username `trapuser` with a different supported authentication and/or privacy protocol combination
  Then: The system allows the new entry to be saved

AC2:
  Given: An existing SNMPv3 trap receiver entry uses username `trapuser` with a specific authentication/privacy protocol combination
  When: The administrator attempts to create or edit another entry to the same username and the same authentication/privacy protocol combination
  Then: The system blocks the save and indicates that the credential tuple already exists

AC3:
  Given: Multiple entries share the same username
  When: The administrator returns to the SNMP V3 Trap Receiver Settings view
  Then: Each entry remains distinguishable by its protocol metadata and the password remains masked

**Tasks**
- [ ] Task 1: <Frontend> Update SNMP V3 Trap Receiver Settings validation to allow duplicate usernames when the protocol tuple differs
- [ ] Task 2: <Frontend> Ensure saved entries display protocol metadata alongside the username without exposing secrets
- [ ] Task 3: <Test> Add UI regression coverage for create, edit, and reload flows with duplicate usernames
- [ ] Task 4: <Documentation> Update user-facing configuration guidance to explain tuple-based uniqueness

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (if UI-impacting)
- [ ] API contract defined (if integration) — N/A
- [ ] Device/protocol compatibility confirmed (if hardware-related)
- [ ] Cross-product dependencies identified and interface agreed
- [ ] On-prem install/upgrade impact assessed
- [ ] Sizing impact assessed (if resource requirements change)
- [ ] Dependencies identified and unblocked
- [ ] NFRs applicable to this story are noted
- [ ] Estimated by team

---

### STORY: S1.2 — Persist SNMPv3 Trap Credentials with Tuple-Qualified Uniqueness

**Parent Epic**: Epic 1 — Protocol-Qualified SNMPv3 Trap Receiver Configuration and Processing  
**BLSS Product**: Shared platform services

**Description**  
As an On-Prem IT / System Admin,  
I want BLSS to persist SNMPv3 trap receiver entries using tuple-qualified uniqueness and protected secret storage,  
So that saved configurations remain secure, restorable, and consistent with the updated validation behavior.

**Context & notes**:
- The issue requires credential uniqueness to be evaluated by username + authentication protocol + privacy protocol.
- Password values must remain encrypted at rest and must not be revealed after save.
- `[TBD — confirm whether configuration is stored in DB, file-based config, or both]`
- On-prem considerations: if schema or index changes are required, they must ship with upgrade and rollback guidance.

**Acceptance Criteria**

AC1:
  Given: A new SNMPv3 trap receiver entry is saved with a unique username + authentication protocol + privacy protocol tuple
  When: The configuration is persisted
  Then: The entry is stored successfully and the secret is encrypted at rest

AC2:
  Given: Two entries share the same username but use different supported protocol tuples
  When: The configuration is persisted and reloaded
  Then: Both entries remain available and unchanged after reload

AC3:
  Given: A persisted entry exists
  When: The system writes audit, validation, or operational logs during save or reload
  Then: The password is not written in plaintext to any log output

AC4:
  Given: A backup/restore cycle is executed for the BLSS application configuration
  When: The SNMPv3 trap receiver settings are restored
  Then: Tuple-qualified entries are restored without collapsing duplicate usernames into a single record

**Tasks**
- [ ] Task 1: <Data/schema> Update the persistence model to enforce uniqueness on username + authentication protocol + privacy protocol
- [ ] Task 2: <Security> Reuse or extend existing encryption-at-rest handling for SNMPv3 trap receiver secrets
- [ ] Task 3: <Upgrade/migration> Create forward and rollback scripts if a schema or index migration is required
- [ ] Task 4: <Test> Add persistence and backup/restore regression coverage for tuple-qualified entries

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (if UI-impacting) — N/A
- [ ] API contract defined (if integration) — N/A
- [ ] Device/protocol compatibility confirmed (if hardware-related)
- [ ] Cross-product dependencies identified and interface agreed
- [ ] On-prem install/upgrade impact assessed
- [ ] Sizing impact assessed (if resource requirements change)
- [ ] Dependencies identified and unblocked
- [ ] NFRs applicable to this story are noted
- [ ] Estimated by team

---

### STORY: S1.3 — Enforce Supported Protocol Combinations and Duplicate Rejection Rules

**Parent Epic**: Epic 1 — Protocol-Qualified SNMPv3 Trap Receiver Configuration and Processing  
**BLSS Product**: Shared platform services

**Description**  
As an Eaton Field Service Engineer,  
I want BLSS to clearly reject unsupported SNMPv3 protocol combinations and truly duplicated credential tuples,  
So that I can configure customers quickly without creating ambiguous runtime behavior.

**Context & notes**:
- The issue scope is limited to currently supported BLSS SNMPv3 authentication and privacy options.
- The story must not introduce new cryptographic algorithms or change password policy.
- `[TBD — confirm the exact supported protocol matrix with the SNMP/Probe Server SME]`
- On-prem considerations: validation must work identically in air-gapped customer environments without external lookup or cloud dependency.

**Acceptance Criteria**

AC1:
  Given: The administrator selects a currently supported BLSS authentication/privacy protocol combination
  When: The administrator saves the entry
  Then: Validation succeeds if the credential tuple is unique

AC2:
  Given: The administrator selects an unsupported or disallowed protocol combination
  When: The administrator attempts to save the entry
  Then: The system blocks the save and provides a validation error without exposing secrets

AC3:
  Given: The administrator changes only the password on an existing credential tuple
  When: The resulting entry would duplicate another existing username + authentication protocol + privacy protocol tuple
  Then: The system blocks the save because password value alone does not define uniqueness

**Tasks**
- [ ] Task 1: <Backend> Centralize tuple-based duplicate detection and supported-protocol validation rules
- [ ] Task 2: <Frontend> Align client-side validation with server-side validation outcomes
- [ ] Task 3: <Test> Add negative tests for unsupported protocol combinations and duplicate tuple attempts
- [ ] Task 4: <Documentation> Capture the supported protocol matrix and duplicate rules for support and services teams

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (if UI-impacting)
- [ ] API contract defined (if integration) — N/A
- [ ] Device/protocol compatibility confirmed (if hardware-related)
- [ ] Cross-product dependencies identified and interface agreed
- [ ] On-prem install/upgrade impact assessed
- [ ] Sizing impact assessed (if resource requirements change)
- [ ] Dependencies identified and unblocked
- [ ] NFRs applicable to this story are noted
- [ ] Estimated by team

---

### STORY: S1.4 — Resolve Inbound SNMPv3 Traps Using Tuple-Qualified Credential Lookup

**Parent Epic**: Epic 1 — Protocol-Qualified SNMPv3 Trap Receiver Configuration and Processing  
**BLSS Product**: Shared platform services

**Description**  
As an Operations / NOC Engineer,  
I want BLSS runtime processing to resolve inbound SNMPv3 traps using the full credential tuple,  
So that traps are processed against the correct configured security profile even when usernames are shared.

**Context & notes**:
- The source capability explicitly requires trap matching based on username + authentication protocol + privacy protocol.
- `[TBD — confirm the exact runtime data available to Probe Server / RDG for tuple resolution]`
- This story depends on the tuple-qualified configuration and persistence changes within this Epic.
- On-prem considerations: no internet dependency is introduced; behavior must remain deterministic in customer-isolated networks.

**Acceptance Criteria**

AC1:
  Given: Multiple configured SNMPv3 trap receiver entries share the same username and differ only by supported protocol tuple
  When: An inbound SNMPv3 trap arrives with one of those supported tuples
  Then: The runtime component resolves the matching credential entry and processes the trap using that entry

AC2:
  Given: A configured deployment uses unique usernames only
  When: An inbound SNMPv3 trap is received
  Then: Runtime behavior remains unchanged from the current supported baseline

AC3:
  Given: An inbound SNMPv3 trap does not match any configured supported tuple
  When: The runtime evaluates the trap
  Then: The trap is not mapped to an incorrect credential entry

**Tasks**
- [ ] Task 1: <Backend> Update Probe Server / RDG credential lookup logic to resolve by username + authentication protocol + privacy protocol
- [ ] Task 2: <Integration> Align the runtime lookup contract with the persisted configuration shape from S1.2
- [ ] Task 3: <Test> Add positive and negative integration tests for shared-username trap processing
- [ ] Task 4: <Performance> Benchmark tuple-qualified lookup against the current baseline

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (if UI-impacting) — N/A
- [ ] API contract defined (if integration)
- [ ] Device/protocol compatibility confirmed (if hardware-related)
- [ ] Cross-product dependencies identified and interface agreed
- [ ] On-prem install/upgrade impact assessed
- [ ] Sizing impact assessed (if resource requirements change)
- [ ] Dependencies identified and unblocked
- [ ] NFRs applicable to this story are noted
- [ ] Estimated by team

---

### STORY: S1.5 — Support Mixed Network-Card and Phased Security Migration Scenarios

**Parent Epic**: Epic 1 — Protocol-Qualified SNMPv3 Trap Receiver Configuration and Processing  
**BLSS Product**: Shared platform services

**Description**  
As a System Integrator / Partner,  
I want BLSS to handle mixed SNMPv3 security profiles that share a username across different device populations,  
So that I can onboard customers with heterogeneous hardware or staged security upgrades without interrupting trap ingestion.

**Context & notes**:
- The Jira issue highlights mixed network card types such as M2, MS, and staged MD5/DES to SHA/AES migrations.
- This story converts those examples into release-qualification scenarios.
- `[TBD — confirm the full customer device matrix that must be represented in test coverage]`
- On-prem considerations: scenario validation should be executable in customer-like lab environments without relying on cloud-hosted services.

**Acceptance Criteria**

AC1:
  Given: A deployment contains two device populations that share a username but use different supported SNMPv3 protocol tuples
  When: Both populations send traps to BLSS
  Then: Traps from both populations are processed successfully in parallel

AC2:
  Given: A customer is migrating devices from one supported SNMPv3 protocol tuple to another while reusing the same username
  When: Old and new profiles are active simultaneously
  Then: BLSS continues to process both profiles without forcing device-side username changes

AC3:
  Given: The release qualification environment exercises the agreed mixed-device matrix
  When: Results are reviewed
  Then: Evidence exists for each required scenario and any uncovered combinations are listed as gaps with owner and follow-up

**Tasks**
- [ ] Task 1: <Test> Define scenario matrix for mixed network-card and phased migration deployments
- [ ] Task 2: <Integration> Build or update lab configurations that represent the agreed supported device profiles
- [ ] Task 3: <Test> Execute end-to-end trap ingestion validation for each scenario
- [ ] Task 4: <Documentation> Capture supported scenario guidance and known limitations for deployment teams

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (if UI-impacting) — N/A
- [ ] API contract defined (if integration)
- [ ] Device/protocol compatibility confirmed (if hardware-related)
- [ ] Cross-product dependencies identified and interface agreed
- [ ] On-prem install/upgrade impact assessed
- [ ] Sizing impact assessed (if resource requirements change)
- [ ] Dependencies identified and unblocked
- [ ] NFRs applicable to this story are noted
- [ ] Estimated by team

---

### STORY: S1.6 — Add Sanitized Diagnostics for Unmatched or Unsupported SNMPv3 Tuples

**Parent Epic**: Epic 1 — Protocol-Qualified SNMPv3 Trap Receiver Configuration and Processing  
**BLSS Product**: Shared platform services

**Description**  
As an Eaton Support Team member,  
I want BLSS to emit clear but sanitized diagnostics when an inbound SNMPv3 trap cannot be matched to a supported configured tuple,  
So that I can troubleshoot customer issues without exposing credentials.

**Context & notes**:
- The runtime must fail safely when tuple resolution does not find a supported match.
- Diagnostics must help support teams distinguish unsupported protocol usage from missing configuration.
- `[TBD — confirm existing logging/audit channels and retention policy for SNMP runtime events]`
- On-prem considerations: diagnostics must remain available in customer-managed environments and comply with local logging policies.

**Acceptance Criteria**

AC1:
  Given: An inbound SNMPv3 trap uses a tuple that is not configured in BLSS
  When: Runtime evaluation fails to find a match
  Then: A sanitized diagnostic event is recorded without logging plaintext credentials or passwords

AC2:
  Given: An inbound trap uses an unsupported or disallowed protocol combination
  When: Runtime evaluation occurs
  Then: BLSS records a sanitized failure reason that distinguishes unsupported protocol use from missing configuration

AC3:
  Given: Support personnel review the diagnostic output
  When: They troubleshoot the issue
  Then: The output contains enough context to identify the configuration gap without revealing secrets

**Tasks**
- [ ] Task 1: <Backend> Define sanitized runtime diagnostic messages for unmatched and unsupported tuple outcomes
- [ ] Task 2: <Security> Review diagnostics and logs for secret exposure risks
- [ ] Task 3: <Test> Add negative-path tests confirming fail-closed behavior and sanitized logging
- [ ] Task 4: <Documentation> Update support troubleshooting guidance for shared-username SNMPv3 deployments

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (if UI-impacting) — N/A
- [ ] API contract defined (if integration)
- [ ] Device/protocol compatibility confirmed (if hardware-related)
- [ ] Cross-product dependencies identified and interface agreed
- [ ] On-prem install/upgrade impact assessed
- [ ] Sizing impact assessed (if resource requirements change)
- [ ] Dependencies identified and unblocked
- [ ] NFRs applicable to this story are noted
- [ ] Estimated by team

---

### STORY: S1.7 — Preserve Upgrade Compatibility for Trap Receiver Settings and Runtime Components

**Parent Epic**: Epic 1 — Protocol-Qualified SNMPv3 Trap Receiver Configuration and Processing  
**BLSS Product**: Shared platform services

**Description**  
As an On-Prem IT / System Admin,  
I want existing SNMPv3 trap receiver settings and dependent runtime components to remain valid after upgrade,  
So that this capability can be adopted without service disruption, forced credential re-entry, or trap-processing regression.

**Context & notes**:
- The source capability explicitly excludes automatic migration of customer device credentials.
- This story covers application-side upgrade safety for both persistence and runtime rollout.
- `[TBD — confirm supported source versions, runtime restart expectations, and rollback behavior]`
- On-prem considerations: upgrade ownership sits with the customer or Lifecycle Management Services, so runbooks and rollback steps must be explicit.

**Acceptance Criteria**

AC1:
  Given: A supported pre-upgrade BLSS environment has existing SNMPv3 trap receiver settings with unique usernames
  When: The environment is upgraded to the release containing this capability
  Then: All existing entries remain present and editable without manual re-entry

AC2:
  Given: The upgrade changes persistence rules and deploys updated runtime components
  When: The upgrade is executed following the runbook
  Then: The system completes without invalidating existing configurations or introducing trap-processing regression

AC3:
  Given: An upgrade rollback is required
  When: The rollback procedure is executed
  Then: The pre-upgrade configuration and runtime state are restorable per the approved rollback process

**Tasks**
- [ ] Task 1: <Upgrade/migration> Define the supported upgrade path and any required migration step for tuple-qualified uniqueness and runtime rollout
- [ ] Task 2: <Test> Execute upgrade and rollback validation on a representative supported environment
- [ ] Task 3: <Documentation> Update upgrade runbook and release notes with operator guidance
- [ ] Task 4: <Service enablement> Update Lifecycle Services deployment guidance for customer upgrades

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (if UI-impacting) — N/A
- [ ] API contract defined (if integration) — N/A
- [ ] Device/protocol compatibility confirmed (if hardware-related)
- [ ] Cross-product dependencies identified and interface agreed
- [ ] On-prem install/upgrade impact assessed
- [ ] Sizing impact assessed (if resource requirements change)
- [ ] Dependencies identified and unblocked
- [ ] NFRs applicable to this story are noted
- [ ] Estimated by team

---

### STORY: S1.8 — Qualify the Release with Regression, Performance, and Security Coverage

**Parent Epic**: Epic 1 — Protocol-Qualified SNMPv3 Trap Receiver Configuration and Processing  
**BLSS Product**: Shared platform services

**Description**  
As a Product Owner,  
I want the release qualified with explicit regression, performance, and security evidence,  
So that the capability can move from evaluation to delivery with defensible on-premises release readiness.

**Context & notes**:
- The Jira issue calls out performance, reliability, and security expectations but does not define a release gate.
- This story converts those expectations into measurable qualification work.
- `[TBD — confirm benchmark data set, trap volume baseline, and release gate criteria with QA/architecture]`
- On-prem considerations: qualification should reflect supported customer architectures and customer-managed upgrade practices.

**Acceptance Criteria**

AC1:
  Given: The updated configuration and runtime logic are available in a testable build
  When: The agreed regression suite is executed
  Then: Existing unique-username scenarios and new shared-username scenarios both pass

AC2:
  Given: Performance testing is executed against the agreed benchmark environment
  When: Results are compared to the current baseline
  Then: Credential-resolution overhead remains within the approved threshold

AC3:
  Given: Security validation reviews the configuration and runtime change
  When: Logs, persisted secrets, and troubleshooting flows are assessed
  Then: No plaintext credential exposure is found and any findings are documented before release approval

**Tasks**
- [ ] Task 1: <Test> Extend automated regression coverage for configuration, runtime lookup, and negative scenarios
- [ ] Task 2: <Performance> Execute benchmark comparison against the current production baseline
- [ ] Task 3: <Security> Review secret handling and logging behavior in positive and negative paths
- [ ] Task 4: <Documentation> Publish release qualification evidence and open issues register

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (if UI-impacting) — N/A
- [ ] API contract defined (if integration)
- [ ] Device/protocol compatibility confirmed (if hardware-related)
- [ ] Cross-product dependencies identified and interface agreed
- [ ] On-prem install/upgrade impact assessed
- [ ] Sizing impact assessed (if resource requirements change)
- [ ] Dependencies identified and unblocked
- [ ] NFRs applicable to this story are noted
- [ ] Estimated by team

---

### STORY: S1.9 — Publish Rollout Guidance and Services Enablement for On-Prem Deployments

**Parent Epic**: Epic 1 — Protocol-Qualified SNMPv3 Trap Receiver Configuration and Processing  
**BLSS Product**: Shared platform services

**Description**  
As an Eaton Field Service Engineer,  
I want concise rollout and validation guidance for this capability,  
So that customer upgrades and post-upgrade verification can be executed consistently in on-premises environments.

**Context & notes**:
- Customer or Lifecycle Management Services teams own upgrade execution in BLSS on-premises deployments.
- This story turns the operational deliverables into explicit release assets.
- `[TBD — confirm the final rollout owner, release train timing, and support handoff process]`
- On-prem considerations: guidance must include upgrade sequence, rollback trigger points, and post-upgrade validation steps in isolated customer networks.

**Acceptance Criteria**

AC1:
  Given: The capability is approved for release
  When: Deployment guidance is prepared
  Then: The runbook describes upgrade order, service restart expectations, rollback steps, and post-upgrade validation for shared-username SNMPv3 scenarios

AC2:
  Given: Lifecycle Services or support teams review the enablement material
  When: They perform the documented validation steps in a representative environment
  Then: They can verify shared-username SNMPv3 configuration and runtime behavior without additional tribal knowledge

AC3:
  Given: Release notes are drafted
  When: Customers or internal stakeholders read them
  Then: The notes clearly explain the new tuple-based uniqueness behavior and any confirmed limitations

**Tasks**
- [ ] Task 1: <Documentation> Draft upgrade runbook and post-upgrade validation checklist
- [ ] Task 2: <Documentation> Draft release notes and known-limitations section
- [ ] Task 3: <Service enablement> Prepare support and Lifecycle Services enablement material
- [ ] Task 4: <Review> Obtain sign-off from product, support, and architecture stakeholders `[TBD — confirm approvers]`

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (if UI-impacting) — N/A
- [ ] API contract defined (if integration) — N/A
- [ ] Device/protocol compatibility confirmed (if hardware-related)
- [ ] Cross-product dependencies identified and interface agreed
- [ ] On-prem install/upgrade impact assessed
- [ ] Sizing impact assessed (if resource requirements change)
- [ ] Dependencies identified and unblocked
- [ ] NFRs applicable to this story are noted
- [ ] Estimated by team
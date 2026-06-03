# BDCSPM-73832 — Licensing Error Display with Descriptions

## Parent Capability

- **Capability**: BDCSPM-73832 — BLSS has the capability to display licensing errors with descriptions for customers
- **BAT Epic (separate)**: BDCSPM-74046 — Business Acceptance Testing (not covered here)

---

## EPIC: [NEW] Licensing Error Contextual Display & Remediation Guidance

**BLSS Product(s)**: Cross-product / Shared Platform Services (BLSS UI Layer)

**Summary**
Enhance the BLSS on-premises UI to present actionable, human-readable licensing error messages — including root-cause identification and step-by-step remediation guidance — so that customers can self-diagnose and resolve licensing issues without contacting support.

---

### Description

- **Problem statement**: Customers regularly receive vague licensing alerts (HC error codes) via system messages with no actionable meaning. This generates significant support ticket volume, particularly around vCenter credential rotation scenarios where customers forget to update Brightlayer's stored credentials. Users cannot understand, diagnose, or communicate the issue effectively.

- **Proposed solution**: Implement an error-to-description mapping service that translates HC error codes into structured error messages containing: (1) what happened, (2) why it happened, and (3) what to do next. Surface these messages in both toast notifications and the system message log with inline remediation guidance.

- **User personas**:
  - Data Center Operators / Admin Users (primary — resolving issues directly)
  - On-Prem IT / System Admin (vCenter owners who rotate passwords)
  - Eaton Field Service Engineer (remote troubleshooting)

- **Business value**:
  - Reduces Brightlayer Support ticket volume related to licensing confusion
  - Improves customer time-to-resolution from hours/days to minutes
  - Decreases operational cost associated with recurring license issues
  - Aligns with Eaton's self-service and digital-first support strategy

- **Key workflows**:
  1. License validation fails → system detects HC error code → mapping service resolves to description → UI renders contextual error with remediation
  2. vCenter credential becomes invalid → license validation fails → specific credential-related guidance displayed
  3. Grace period activates → user sees warning with time remaining and required corrective action
  4. User reviews historical licensing errors in system message log for audit/troubleshooting

- **Cross-product touchpoints**: This capability operates at the shared platform UI layer. All BLSS products that surface licensing errors (DCPM, EPMS, Distributed IT, Brightlayer Power) will benefit. No changes required within individual product modules — the licensing error display is handled by the shared system messaging framework.

- **UX/UI references**: `[TBD — UX mockups to be created as part of this Epic]`
  - Must follow Brightlayer Design System guidelines
  - Error messages in toast notifications AND system message window/log
  - Structured format: What happened | Why | What to do next

- **On-prem operational impact**:
  - Install/upgrade impact: New error code mapping configuration (JSON/DB-based); schema addition for error catalog if DB-backed
  - Sizing impact: Minimal — error catalog is small; logging volume increase negligible
  - Network impact: None — all processing is local, no external calls
  - Backup impact: Error catalog data should be included in standard backup; no schema migration concerns beyond initial deployment

---

### Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Performance | Error message rendering from detection to display | < 2 seconds | P0 |
| Reliability | Error events persisted for auditability | 100% of licensing error events logged | P0 |
| Usability | Messages displayed in both toast and system message log | Both channels simultaneously | P0 |
| Internationalization | Message structure supports translation (externalized strings) | All user-facing strings externalized | P1 |
| Accessibility | Error messages accessible via screen readers | WCAG 2.1 AA compliance | P1 |
| Maintainability | Error code catalog extensible without code deployment | Configuration-based addition of new codes | P1 |
| Compatibility | Browser support | Chrome v120+, Edge v120+, Firefox v115+, Safari v17+ | P0 |
| Data Retention | Licensing error events retained in system message log | Minimum 90 days `[TBD — confirm with ops]` | P2 |

---

### Acceptance Criteria (Epic-level)

- [ ] AC1: When a licensing validation failure occurs, the system displays a human-readable error message containing: what happened, why it happened, and recommended corrective action — within 2 seconds of detection.
- [ ] AC2: When vCenter credentials are invalid, the error message specifically identifies credential failure as root cause and instructs user to update credentials in Brightlayer configuration.
- [ ] AC3: When a grace period is activated, the system displays a warning with time remaining and required corrective action.
- [ ] AC4: All licensing error messages are visible in both toast notifications and the system message history/log.
- [ ] AC5: All licensing error events are persisted in the system log for auditability.
- [ ] AC6: Error code descriptions can be added/updated via configuration without requiring a code deployment.
- [ ] AC7: All user-facing error strings are externalized and ready for internationalization.
- [ ] AC8: No backend SSH access is required by customers to view or act on error information.

---

### Scope

- **In scope**:
  - HC error code to human-readable description mapping service
  - Structured error messages (what / why / what to do) for licensing errors
  - Toast notification display of licensing errors with remediation
  - System message log display of licensing errors with remediation
  - vCenter credential failure detection and specific guidance
  - Grace period activation warning with time remaining
  - Historical error viewing in system message log
  - Internationalization-ready string externalization
  - Error catalog extensibility via configuration

- **Out of scope**:
  - Automatic remediation (e.g., password reset, credential sync)
  - Backend SSH-based configuration or fixes
  - Changes to licensing model or validation algorithm
  - Integration with external identity/password management systems
  - Actual translations into specific languages (only structural readiness)
  - Mobile-native app support (web browser only)

- **Assumptions**:
  1. ⚠️ ASSUMPTION: The existing system message framework (toast + log) can be extended without major architectural changes.
  2. ⚠️ ASSUMPTION: A centralized HC error code catalog does NOT currently exist in structured format and must be created as part of this effort.
  3. ⚠️ ASSUMPTION: Grace period time remaining IS available from the licensing engine API.
  4. ⚠️ ASSUMPTION: The licensing validation logic can emit structured error codes (not just generic failures) that enable root-cause differentiation.
  5. ⚠️ ASSUMPTION: This operates at the shared BLSS UI layer; no individual product module changes required.
  6. ⚠️ ASSUMPTION: Target release is BLSS v8.x (next release after v8.0.0).

---

### Dependencies & Risks

| Item | Type | BLSS Product | Owner | Mitigation | Status |
|---|---|---|---|---|---|
| Existing system message framework | Dep | Shared Platform | `[TBD]` | Assess extensibility early in sprint 1 | Not started |
| HC error code catalog (centralized) | Dep | Shared Platform | `[TBD — licensing team]` | Create catalog as Story S1; engage licensing SME | Not started |
| License validation logic emits structured codes | Dep | Shared Platform | `[TBD — licensing team]` | Validate current error output format in spike | Not started |
| Grace period remaining time from licensing engine | Dep | Shared Platform | `[TBD — licensing team]` | Confirm API exposes this data; fallback: omit countdown | Not started |
| vCenter credential validation differentiation | Dep | Shared Platform | `[TBD — licensing team]` | Confirm licensing can distinguish credential failure vs. other failures | Not started |
| UX mockups (Brightlayer Design System) | Dep | UX Team | `[TBD]` | Design sprint parallel to backend catalog work | Not started |
| Incomplete HC error code mapping | Risk | — | — | Prioritize top 10 ticket-generating codes; iterate | — |
| Root cause misdiagnosis (simplistic detection) | Risk | — | — | Include confidence indicator; fallback to generic message | — |
| UI clutter from verbose error messages | Risk | — | — | UX review; progressive disclosure (summary + expand) | — |

---

### Operational Deliverables (required for on-prem)

- [ ] Installation/upgrade runbook updated (new error catalog deployment)
- [ ] Sizing guide updated (minimal impact — note only)
- [ ] Backup/restore procedure updated (error catalog included in backup scope)
- [ ] Firewall/port documentation updated (N/A — no new network requirements)
- [ ] Release notes entry drafted
- [ ] Lifecycle Services training/enablement updated (error message interpretation guide)

---

### Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| S1 | Define and catalog HC error codes with human-readable descriptions | 5 | Sprint 1 |
| S2 | Implement error-to-description mapping service | 8 | Sprint 1-2 |
| S3 | Display detailed licensing error in toast notification | 5 | Sprint 2 |
| S4 | Display detailed licensing error in system message log | 5 | Sprint 2 |
| S5 | Display vCenter credential failure with specific remediation guidance | 5 | Sprint 3 |
| S6 | Display grace period warning with time remaining and corrective actions | 5 | Sprint 3 |
| S7 | Externalize error message strings for internationalization readiness | 3 | Sprint 3 |
| S8 | Persist all licensing error events for audit trail | 3 | Sprint 2 |
| S9 | Update documentation and release notes | 2 | Sprint 4 |

---

---

## Story Details

---

### STORY: S1 — Define and Catalog HC Error Codes with Human-Readable Descriptions

**Parent Epic**: Licensing Error Contextual Display & Remediation Guidance
**BLSS Product**: Shared Platform Services

**Description**
As an On-Prem IT / System Admin,
I want a comprehensive catalog of HC licensing error codes with human-readable descriptions and remediation steps,
So that the system can present meaningful guidance when errors occur.

**Context & notes**:
- This is a foundational story — all subsequent stories depend on this catalog.
- Must engage licensing SME to identify all current HC error codes and their meanings.
- Catalog format should be configuration-based (JSON or DB table) to allow extension without code changes.
- Each entry must contain: error code, severity, short description, detailed explanation, root cause category, and remediation steps.
- ⚠️ NEEDS VALIDATION: Confirm with licensing team whether all HC codes are documented somewhere today.

**Acceptance Criteria**

AC1:
  Given: The licensing team has been consulted
  When: The error catalog is reviewed
  Then: It contains entries for all known HC error codes with: code, severity, short description, detailed explanation, root cause category, and remediation steps.

AC2:
  Given: A new HC error code needs to be added in the future
  When: An administrator adds a new entry to the catalog configuration
  Then: The new code is recognized by the mapping service without a code deployment.

AC3:
  Given: The error catalog is populated
  When: It is validated against historical support tickets
  Then: The top 10 most frequent licensing-related ticket drivers have catalog entries.

**Tasks**
- [ ] Task 1: (Research) Engage licensing SME; collect all existing HC error codes and their meanings
- [ ] Task 2: (Data/schema) Design catalog schema (fields: code, severity, short_desc, detail, root_cause_category, remediation_steps, created_date, updated_date)
- [ ] Task 3: (Data/schema) Create DB migration script for error catalog table OR define JSON configuration file format
- [ ] Task 4: (Data) Populate catalog with all known HC error codes
- [ ] Task 5: (Validation) Cross-reference catalog against top support ticket categories to ensure coverage
- [ ] Task 6: (Documentation) Document catalog schema and extension procedure for future additions
- [ ] Task 7: (Installer/config) Update installer to deploy initial error catalog

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Licensing SME identified and available for consultation
- [ ] Access to historical support ticket data for validation
- [ ] Catalog format decision made (DB vs. JSON config)
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: S2 — Implement Error-to-Description Mapping Service

**Parent Epic**: Licensing Error Contextual Display & Remediation Guidance
**BLSS Product**: Shared Platform Services

**Description**
As the BLSS system,
I want a backend service that maps HC error codes to their catalog entries and returns structured error descriptions,
So that the UI layer can render contextual licensing error messages.

**Context & notes**:
- Depends on S1 (error catalog must exist).
- Service should accept an error code and return structured response: { code, severity, shortDescription, detailedExplanation, rootCauseCategory, remediationSteps[] }.
- Must handle unknown codes gracefully (return generic "contact support" message).
- Must be performant (< 100ms lookup time) to meet the 2-second end-to-end NFR.
- Consider caching catalog in memory for performance.

**Acceptance Criteria**

AC1:
  Given: A known HC error code is received by the mapping service
  When: The service processes the request
  Then: It returns the complete structured description (short description, detailed explanation, root cause, remediation steps) within 100ms.

AC2:
  Given: An unknown/unmapped error code is received
  When: The service processes the request
  Then: It returns a generic fallback message ("An unrecognized licensing error occurred. Please contact Brightlayer Support with error code: {code}") and logs a warning.

AC3:
  Given: The error catalog has been updated with a new entry
  When: The mapping service receives that new code
  Then: It returns the new entry without requiring a service restart (or within a configurable refresh interval).

**Tasks**
- [ ] Task 1: (Backend) Design and implement mapping service API (internal, not public-facing)
- [ ] Task 2: (Backend) Implement catalog lookup with in-memory caching and configurable refresh interval
- [ ] Task 3: (Backend) Implement fallback/unknown code handling with logging
- [ ] Task 4: (Test) Write unit tests for mapping service (known codes, unknown codes, cache refresh)
- [ ] Task 5: (Test) Write integration test verifying end-to-end lookup from error catalog

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] S1 (error catalog) is complete or in parallel with agreed schema
- [ ] API contract defined (internal service interface)
- [ ] Performance target confirmed (< 100ms)
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: S3 — Display Detailed Licensing Error in Toast Notification

**Parent Epic**: Licensing Error Contextual Display & Remediation Guidance
**BLSS Product**: Shared Platform Services

**Description**
As a Data Center Operator,
I want to see a clear, actionable toast notification when a licensing error occurs,
So that I immediately understand what happened and what to do without navigating away from my current workflow.

**Context & notes**:
- Depends on S2 (mapping service).
- Toast must show: severity icon, short description, and a "View Details" action to expand or navigate to system message log.
- Must not block user workflow (auto-dismiss after configurable timeout, default 10s `[TBD]`).
- Must follow Brightlayer Design System toast/snackbar component guidelines.
- Progressive disclosure: toast shows summary; full details available on click/expand.
- ⚠️ NEEDS VALIDATION: Confirm existing toast notification component supports structured content (icon + text + action button).

**Acceptance Criteria**

AC1:
  Given: A licensing validation failure occurs
  When: The system detects the HC error code
  Then: A toast notification appears within 2 seconds showing: severity icon, short error description, and a "View Details" action.

AC2:
  Given: A licensing error toast is displayed
  When: The user clicks "View Details"
  Then: The user is navigated to the system message log entry showing the full detailed explanation and remediation steps.

AC3:
  Given: A licensing error toast is displayed
  When: The user takes no action
  Then: The toast auto-dismisses after the configured timeout (default 10 seconds) without blocking user interaction.

AC4 (negative):
  Given: Multiple licensing errors occur in rapid succession (< 5 seconds apart)
  When: The system would display multiple toasts
  Then: Toasts are queued (not stacked/overlapping) and displayed sequentially, with a maximum of 3 pending toasts before consolidating into a summary ("3 licensing errors — view system messages").

**Tasks**
- [ ] Task 1: (Frontend) Extend existing toast notification component to support structured licensing error content
- [ ] Task 2: (Frontend) Implement "View Details" navigation action to system message log
- [ ] Task 3: (Frontend) Implement toast queuing/consolidation logic for rapid-fire errors
- [ ] Task 4: (Frontend) Apply Brightlayer Design System styling (severity icon, typography, action button)
- [ ] Task 5: (Test) Write UI tests for toast display, auto-dismiss, and navigation
- [ ] Task 6: (Test) Write test for rapid-fire toast consolidation behavior

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached for toast notification layout
- [ ] Brightlayer Design System toast component reviewed for extensibility
- [ ] S2 (mapping service) interface contract agreed
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: S4 — Display Detailed Licensing Error in System Message Log

**Parent Epic**: Licensing Error Contextual Display & Remediation Guidance
**BLSS Product**: Shared Platform Services

**Description**
As a Data Center Operator,
I want to view detailed licensing error information in the system message log,
So that I can review historical errors, understand root causes, and follow remediation steps at my own pace.

**Context & notes**:
- Depends on S2 (mapping service).
- System message log entry must show: timestamp, severity, error code, short description, detailed explanation, root cause, and remediation steps.
- Must integrate with existing system message window/log framework.
- Remediation steps should be rendered as an ordered list for clarity.
- Users must be able to filter system messages by "Licensing" category.

**Acceptance Criteria**

AC1:
  Given: A licensing validation failure has occurred
  When: The user navigates to the system message log
  Then: The licensing error entry displays: timestamp, severity indicator, error code, short description, detailed explanation, root cause category, and numbered remediation steps.

AC2:
  Given: Multiple licensing errors have occurred over time
  When: The user filters system messages by "Licensing" category
  Then: Only licensing-related messages are displayed, in reverse chronological order.

AC3:
  Given: A licensing error is displayed in the system message log
  When: The user reads the remediation steps
  Then: Each step is actionable and self-contained (no requirement to consult external documentation for resolution).

**Tasks**
- [ ] Task 1: (Frontend) Extend system message log entry component to render structured licensing error format
- [ ] Task 2: (Frontend) Implement "Licensing" category filter in system message log
- [ ] Task 3: (Frontend) Render remediation steps as ordered list with Brightlayer Design System styling
- [ ] Task 4: (Backend) Ensure mapping service output is stored in system message log with full structure
- [ ] Task 5: (Test) Write UI tests for message log rendering and filtering
- [ ] Task 6: (Test) Verify historical messages display correctly after upgrade (backward compatibility)

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached for system message log entry layout
- [ ] Existing system message log framework reviewed for extensibility
- [ ] S2 (mapping service) interface contract agreed
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: S5 — Display vCenter Credential Failure with Specific Remediation Guidance

**Parent Epic**: Licensing Error Contextual Display & Remediation Guidance
**BLSS Product**: Shared Platform Services

**Description**
As an On-Prem IT / System Admin,
I want to see a specific error message when licensing fails due to invalid vCenter credentials,
So that I immediately know to update my vCenter credentials in Brightlayer configuration instead of contacting support.

**Context & notes**:
- Depends on S2 (mapping service) and S3/S4 (display channels).
- This is the #1 support ticket driver per the capability description.
- The licensing engine must be able to differentiate "credential failure" from other HC errors.
- ⚠️ NEEDS VALIDATION: Confirm the licensing validation logic can distinguish between vCenter credential failure, vCenter connectivity failure, and other license validation failures.
- Message must explicitly state: "License validation failed due to invalid vCenter credentials. Update credentials in [specific location in Brightlayer configuration]."
- Consider deep-linking to the credential configuration page if technically feasible.

**Acceptance Criteria**

AC1:
  Given: The BLSS system attempts license validation and vCenter credentials are invalid (authentication rejected)
  When: The error is processed
  Then: The system displays: "License validation failed due to invalid vCenter credentials" with remediation: "Update vCenter credentials in [Settings > vCenter Configuration]" `[TBD — confirm exact navigation path]`.

AC2:
  Given: The vCenter server is unreachable (network/connectivity issue, NOT credential failure)
  When: The error is processed
  Then: The system displays a DIFFERENT message: "Unable to reach vCenter server for license validation" with appropriate network troubleshooting guidance (not credential update guidance).

AC3:
  Given: A vCenter credential failure message is displayed
  When: The user follows the remediation steps
  Then: The steps guide the user to the exact configuration location where credentials can be updated.

**Tasks**
- [ ] Task 1: (Backend) Identify and validate how licensing engine distinguishes credential failure vs. connectivity failure vs. other failures
- [ ] Task 2: (Backend) Ensure mapping service handles vCenter-specific error codes with tailored descriptions
- [ ] Task 3: (Frontend) Implement deep-link or navigation hint to credential configuration page in remediation steps
- [ ] Task 4: (Data) Add vCenter-specific entries to error catalog (credential failure, connectivity failure)
- [ ] Task 5: (Test) Write integration test simulating vCenter credential failure → correct message displayed
- [ ] Task 6: (Test) Write integration test simulating vCenter connectivity failure → different message displayed

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Licensing team confirms error code differentiation for vCenter scenarios
- [ ] Exact UI navigation path to credential configuration confirmed
- [ ] S2-S4 dependencies in progress or complete
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: S6 — Display Grace Period Warning with Time Remaining and Corrective Actions

**Parent Epic**: Licensing Error Contextual Display & Remediation Guidance
**BLSS Product**: Shared Platform Services

**Description**
As a Data Center Operator,
I want to see a clear warning when the system enters a licensing grace period, including how much time remains and what I must do,
So that I can resolve the licensing issue before the grace period expires and service is interrupted.

**Context & notes**:
- Depends on S2 (mapping service) and S3/S4 (display channels).
- Grace period behavior already exists in system messaging (per BLSS v8.0.0 documentation); this story enhances it with structured guidance.
- ⚠️ NEEDS VALIDATION: Confirm that the licensing engine API exposes grace period remaining time.
- Warning severity should escalate as time remaining decreases (e.g., Info → Warning → Critical at < 24 hours).
- Consider recurring reminder notifications at configurable intervals during grace period.

**Acceptance Criteria**

AC1:
  Given: The system license becomes invalid and the grace period is activated
  When: The grace period begins
  Then: The system displays a warning message explaining: the license is invalid, the grace period is active, time remaining (days/hours), and the specific corrective action required.

AC2:
  Given: The grace period is active and time remaining drops below 24 hours
  When: The threshold is crossed
  Then: The warning severity escalates to Critical, and a new toast notification is triggered to alert active users.

AC3:
  Given: The grace period warning is displayed
  When: The user views the message
  Then: The corrective action section identifies the specific license error that triggered the grace period (with link to the relevant system message log entry).

AC4 (negative):
  Given: The grace period expires
  When: The license is no longer valid
  Then: The system displays a critical error indicating service limitations and directs user to contact Brightlayer Support with specific error reference.

**Tasks**
- [ ] Task 1: (Backend) Validate licensing engine API exposes grace period start time, duration, and remaining time
- [ ] Task 2: (Backend) Implement grace period status monitoring and severity escalation logic
- [ ] Task 3: (Frontend) Implement grace period warning display with countdown/time remaining
- [ ] Task 4: (Frontend) Implement severity escalation visual treatment (Info → Warning → Critical)
- [ ] Task 5: (Frontend) Implement recurring reminder notification logic during grace period
- [ ] Task 6: (Data) Add grace period entries to error catalog (activation, escalation, expiration)
- [ ] Task 7: (Test) Write tests for grace period activation, escalation at threshold, and expiration scenarios

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Licensing team confirms grace period time remaining is available via API
- [ ] Severity escalation thresholds agreed (24h = Critical; configurable)
- [ ] S2-S4 dependencies in progress or complete
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: S7 — Externalize Error Message Strings for Internationalization Readiness

**Parent Epic**: Licensing Error Contextual Display & Remediation Guidance
**BLSS Product**: Shared Platform Services

**Description**
As a Product Owner,
I want all user-facing licensing error message strings externalized into a translatable resource format,
So that future localization efforts can translate messages without code changes.

**Context & notes**:
- Depends on S3-S6 (all UI-facing stories must use externalized strings).
- This story ensures structural readiness; actual translations are OUT OF SCOPE.
- Use existing BLSS i18n framework/pattern (resource bundles, JSON locale files, etc.).
- ⚠️ NEEDS VALIDATION: Confirm current BLSS i18n approach (resource bundle format, tooling).
- Error catalog descriptions (from S1) should reference message keys, not hardcoded strings.

**Acceptance Criteria**

AC1:
  Given: All licensing error messages are implemented (S3-S6)
  When: The codebase is reviewed
  Then: Zero hardcoded user-facing strings exist; all text references externalized message keys.

AC2:
  Given: A new locale file is added (e.g., fr-FR.json)
  When: The application loads with that locale configured
  Then: Licensing error messages render in the new locale (assuming translations are provided).

AC3:
  Given: The error catalog contains remediation steps
  When: Steps are rendered in the UI
  Then: Step text is sourced from locale-aware resources, supporting parameterized values (e.g., "{settingsPath}").

**Tasks**
- [ ] Task 1: (Frontend) Audit all licensing error UI strings and extract to resource files
- [ ] Task 2: (Frontend) Implement parameterized message templates (e.g., "Update credentials in {path}")
- [ ] Task 3: (Backend) Ensure error catalog supports locale-keyed descriptions or message key references
- [ ] Task 4: (Test) Verify no hardcoded strings via static analysis / lint rule
- [ ] Task 5: (Test) Verify rendering with a test locale to confirm i18n pipeline works end-to-end

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Current BLSS i18n framework/pattern confirmed
- [ ] S3-S6 in progress (string externalization done incrementally)
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: S8 — Persist All Licensing Error Events for Audit Trail

**Parent Epic**: Licensing Error Contextual Display & Remediation Guidance
**BLSS Product**: Shared Platform Services

**Description**
As an On-Prem IT / System Admin,
I want all licensing error events persisted in the system log with full detail,
So that I can audit historical licensing issues and provide accurate information to support if needed.

**Context & notes**:
- Must persist: timestamp, error code, severity, description, root cause, user session context (if applicable), resolution status.
- Retention period: minimum 90 days `[TBD — confirm with ops/compliance]`.
- Must integrate with existing system message persistence layer.
- Events must survive application restarts (DB-backed, not in-memory only).
- Consider: should resolved events be marked as resolved? (e.g., after credentials are updated successfully)

**Acceptance Criteria**

AC1:
  Given: Any licensing validation error occurs
  When: The error is detected by the system
  Then: A complete audit record is persisted containing: timestamp, error code, severity, mapped description, root cause category, and system context.

AC2:
  Given: Licensing errors have been occurring for 90 days
  When: An admin queries the system message log
  Then: All events within the 90-day retention window are available and complete.

AC3:
  Given: The application server is restarted
  When: The admin accesses system message log after restart
  Then: All previously persisted licensing error events are intact and accessible.

AC4:
  Given: A licensing error was previously logged
  When: The root cause is resolved (e.g., credentials updated, license revalidated successfully)
  Then: The original error event is annotated with resolution timestamp `[TBD — confirm if resolution tracking is in scope]`.

**Tasks**
- [ ] Task 1: (Backend) Define persistence schema for licensing error audit events
- [ ] Task 2: (Backend) Implement event persistence on every licensing error detection
- [ ] Task 3: (Backend) Implement retention policy (90-day minimum; configurable)
- [ ] Task 4: (Backend) Implement resolution tracking annotation (if confirmed in scope)
- [ ] Task 5: (Installer/config) Include audit table in DB migration script; include in backup scope documentation
- [ ] Task 6: (Test) Write tests for persistence, retention enforcement, and survival across restarts

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] Retention period confirmed with ops/compliance
- [ ] Resolution tracking decision made (in/out of scope)
- [ ] Existing persistence layer reviewed for extensibility
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

### STORY: S9 — Update Documentation and Release Notes

**Parent Epic**: Licensing Error Contextual Display & Remediation Guidance
**BLSS Product**: Shared Platform Services / Lifecycle Management Services

**Description**
As an Eaton Field Service Engineer,
I want updated documentation covering the new licensing error display capability,
So that I can guide customers on using the feature and Lifecycle Services can include it in deployment enablement.

**Context & notes**:
- Must update: Administration Guide, Web Interface User Guide, Release Notes.
- Must create: Error code reference document for support team.
- Must update: Lifecycle Services deployment runbook (if install steps change).
- Consider: Knowledge base article for common vCenter credential rotation scenario.

**Acceptance Criteria**

AC1:
  Given: The licensing error display feature is complete
  When: Documentation is reviewed
  Then: The Administration Guide includes a section on licensing error messages, error code catalog, and how to extend it.

AC2:
  Given: The feature is released
  When: Release notes are reviewed
  Then: They include a clear description of the new licensing error display capability with screenshots.

AC3:
  Given: A Lifecycle Services engineer is deploying BLSS
  When: They reference the deployment runbook
  Then: It includes steps for verifying error catalog deployment and initial configuration.

**Tasks**
- [ ] Task 1: (Documentation) Update Administration Guide with error catalog management section
- [ ] Task 2: (Documentation) Update Web Interface User Guide with licensing error message screenshots and workflow
- [ ] Task 3: (Documentation) Draft release notes entry with feature description and screenshots
- [ ] Task 4: (Documentation) Create error code reference document for Brightlayer Support team
- [ ] Task 5: (Service enablement) Update Lifecycle Services deployment runbook
- [ ] Task 6: (Documentation) Create knowledge base article for vCenter credential rotation scenario

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] All feature stories (S1-S8) complete or near-complete
- [ ] Screenshots available from test environment
- [ ] Lifecycle Services team engaged for runbook review
- [ ] Dependencies identified and unblocked
- [ ] Estimated by team

---

---

## Validation Checklist

- [x] All AC are testable
- [x] NFRs are quantified (< 2s render, < 100ms lookup, 90-day retention, WCAG 2.1 AA)
- [x] BLSS product(s) explicitly identified for every Epic and Story (Shared Platform Services)
- [x] Scope boundaries are explicit (in-scope / out-of-scope defined)
- [x] Cross-product dependencies are identified (shared UI layer, all products benefit)
- [x] On-prem operational impact assessed (install, upgrade, sizing, network, backup)
- [x] Dependencies are identified (licensing engine, error catalog, system message framework, UX)
- [x] No invented features, UI labels, device models, or protocol details (TBD placeholders used)
- [x] Terminology matches BLSS product glossary
- [x] Placeholders marked with `[TBD]`
- [x] Operational deliverables included (runbooks, sizing guide, release notes)
- [x] Service enablement (Lifecycle Services) considered

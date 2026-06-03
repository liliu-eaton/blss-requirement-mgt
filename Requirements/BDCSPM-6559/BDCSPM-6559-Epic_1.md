# BDCSPM-6559 — Backup Enhancements

## EPIC 1: Backup Plan Configuration & Scheduling

**BLSS Product(s)**: Shared platform services (Server Admin)
**Summary** (1-2 sentences)
Provide On-Premises IT / System Admins and Eaton Field Service Engineers with the ability to configure backup plans (full/incremental), set flexible scheduling (daily/weekly/monthly), and enable or disable automatic backups through the Server Admin UI, so that BLSS environments have reliable, customizable data protection aligned to customer operational windows.

---

### Description

- **Problem statement**: The current BLSS backup mechanism offers limited configurability — administrators cannot choose between full and incremental backup types, cannot adjust backup schedules beyond a fixed daily window, and cannot disable automatic backups when using 3rd-party backup solutions (wasting disk space). This forces customers into a one-size-fits-all backup strategy that does not align with diverse operational requirements.

- **Proposed solution**: Enhance the Server Admin > Backup & Restore page to support:
  1. Selection of backup type: Full or Incremental
  2. Configurable schedule frequency: Daily, Weekly (with day-of-week selector), or Monthly (with day-of-month selector)
  3. Configurable backup time
  4. A master enable/disable toggle with confirmation dialog for disabling
  5. A live schedule summary showing the next planned backup run
  6. Backend scheduler service that executes backups per the saved configuration

- **User personas**: On-Prem IT / System Admin, Eaton Field Service Engineer

- **Business value**: Reduces customer complaints about inflexible backup windows, enables alignment with customer maintenance schedules, and eliminates wasted disk space for customers using 3rd-party backup tools. Estimated to reduce backup-related support tickets by `[TBD — confirm with support team]`%.

- **Key workflows**:
  1. Admin navigates to Server Admin > Backup & Restore > Backup Plan tab
  2. Admin selects backup type (Full / Incremental)
  3. Admin configures schedule (frequency, time, day-of-week/month)
  4. Admin reviews schedule summary preview
  5. Admin saves configuration (confirmation modal with change summary)
  6. Backend scheduler picks up new configuration and runs backups accordingly
  7. Alternatively: Admin disables backup (confirmation modal warns about consequences)

- **Cross-product touchpoints**: The backup covers all BLSS products installed on the Application Server (DCPM, EPMS Standard, Distributed IT, Brightlayer Power databases and configuration). No product-specific API integration required — backup operates at the shared services layer.

- **UX/UI references**: See prototype `Requirements/BDCSPM-6559/prototype-backup-enhancements.html` — "Backup Configuration" card and schedule fields.

- **On-prem operational impact**:
  - Install/upgrade impact: New backup configuration schema (config file or DB table) must be created during install; migration script must preserve existing backup settings and map them to the new schema.
  - Sizing impact: Incremental backup reduces disk usage compared to full; no additional CPU/RAM beyond existing backup agent.
  - Network impact: None for this Epic (remote backup is Epic 3).
  - Backup impact: This Epic IS the backup configuration feature. Schema migration must be backward-compatible. Existing backup files must be retained during upgrade.

---

### Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Performance | Backup configuration save (UI → backend) response time | < 1s at P95 on reference hardware (Medium tier) | P1 |
| Performance | Full backup completion time for Medium tier (5,000 devices, ~50 GB data) | ≤ 4 hours `[TBD — confirm with architecture team]` | P1 |
| Performance | Incremental backup completion time for Medium tier | ≤ 30 minutes `[TBD — confirm with architecture team]` | P1 |
| Reliability | Scheduled backup must execute within ±5 minutes of configured time | 99.5% of scheduled runs | P0 |
| Reliability | Backup process must not crash or corrupt data on unexpected server shutdown during backup | Graceful abort with partial cleanup | P0 |
| Usability | Backup configuration UI must be accessible and operable within 3 clicks from Server Admin landing | ≤ 3 clicks | P2 |
| Security | Backup configuration changes must be audit-logged with user identity and timestamp | 100% of changes logged | P1 |
| Compatibility | Must work on supported OS (RHEL 9/10) and PostgreSQL 14+ | All supported combinations | P0 |
| Data integrity | Backup file integrity must be verifiable (checksum) | SHA-256 checksum per backup | P1 |

---

### Acceptance Criteria (Epic-level)

- [ ] AC1: An On-Prem IT / System Admin can select Full or Incremental backup type and save the selection; the next scheduled backup uses the selected type.
- [ ] AC2: An admin can configure backup schedule as Daily (with time), Weekly (with day + time), or Monthly (with day-of-month + time); the scheduler executes accordingly.
- [ ] AC3: An admin can disable automatic backups via toggle; a confirmation dialog warns that no automatic backups will be performed; the scheduler stops executing.
- [ ] AC4: An admin can re-enable automatic backups after disabling; the scheduler resumes with the saved configuration.
- [ ] AC5: The Backup Plan tab displays a schedule summary showing the next planned backup type, frequency, and time.
- [ ] AC6: Configuration changes are persisted across Application Server restarts.
- [ ] AC7: Upgrade from previous BLSS version preserves existing backup schedule (mapped to new schema).

---

### Scope

- **In scope**:
  - Backup type selection: Full, Incremental
  - Schedule configuration: Daily, Weekly, Monthly with time and day selectors
  - Enable/disable backup toggle with confirmation
  - Schedule summary display with next-run preview
  - Backend configuration persistence (config file or DB table)
  - Backend scheduled job execution for full and incremental backups
  - Audit logging of configuration changes
  - Migration script for existing backup settings

- **Out of scope**:
  - Remote backup location (Epic 3)
  - Retention/rotation policies (Epic 2)
  - Backup & restore history UI (Epic 3)
  - Multi-node / distributed backup coordination
  - Application-level backup (e.g., per-product selective backup)
  - Cloud backup destinations

- **Assumptions**:
  1. ⚠️ ASSUMPTION: Incremental backup is technically feasible with the current BLSS database and file system architecture. Confirm with architecture team.
  2. ⚠️ ASSUMPTION: The backup scheduler is a system-level service (systemd timer or cron equivalent on RHEL) managed by the BLSS platform, not an external dependency.
  3. ⚠️ ASSUMPTION: "Full backup" includes all BLSS databases (PostgreSQL), configuration files, and application state. Confirm exact scope with architecture team.
  4. ⚠️ ASSUMPTION: Incremental backup uses PostgreSQL WAL archiving or equivalent mechanism. `[TBD — confirm implementation approach]`
  5. ⚠️ ASSUMPTION: The prototype HTML represents the approved UX direction.
  6. ⚠️ ASSUMPTION: Single-node (All-in-One) deployment is the primary scope.

---

### Dependencies & Risks

| Item | Type (Dep/Risk) | BLSS Product | Owner | Mitigation | Status |
|---|---|---|---|---|---|
| Incremental backup technical feasibility | Risk | Shared platform | Architecture team | Spike/PoC to validate incremental approach before committing stories | `[TBD]` |
| Existing backup schema migration | Dep | Shared platform | Platform team | Identify current schema early; write forward + rollback migration scripts | `[TBD]` |
| Backup agent service compatibility with RHEL 9/10 | Risk | Shared platform | Platform team | Validate on both OS versions in CI | `[TBD]` |
| UX design finalization (BDCSPM-38263) | Dep | Shared platform | UX team | Prototype exists; confirm design review is complete | Funnel |
| SA Evaluation completion (BDCSPM-38262) | Dep | Shared platform | Architecture team | Must complete architecture evaluation before implementation | Backlog |

---

### Operational deliverables (required for on-prem)

- [ ] Installation/upgrade runbook updated — include new backup configuration schema, default settings, migration steps
- [ ] Sizing guide updated — document disk space impact of full vs. incremental backup by sizing tier
- [ ] Backup/restore procedure updated — this IS the backup procedure; must be rewritten
- [ ] Firewall/port documentation updated — N/A for this Epic
- [ ] Release notes entry drafted — describe new backup configuration capabilities
- [ ] Lifecycle Services training/enablement updated — train on new backup plan configuration options

---

### Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| S1.1 | Configure backup type and schedule frequency | 5 | `[TBD]` |
| S1.2 | Enable/disable automatic backup toggle with confirmation | 3 | `[TBD]` |
| S1.3 | Display backup schedule summary with next-run preview | 3 | `[TBD]` |
| S1.4 | Persist and load backup configuration settings | 5 | `[TBD]` |
| S1.5 | Execute scheduled backup job (full backup) | 8 | `[TBD]` |
| S1.6 | Execute scheduled backup job (incremental backup) | 8 | `[TBD]` |

---
---

## Stories for Epic 1

---

### STORY: S1.1 — Configure Backup Type and Schedule Frequency

**Parent Epic**: Epic 1 — Backup Plan Configuration & Scheduling
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want to select the backup type (Full or Incremental) and configure the schedule frequency (Daily, Weekly, or Monthly) with a specific time and day,
So that backups align with my data center's operational maintenance windows and storage constraints.

**Context & notes**:
- See prototype `Requirements/BDCSPM-6559/prototype-backup-enhancements.html` — "Backup Configuration" card: Backup Type dropdown, Schedule Frequency dropdown, Backup Time input, Weekly Day selector, Monthly Day selector.
- When frequency = Daily: only Backup Time is shown.
- When frequency = Weekly: Day-of-Week selector appears.
- When frequency = Monthly: Day-of-Month selector appears (1st, 7th, 14th, 15th, 28th, Last day).
- The form fields are only visible when backup is enabled (see S1.2).
- On-prem consideration: configuration must be persisted locally (see S1.4); no external dependency.

**Acceptance Criteria**

AC1:
  Given: The admin is on the Backup Plan tab and backup is enabled
  When: The admin selects "Full Backup" from the Backup Type dropdown
  Then: The backup type is set to Full and the schedule summary updates to reflect "Full Backup"

AC2:
  Given: The admin is on the Backup Plan tab
  When: The admin selects "Incremental Backup" from the Backup Type dropdown
  Then: The backup type is set to Incremental and the schedule summary updates to reflect "Incremental Backup"

AC3:
  Given: The admin selects schedule frequency "Weekly"
  When: The form renders
  Then: A Day-of-Week selector appears (Sunday–Saturday) and the Monthly Day selector is hidden

AC4:
  Given: The admin selects schedule frequency "Monthly"
  When: The form renders
  Then: A Day-of-Month selector appears (1st, 7th, 14th, 15th, 28th, Last day) and the Weekly Day selector is hidden

AC5:
  Given: The admin selects schedule frequency "Daily"
  When: The form renders
  Then: Neither the Day-of-Week nor Day-of-Month selector is visible; only Backup Time is shown

AC6 (negative):
  Given: The admin has not changed any field
  When: The admin clicks "Save Configuration"
  Then: The system saves the current (unchanged) values without error

**Tasks**
- [ ] Task 1: <Frontend> Implement Backup Type dropdown (Full / Incremental) per prototype design
- [ ] Task 2: <Frontend> Implement Schedule Frequency dropdown (Daily / Weekly / Monthly) with conditional day selectors
- [ ] Task 3: <Frontend> Implement Backup Time input field
- [ ] Task 4: <Frontend> Wire conditional show/hide logic for Weekly Day and Monthly Day selectors based on frequency selection
- [ ] Task 5: <Test> Write unit tests for conditional form rendering (all 3 frequencies)
- [ ] Task 6: <Test> Write E2E test for backup type + schedule configuration flow

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — prototype HTML available
- [ ] API contract defined — N/A (frontend only; persistence in S1.4)
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — N/A
- [ ] On-prem install/upgrade impact assessed — minimal (UI only)
- [ ] Sizing impact assessed — N/A
- [ ] Dependencies identified and unblocked — depends on S1.4 for save; can be developed in parallel
- [ ] NFRs applicable to this story are noted — Usability: ≤ 3 clicks
- [ ] Estimated by team

---

### STORY: S1.2 — Enable/Disable Automatic Backup Toggle with Confirmation

**Parent Epic**: Epic 1 — Backup Plan Configuration & Scheduling
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want to enable or disable automatic backups via a toggle switch, with a confirmation dialog when disabling,
So that I can opt out of built-in backups when using a 3rd-party backup solution and save disk space, while being warned about the consequences.

**Context & notes**:
- See prototype: Enable/disable toggle at top of Backup Configuration card.
- When disabled: all backup configuration fields are hidden; a warning banner is displayed.
- When re-enabled: all fields reappear with previously saved values.
- Disabling triggers a modal confirmation: "Are you sure you want to disable automatic backups?" with warning text about no automatic backups being performed.
- Existing backup files must NOT be deleted when backup is disabled.
- On-prem consideration: customers using Veeam, Commvault, or other 3rd-party tools need this to avoid redundant disk usage.

**Acceptance Criteria**

AC1:
  Given: Backup is currently enabled
  When: The admin toggles the backup switch to OFF
  Then: A confirmation modal appears with warning text: "No automatic backups will be performed. Only disable this if you are using a 3rd-party backup solution. Existing backup files will be retained."

AC2:
  Given: The confirmation modal is open
  When: The admin clicks "Disable Backup"
  Then: The toggle moves to OFF, all configuration fields are hidden, a warning banner appears, and a success toast "Automatic backup has been disabled" is shown

AC3:
  Given: The confirmation modal is open
  When: The admin clicks "Cancel"
  Then: The modal closes and the toggle remains in the ON position; no changes are made

AC4:
  Given: Backup is currently disabled
  When: The admin toggles the backup switch to ON
  Then: The configuration fields reappear with the last saved values; a success toast "Automatic backup has been enabled" is shown; no confirmation modal is needed

AC5 (negative):
  Given: Backup is disabled
  When: The scheduled backup time arrives
  Then: No backup job is executed; no error is logged (expected behavior)

AC6:
  Given: Backup was disabled
  When: The admin navigates away and returns to the Backup & Restore page
  Then: The toggle remains OFF, the warning banner is visible, and configuration fields remain hidden

**Tasks**
- [ ] Task 1: <Frontend> Implement backup enable/disable toggle switch per prototype design
- [ ] Task 2: <Frontend> Implement "Disable Backup" confirmation modal with warning text
- [ ] Task 3: <Frontend> Implement conditional show/hide of configuration fields and warning banner based on toggle state
- [ ] Task 4: <Frontend> Implement toast notifications for enable/disable actions
- [ ] Task 5: <Backend> Add `backupEnabled` flag to backup configuration; scheduler checks flag before executing
- [ ] Task 6: <Test> Write unit tests for toggle state transitions (enable → disable → enable)
- [ ] Task 7: <Test> Write E2E test for disable confirmation flow (confirm + cancel paths)

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — prototype HTML available
- [ ] API contract defined — `backupEnabled` flag in configuration
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — N/A
- [ ] On-prem install/upgrade impact assessed — new config flag; default = enabled (backward-compatible)
- [ ] Sizing impact assessed — disabling saves disk space (customer benefit)
- [ ] Dependencies identified and unblocked — S1.4 for persistence
- [ ] NFRs applicable to this story are noted — Reliability: backup must not execute when disabled
- [ ] Estimated by team

---

### STORY: S1.3 — Display Backup Schedule Summary with Next-Run Preview

**Parent Epic**: Epic 1 — Backup Plan Configuration & Scheduling
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want to see a real-time summary of the configured backup schedule showing the next planned backup type, frequency, and time,
So that I can quickly verify the backup configuration is correct without reading individual form fields.

**Context & notes**:
- See prototype: Blue schedule summary banner with calendar icon: "Next backup: **Full Backup** scheduled **daily at 02:00**"
- The summary updates dynamically as the admin changes any configuration field (backup type, frequency, time, day).
- This is a read-only display element; it does not modify configuration.
- Only visible when backup is enabled.

**Acceptance Criteria**

AC1:
  Given: Backup type = Full, Frequency = Daily, Time = 02:00
  When: The schedule summary renders
  Then: It displays "Next backup: Full Backup scheduled daily at 02:00"

AC2:
  Given: Backup type = Incremental, Frequency = Weekly, Day = Sunday, Time = 03:00
  When: The schedule summary renders
  Then: It displays "Next backup: Incremental Backup scheduled every Sunday at 03:00"

AC3:
  Given: Backup type = Full, Frequency = Monthly, Day = 15th, Time = 01:00
  When: The schedule summary renders
  Then: It displays "Next backup: Full Backup scheduled monthly on the 15th at 01:00"

AC4:
  Given: The admin changes the backup type from Full to Incremental
  When: The dropdown value changes
  Then: The schedule summary updates immediately without requiring a page refresh or save

AC5 (edge case):
  Given: Frequency = Monthly, Day = Last day
  When: The schedule summary renders
  Then: It displays "...scheduled monthly on the last day at [time]"

**Tasks**
- [ ] Task 1: <Frontend> Implement schedule summary banner component per prototype design (blue left-border card with calendar icon)
- [ ] Task 2: <Frontend> Implement dynamic text generation based on backup type, frequency, time, and day values
- [ ] Task 3: <Frontend> Wire real-time updates on any form field change (no save required)
- [ ] Task 4: <Test> Write unit tests for all schedule text permutations (3 frequencies × 2 types = 6 base cases + edge cases)

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — prototype HTML available
- [ ] API contract defined — N/A (frontend display only)
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — N/A
- [ ] On-prem install/upgrade impact assessed — N/A (UI only)
- [ ] Sizing impact assessed — N/A
- [ ] Dependencies identified and unblocked — depends on S1.1 form fields being implemented
- [ ] NFRs applicable to this story are noted — Usability
- [ ] Estimated by team

---

### STORY: S1.4 — Persist and Load Backup Configuration Settings

**Parent Epic**: Epic 1 — Backup Plan Configuration & Scheduling
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want backup configuration settings to be saved to persistent storage and loaded on page open,
So that my configuration survives Application Server restarts and is available to the backup scheduler service.

**Context & notes**:
- See prototype: "Save Configuration" button triggers a confirmation modal with change summary; "Reset" button reverts to last saved values.
- Configuration includes: backupEnabled, backupType, scheduleFrequency, backupTime, weeklyDay, monthlyDay.
- Storage mechanism: `[TBD — confirm with architecture team: config file (e.g., settings.xml / YAML) or DB table]`
- The save action must be atomic — partial saves must not leave the system in an inconsistent state.
- Validation: backupsToKeep (1–100 range) is validated client-side before save; server-side validation required.
- On-prem consideration: configuration must be included in backup scope itself (meta-backup); must survive upgrade with migration script.

**Acceptance Criteria**

AC1:
  Given: The admin has modified backup configuration fields
  When: The admin clicks "Save Configuration"
  Then: A confirmation modal appears showing a summary of all current settings (Backup Type, Schedule, Backups to Keep, Rotation Policy, Remote Backup status)

AC2:
  Given: The confirmation modal is displayed
  When: The admin clicks "Confirm & Save"
  Then: The configuration is persisted to storage, the modal closes, and a success toast "Backup configuration saved successfully" is shown

AC3:
  Given: The confirmation modal is displayed
  When: The admin clicks "Cancel"
  Then: The modal closes; no changes are saved; form retains unsaved modifications

AC4:
  Given: Configuration has been previously saved
  When: The admin navigates to the Backup & Restore page
  Then: All form fields are populated with the last saved configuration values

AC5:
  Given: The admin has unsaved changes
  When: The admin clicks "Reset"
  Then: All form fields revert to the last saved values

AC6:
  Given: The Application Server is restarted
  When: The admin opens the Backup & Restore page after restart
  Then: The last saved configuration is loaded correctly

AC7 (negative — validation):
  Given: The admin enters an invalid value (e.g., backupsToKeep = 0 or > 100)
  When: The admin clicks "Save Configuration"
  Then: A validation error is displayed; the configuration is NOT saved

AC8 (upgrade):
  Given: The BLSS Application Server is upgraded from a previous version with legacy backup settings
  When: The upgrade migration runs
  Then: The legacy backup settings are mapped to the new configuration schema; no data loss

**Tasks**
- [ ] Task 1: <Backend> Create REST API endpoint `PUT /api/v2/admin/backup/config` to save backup configuration
- [ ] Task 2: <Backend> Create REST API endpoint `GET /api/v2/admin/backup/config` to load backup configuration
- [ ] Task 3: <Data/schema> Define backup configuration schema (backupEnabled, backupType, scheduleFrequency, backupTime, weeklyDay, monthlyDay) — `[TBD: config file or DB table]`
- [ ] Task 4: <Backend> Implement server-side validation for all configuration fields
- [ ] Task 5: <Frontend> Wire Save Configuration button to PUT endpoint with confirmation modal
- [ ] Task 6: <Frontend> Wire page load to GET endpoint to populate form fields
- [ ] Task 7: <Frontend> Implement Reset button to reload last saved configuration
- [ ] Task 8: <Upgrade/migration> Write migration script to map legacy backup settings to new schema; include rollback script
- [ ] Task 9: <Installer/config> Update installer to seed default backup configuration (backupEnabled=true, backupType=full, scheduleFrequency=daily, backupTime=02:00)
- [ ] Task 10: <Test> Write integration tests for save/load API endpoints (happy path + validation errors)
- [ ] Task 11: <Test> Write upgrade migration test (legacy → new schema)
- [ ] Task 12: <Documentation> Update Server Admin API documentation for new backup config endpoints

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — prototype HTML available
- [ ] API contract defined — PUT/GET `/api/v2/admin/backup/config` `[TBD — confirm with architecture team]`
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — N/A
- [ ] On-prem install/upgrade impact assessed — new schema + migration script + installer defaults
- [ ] Sizing impact assessed — negligible (config data is small)
- [ ] Dependencies identified and unblocked — schema decision from architecture team
- [ ] NFRs applicable to this story are noted — Performance: < 1s save; Security: audit logging; Data integrity: atomic save
- [ ] Estimated by team

---

### STORY: S1.5 — Execute Scheduled Backup Job (Full Backup)

**Parent Epic**: Epic 1 — Backup Plan Configuration & Scheduling
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want the system to execute a full backup automatically at the scheduled time,
So that all BLSS data (databases, configuration files, application state) is protected without manual intervention.

**Context & notes**:
- Full backup captures all PostgreSQL databases, BLSS configuration files, and application state files.
- The backup agent/service reads the saved configuration (S1.4) to determine schedule and type.
- Backup output is stored in a local directory `[TBD — confirm default backup path with architecture team]`.
- Each backup file must include a SHA-256 checksum for integrity verification.
- Backup must not degrade BLSS application performance beyond acceptable thresholds during execution.
- On-prem consideration: must work on RHEL 9/10; must handle disk-full conditions gracefully.

**Acceptance Criteria**

AC1:
  Given: Backup is enabled, type = Full, schedule = Daily at 02:00
  When: The system clock reaches 02:00
  Then: A full backup job starts, capturing all BLSS databases and configuration files

AC2:
  Given: A full backup job is executing
  When: The backup completes successfully
  Then: A backup archive file is created in the local backup directory with a SHA-256 checksum file, and the backup history is updated with status "Success"

AC3:
  Given: A full backup job is executing
  When: The backup fails (e.g., disk full, database lock timeout)
  Then: The backup history is updated with status "Failed" and an error description; an alarm is raised in the system log; partial backup files are cleaned up

AC4:
  Given: Backup is enabled, schedule = Weekly on Sunday at 03:00
  When: Sunday 03:00 arrives
  Then: A full backup job starts; on other days, no backup job is triggered

AC5 (negative):
  Given: Backup is disabled
  When: The scheduled backup time arrives
  Then: No backup job is executed

AC6:
  Given: A full backup is in progress
  When: The BLSS application is under normal operational load
  Then: Application response times do not degrade by more than 20% `[TBD — confirm threshold with architecture team]`

**Tasks**
- [ ] Task 1: <Backend> Implement backup scheduler service (systemd timer or equivalent) that reads configuration and triggers backup at scheduled time
- [ ] Task 2: <Backend> Implement full backup execution: PostgreSQL `pg_dump` (or `pg_basebackup`) + configuration file archival
- [ ] Task 3: <Backend> Implement backup archive packaging (tar.gz or equivalent) with SHA-256 checksum generation
- [ ] Task 4: <Backend> Implement backup status recording to backup history (success/failure/warning + size + duration)
- [ ] Task 5: <Backend> Implement error handling: disk-full detection, database lock timeout, partial cleanup on failure
- [ ] Task 6: <Backend> Implement alarm/log entry on backup failure
- [ ] Task 7: <Installer/config> Configure default backup output directory in installer; ensure directory permissions are correct
- [ ] Task 8: <Test> Write integration tests for scheduled full backup execution (success + failure scenarios)
- [ ] Task 9: <Test> Write performance test to verify application degradation stays within threshold during backup
- [ ] Task 10: <Documentation> Document full backup contents, output format, and checksum verification procedure

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — N/A (backend service)
- [ ] API contract defined — backup scheduler reads config from S1.4 storage
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — backup covers all installed BLSS products
- [ ] On-prem install/upgrade impact assessed — new scheduler service; backup directory creation
- [ ] Sizing impact assessed — full backup size depends on data volume; must document per sizing tier
- [ ] Dependencies identified and unblocked — S1.4 (configuration persistence)
- [ ] NFRs applicable to this story are noted — Performance: ≤ 4h (Medium tier); Reliability: 99.5% on-time; Data integrity: SHA-256
- [ ] Estimated by team

---

### STORY: S1.6 — Execute Scheduled Backup Job (Incremental Backup)

**Parent Epic**: Epic 1 — Backup Plan Configuration & Scheduling
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want the system to execute an incremental backup that captures only changes since the last backup,
So that backup storage requirements and backup duration are minimized while still protecting my data.

**Context & notes**:
- Incremental backup captures only data changed since the last successful full or incremental backup.
- Implementation approach: `[TBD — confirm with architecture team: PostgreSQL WAL archiving, file-level diff, or block-level diff]`
- The first incremental backup after enabling incremental mode must automatically perform a full backup as the baseline.
- Restore from incremental requires the base full backup plus all subsequent incremental backups — this must be documented.
- On-prem consideration: incremental backup reduces disk I/O and storage usage, which is important for customers with constrained infrastructure.

**Acceptance Criteria**

AC1:
  Given: Backup is enabled, type = Incremental, and a previous full backup exists
  When: The scheduled backup time arrives
  Then: An incremental backup job starts, capturing only changes since the last successful backup

AC2:
  Given: Backup is enabled, type = Incremental, and NO previous full backup exists
  When: The scheduled backup time arrives
  Then: A full backup is automatically performed as the baseline; a log entry notes "Initial full backup created for incremental baseline"

AC3:
  Given: An incremental backup completes successfully
  When: The backup finishes
  Then: A backup archive is created that is smaller than the full backup equivalent; backup history records type = "Incremental", status = "Success", and file size

AC4:
  Given: Incremental backup is configured
  When: The admin switches backup type from Incremental to Full and saves
  Then: The next scheduled backup performs a full backup; the previous incremental chain is preserved (not deleted)

AC5 (negative):
  Given: The last full backup (baseline) is corrupted or missing
  When: An incremental backup is triggered
  Then: The system detects the missing baseline, performs a full backup instead, logs a warning, and records it in backup history

AC6:
  Given: Incremental backup is in progress on Medium tier
  When: The backup completes
  Then: Backup duration is ≤ 30 minutes `[TBD — confirm with architecture team]`

**Tasks**
- [ ] Task 1: <Backend> Implement incremental backup mechanism — `[TBD: PostgreSQL WAL archiving / file-diff / block-diff]`
- [ ] Task 2: <Backend> Implement automatic full-backup baseline creation when no prior full backup exists
- [ ] Task 3: <Backend> Implement baseline integrity check before incremental execution; fall back to full if baseline is missing/corrupted
- [ ] Task 4: <Backend> Track incremental backup chain metadata (parent backup ID, sequence number)
- [ ] Task 5: <Backend> Implement incremental backup archive packaging with checksum
- [ ] Task 6: <Test> Write integration tests for incremental backup (with baseline, without baseline, corrupted baseline)
- [ ] Task 7: <Test> Write performance test comparing incremental vs. full backup duration and size
- [ ] Task 8: <Documentation> Document incremental backup mechanism, restore procedure (full + incremental chain), and limitations

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — N/A (backend service; UI type selector covered in S1.1)
- [ ] API contract defined — shares scheduler infrastructure with S1.5
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — same scope as S1.5 (all installed products)
- [ ] On-prem install/upgrade impact assessed — incremental mechanism may require PostgreSQL WAL configuration
- [ ] Sizing impact assessed — incremental reduces storage by `[TBD]`% vs. full
- [ ] Dependencies identified and unblocked — S1.4 (config), S1.5 (full backup as baseline)
- [ ] NFRs applicable to this story are noted — Performance: ≤ 30 min (Medium tier); Data integrity: checksum
- [ ] Estimated by team

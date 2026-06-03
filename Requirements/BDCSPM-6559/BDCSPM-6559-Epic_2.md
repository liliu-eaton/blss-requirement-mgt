# BDCSPM-6559 — Backup Enhancements

## EPIC 2: Retention & Rotation Policy Management

**BLSS Product(s)**: Shared platform services (Server Admin)
**Summary** (1-2 sentences)
Provide On-Premises IT / System Admins with configurable backup retention limits and rotation policies (FIFO, Grandfather-Father-Son, Tower of Hanoi) in the Server Admin UI, so that backup storage is managed automatically according to the customer's data protection strategy and disk capacity constraints.

---

### Description

- **Problem statement**: Without configurable retention and rotation policies, backup storage grows unbounded until disk space is exhausted, or administrators must manually delete old backups — a tedious and error-prone process. Customers have different data protection requirements: some need only the last few backups (FIFO), while others need a mix of recent and historical restore points (GFS). The current system does not offer this flexibility.

- **Proposed solution**: Enhance the Server Admin > Backup & Restore > Backup Plan tab to support:
  1. Configurable "Backups to Keep" count (1–100) with client-side and server-side validation
  2. Rotation policy selection: FIFO (default), Grandfather-Father-Son (GFS), Tower of Hanoi
  3. GFS sub-configuration: daily/weekly/monthly backup counts to keep
  4. Automatic cleanup of expired backup files per the selected rotation policy after each successful backup

- **User personas**: On-Prem IT / System Admin, Eaton Field Service Engineer

- **Business value**: Prevents disk-full incidents caused by unmanaged backup growth (reduces support escalations). Provides enterprise-grade data retention strategies that align with customer compliance and disaster recovery policies. Enables customers to optimize disk usage without manual intervention.

- **Key workflows**:
  1. Admin navigates to Server Admin > Backup & Restore > Backup Plan > Retention & Rotation Policy card
  2. Admin sets "Backups to Keep" count
  3. Admin selects rotation policy (FIFO / GFS / Tower of Hanoi)
  4. If GFS: admin configures daily, weekly, and monthly retention counts
  5. Admin saves configuration (via Epic 1's save flow)
  6. After each successful backup, the rotation engine evaluates existing backups and deletes expired ones per policy

- **Cross-product touchpoints**: None directly. The retention policy operates at the shared services layer and applies to all backup archives regardless of which BLSS products are installed.

- **UX/UI references**: See prototype `Requirements/BDCSPM-6559/prototype-backup-enhancements.html` — "Retention & Rotation Policy" sub-card within Backup Configuration.

- **On-prem operational impact**:
  - Install/upgrade impact: New rotation policy fields in backup configuration schema; migration must set default (FIFO, keep 5) for existing installations.
  - Sizing impact: Retention policy directly controls disk usage. Must document expected disk consumption per policy type per sizing tier.
  - Network impact: None.
  - Backup impact: The rotation engine deletes old backup files — must ensure deletion is logged and recoverable (soft-delete or confirmation before permanent removal).

---

### Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Performance | Rotation policy evaluation and cleanup after backup completion | ≤ 60 seconds for up to 100 backup files | P1 |
| Reliability | Rotation engine must never delete the most recent successful backup | 100% — enforced by code guard | P0 |
| Reliability | Rotation engine must handle concurrent access (no race condition with active backup) | Mutex/lock on backup directory during cleanup | P1 |
| Data integrity | Deleted backups must be logged with timestamp, filename, and reason | 100% of deletions logged | P1 |
| Usability | GFS sub-fields only visible when GFS policy is selected | Conditional rendering | P2 |
| Security | Only admin-role users can modify retention policy | Role-based access control | P1 |

---

### Acceptance Criteria (Epic-level)

- [ ] AC1: An admin can set the number of backups to keep (1–100); values outside this range are rejected with a validation error.
- [ ] AC2: An admin can select FIFO rotation policy; after each backup, the oldest backups beyond the retention count are automatically deleted.
- [ ] AC3: An admin can select GFS rotation policy and configure daily, weekly, and monthly retention counts; the system retains backups according to the GFS schedule.
- [ ] AC4: An admin can select Tower of Hanoi rotation policy; the system rotates backups per the Tower of Hanoi algorithm.
- [ ] AC5: The most recent successful backup is NEVER deleted by the rotation engine, regardless of policy configuration.
- [ ] AC6: All automatic backup deletions are logged with backup filename, deletion timestamp, and rotation policy reason.
- [ ] AC7: Upgrade from previous BLSS version sets retention policy to FIFO with 5 backups to keep (backward-compatible default).

---

### Scope

- **In scope**:
  - Backups-to-keep count configuration with validation (1–100)
  - FIFO rotation policy implementation
  - GFS rotation policy implementation with daily/weekly/monthly sub-configuration
  - Tower of Hanoi rotation policy implementation
  - Automatic cleanup execution after each successful backup
  - Deletion audit logging
  - Migration default for upgrades

- **Out of scope**:
  - Manual backup deletion from UI (users manage via OS file system if needed)
  - Backup archival to tape or cold storage
  - Policy-based retention across remote and local backups independently (same policy applies to both)
  - Custom rotation policy scripts

- **Assumptions**:
  1. ⚠️ ASSUMPTION: Rotation policy applies to local backup files. If remote backup is enabled (Epic 3), the same policy applies to remote copies. Confirm with architecture team if independent policies are needed.
  2. ⚠️ ASSUMPTION: Tower of Hanoi rotation is a standard implementation using logarithmic-depth backup sets. Confirm algorithm details with architecture team.
  3. ⚠️ ASSUMPTION: GFS policy categorizes backups as daily/weekly/monthly based on their creation date, not the schedule frequency configured in Epic 1.
  4. ⚠️ ASSUMPTION: Deletion is permanent (no recycle bin). Customers are expected to have their own file-system-level recoverability if needed.

---

### Dependencies & Risks

| Item | Type (Dep/Risk) | BLSS Product | Owner | Mitigation | Status |
|---|---|---|---|---|---|
| Epic 1 (S1.4) — Configuration persistence | Dep | Shared platform | Platform team | Retention fields stored in same config schema | `[TBD]` |
| Epic 1 (S1.5/S1.6) — Backup execution | Dep | Shared platform | Platform team | Rotation runs after backup completes | `[TBD]` |
| Accidental deletion of all backups | Risk | Shared platform | Platform team | Code guard: never delete the most recent successful backup; minimum retention = 1 | `[TBD]` |
| Tower of Hanoi complexity / customer confusion | Risk | Shared platform | UX team | Provide tooltip explanation in UI; consider help documentation link | `[TBD]` |
| Disk I/O during cleanup could overlap with next scheduled task | Risk | Shared platform | Architecture team | Implement mutex/lock on backup directory during rotation cleanup | `[TBD]` |

---

### Operational deliverables (required for on-prem)

- [ ] Installation/upgrade runbook updated — document default retention policy (FIFO, keep 5) and how to change it
- [ ] Sizing guide updated — document expected disk usage per rotation policy per sizing tier
- [ ] Backup/restore procedure updated — document that old backups are auto-deleted per policy; warn about recovery implications
- [ ] Firewall/port documentation updated — N/A
- [ ] Release notes entry drafted — describe new retention and rotation policy options
- [ ] Lifecycle Services training/enablement updated — train on FIFO vs. GFS vs. Tower of Hanoi trade-offs for customer advisement

---

### Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| S2.1 | Configure number of backups to keep with validation | 3 | `[TBD]` |
| S2.2 | Implement FIFO rotation policy (delete oldest first) | 5 | `[TBD]` |
| S2.3 | Implement Grandfather-Father-Son (GFS) rotation policy | 8 | `[TBD]` |
| S2.4 | Implement Tower of Hanoi rotation policy | 5 | `[TBD]` |
| S2.5 | Automatic cleanup of expired backups per rotation policy | 5 | `[TBD]` |

---
---

## Stories for Epic 2

---

### STORY: S2.1 — Configure Number of Backups to Keep with Validation

**Parent Epic**: Epic 2 — Retention & Rotation Policy Management
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want to specify how many backup copies to retain (between 1 and 100),
So that I can control disk usage while ensuring sufficient restore points are available.

**Context & notes**:
- See prototype: "Backups to Keep" numeric input field in the Retention & Rotation Policy sub-card.
- Default value: 5.
- Client-side validation: 1–100 range; error text "Must be between 1 and 100" shown below the field.
- Server-side validation must match client-side rules.
- This value is used by all rotation policies (FIFO uses it directly; GFS and Tower of Hanoi use it as a total cap).
- Persisted via the save flow in S1.4.

**Acceptance Criteria**

AC1:
  Given: The admin is on the Retention & Rotation Policy section
  When: The admin enters a value of 5 in the "Backups to Keep" field
  Then: No validation error is shown; the value is accepted

AC2:
  Given: The admin enters a value of 0 in the "Backups to Keep" field
  When: The field loses focus or the admin clicks Save
  Then: A validation error "Must be between 1 and 100" is displayed; save is blocked

AC3:
  Given: The admin enters a value of 101 in the "Backups to Keep" field
  When: The field loses focus or the admin clicks Save
  Then: A validation error "Must be between 1 and 100" is displayed; save is blocked

AC4:
  Given: The admin enters a non-numeric value (e.g., "abc")
  When: The field loses focus
  Then: A validation error is displayed; save is blocked

AC5:
  Given: The admin saves a valid "Backups to Keep" value
  When: The admin reloads the page
  Then: The saved value is displayed in the field

**Tasks**
- [ ] Task 1: <Frontend> Implement "Backups to Keep" numeric input field with min=1, max=100 per prototype
- [ ] Task 2: <Frontend> Implement client-side validation with error text display on invalid input
- [ ] Task 3: <Backend> Add `backupsToKeep` field to backup configuration schema; add server-side validation (1–100, integer)
- [ ] Task 4: <Upgrade/migration> Add `backupsToKeep = 5` as default in migration script for existing installations
- [ ] Task 5: <Test> Write unit tests for client-side validation (boundary values: 0, 1, 50, 100, 101, non-numeric)
- [ ] Task 6: <Test> Write API test for server-side validation

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — prototype HTML available
- [ ] API contract defined — `backupsToKeep` in config schema
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — N/A
- [ ] On-prem install/upgrade impact assessed — new config field with default
- [ ] Sizing impact assessed — value directly affects disk usage
- [ ] Dependencies identified and unblocked — S1.4 (config persistence)
- [ ] NFRs applicable to this story are noted — Usability: clear validation feedback
- [ ] Estimated by team

---

### STORY: S2.2 — Implement FIFO Rotation Policy (Delete Oldest First)

**Parent Epic**: Epic 2 — Retention & Rotation Policy Management
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want to select FIFO (First-In, First-Out) as the rotation policy,
So that the system automatically deletes the oldest backup files when the number of backups exceeds the configured retention count, keeping only the most recent backups.

**Context & notes**:
- FIFO is the default rotation policy.
- See prototype: "Rotation Policy" dropdown with "FIFO — Delete oldest first" as the first option.
- The FIFO engine runs after each successful backup (triggered in S2.5).
- Backups are ordered by creation timestamp; the oldest beyond `backupsToKeep` are deleted.
- The most recent successful backup must NEVER be deleted (safety guard).
- Deletion is permanent; each deletion is logged with filename, timestamp, and reason.

**Acceptance Criteria**

AC1:
  Given: Rotation policy = FIFO, Backups to Keep = 5, and there are currently 5 backup files
  When: A new backup completes successfully (6 total)
  Then: The oldest backup file is deleted; 5 backups remain

AC2:
  Given: Rotation policy = FIFO, Backups to Keep = 5, and there are currently 3 backup files
  When: A new backup completes successfully (4 total)
  Then: No backup files are deleted (4 < 5)

AC3:
  Given: Rotation policy = FIFO, Backups to Keep = 1
  When: A new backup completes successfully
  Then: Only the newest backup is retained; all older backups are deleted

AC4 (safety guard):
  Given: Rotation policy = FIFO, Backups to Keep = 1, and the only existing backup has status "Failed"
  When: A new backup completes successfully
  Then: The failed backup is deleted; the new successful backup is retained

AC5:
  Given: A backup file is deleted by FIFO rotation
  When: The deletion occurs
  Then: An audit log entry is created with: filename, deletion timestamp, reason = "FIFO rotation: exceeded retention count of [N]"

**Tasks**
- [ ] Task 1: <Backend> Implement FIFO rotation algorithm: sort backups by creation date ascending, delete oldest beyond retention count
- [ ] Task 2: <Backend> Implement safety guard: never delete the most recent successful backup
- [ ] Task 3: <Backend> Implement deletion audit logging (filename, timestamp, reason)
- [ ] Task 4: <Backend> Implement file deletion with error handling (file locked, permission denied)
- [ ] Task 5: <Test> Write unit tests for FIFO algorithm (exact count, under count, over count, edge case with 1 backup)
- [ ] Task 6: <Test> Write integration test for FIFO rotation after backup completion

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — N/A (backend logic; UI selector in S2.1)
- [ ] API contract defined — rotation engine invoked post-backup
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — N/A
- [ ] On-prem install/upgrade impact assessed — FIFO is default; no action needed for existing installs
- [ ] Sizing impact assessed — disk space freed by rotation
- [ ] Dependencies identified and unblocked — S2.1 (backups-to-keep), S2.5 (cleanup trigger)
- [ ] NFRs applicable to this story are noted — Reliability: never delete most recent; Performance: ≤ 60s cleanup
- [ ] Estimated by team

---

### STORY: S2.3 — Implement Grandfather-Father-Son (GFS) Rotation Policy

**Parent Epic**: Epic 2 — Retention & Rotation Policy Management
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want to select Grandfather-Father-Son (GFS) as the rotation policy and configure how many daily, weekly, and monthly backups to keep,
So that I can maintain a mix of recent and historical restore points aligned with enterprise data protection policies.

**Context & notes**:
- See prototype: When "Grandfather-Father-Son (GFS)" is selected in the Rotation Policy dropdown, three additional fields appear: Daily Backups to Keep (default 7, max 30), Weekly Backups to Keep (default 4, max 12), Monthly Backups to Keep (default 3, max 12).
- GFS classification logic:
  - "Son" (daily): the most recent N daily backups
  - "Father" (weekly): the most recent backup from each of the last N weeks (e.g., Sunday's backup)
  - "Grandfather" (monthly): the most recent backup from each of the last N months (e.g., 1st of month)
- A single backup file can satisfy multiple roles (e.g., a Sunday backup counts as both daily and weekly).
- On-prem consideration: GFS is commonly required by customers with regulatory compliance needs.

**Acceptance Criteria**

AC1:
  Given: Rotation policy = GFS, Daily = 7, Weekly = 4, Monthly = 3
  When: The admin selects GFS from the dropdown
  Then: Three sub-fields appear: "Daily Backups to Keep", "Weekly Backups to Keep", "Monthly Backups to Keep" with default values

AC2:
  Given: GFS is configured with Daily = 7, Weekly = 4, Monthly = 3
  When: Rotation runs after a successful backup
  Then: The system retains the 7 most recent daily backups, the last backup from each of the 4 most recent weeks, and the last backup from each of the 3 most recent months; all other backups are deleted

AC3:
  Given: A backup satisfies both the daily and weekly retention criteria
  When: Rotation evaluates this backup
  Then: The backup is retained (not deleted) and counted against both daily and weekly quotas

AC4:
  Given: The admin switches from GFS back to FIFO
  When: The admin selects FIFO from the dropdown
  Then: The GFS sub-fields are hidden; the next rotation uses FIFO logic

AC5 (validation):
  Given: The admin enters Daily = 0 in the GFS sub-field
  When: Validation runs
  Then: An error is shown; the value must be between 1 and 30

AC6:
  Given: GFS rotation deletes a backup
  When: The deletion occurs
  Then: An audit log entry is created with reason = "GFS rotation: backup does not satisfy daily/weekly/monthly retention criteria"

**Tasks**
- [ ] Task 1: <Frontend> Implement conditional rendering of GFS sub-fields (Daily/Weekly/Monthly keep counts) when GFS policy is selected
- [ ] Task 2: <Frontend> Implement validation for GFS sub-fields (Daily: 1–30, Weekly: 1–12, Monthly: 1–12)
- [ ] Task 3: <Backend> Add GFS configuration fields to backup config schema (gfsDailyKeep, gfsWeeklyKeep, gfsMonthlyKeep)
- [ ] Task 4: <Backend> Implement GFS classification algorithm: categorize each backup as Son/Father/Grandfather based on creation date
- [ ] Task 5: <Backend> Implement GFS retention evaluation: determine which backups to keep and which to delete
- [ ] Task 6: <Backend> Implement audit logging for GFS deletions
- [ ] Task 7: <Test> Write unit tests for GFS classification (daily boundaries, weekly boundaries, month boundaries, overlap cases)
- [ ] Task 8: <Test> Write integration test with realistic backup history spanning multiple weeks and months
- [ ] Task 9: <Documentation> Document GFS policy behavior, retention examples, and edge cases for admin guide

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — prototype HTML available
- [ ] API contract defined — GFS fields in config schema
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — N/A
- [ ] On-prem install/upgrade impact assessed — new config fields (not applicable to existing installs unless GFS is selected)
- [ ] Sizing impact assessed — GFS typically retains more total backups than FIFO; document disk impact
- [ ] Dependencies identified and unblocked — S2.1 (base retention config), S2.5 (cleanup trigger)
- [ ] NFRs applicable to this story are noted — Reliability: correct classification; never delete most recent backup
- [ ] Estimated by team

---

### STORY: S2.4 — Implement Tower of Hanoi Rotation Policy

**Parent Epic**: Epic 2 — Retention & Rotation Policy Management
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want to select Tower of Hanoi as the rotation policy,
So that the system uses a logarithmic-depth rotation scheme that provides exponentially spaced restore points with a fixed number of backup slots, optimizing the trade-off between storage usage and historical coverage.

**Context & notes**:
- See prototype: "Tower of Hanoi" is the third option in the Rotation Policy dropdown.
- Tower of Hanoi rotation uses the "Backups to Keep" count (from S2.1) as the number of backup slots (tiers).
- The algorithm assigns each backup to a tier based on a binary pattern: Tier 1 gets every other backup, Tier 2 every 4th, Tier 3 every 8th, etc.
- When a slot is reused, the previous backup in that slot is overwritten (deleted).
- ⚠️ NEEDS VALIDATION: Confirm the specific Tower of Hanoi variant to implement with the architecture team.
- On-prem consideration: provides the best historical coverage per unit of disk space — valuable for space-constrained deployments.

**Acceptance Criteria**

AC1:
  Given: Rotation policy = Tower of Hanoi, Backups to Keep = 5 (5 tiers)
  When: The admin selects Tower of Hanoi
  Then: No additional sub-fields are shown (uses "Backups to Keep" as the number of tiers)

AC2:
  Given: Tower of Hanoi with 5 tiers, and backups have been running for 2 weeks
  When: Rotation runs after a successful backup
  Then: The system retains backups distributed across tiers according to the Tower of Hanoi algorithm; older tiers contain exponentially older backups

AC3:
  Given: Tower of Hanoi rotation determines that a tier slot should be overwritten
  When: The rotation engine processes the slot
  Then: The old backup in that slot is deleted and replaced by the new backup; an audit log entry is created

AC4 (safety guard):
  Given: Tower of Hanoi rotation is active
  When: Any rotation cycle runs
  Then: The most recent successful backup is NEVER deleted

AC5:
  Given: Tower of Hanoi with 3 tiers
  When: 8 sequential backups have been created
  Then: Tier 1 holds backups 7 or 8 (most recent alternating), Tier 2 holds backup 6 or 4 (every 4th), Tier 3 holds backup 8 or earlier (every 8th) — exact behavior per confirmed algorithm variant `[TBD]`

**Tasks**
- [ ] Task 1: <Backend> Implement Tower of Hanoi rotation algorithm with configurable number of tiers
- [ ] Task 2: <Backend> Implement tier assignment logic based on backup sequence number
- [ ] Task 3: <Backend> Implement slot overwrite logic with old backup deletion
- [ ] Task 4: <Backend> Implement safety guard: never delete the most recent successful backup
- [ ] Task 5: <Backend> Implement audit logging for Tower of Hanoi deletions (include tier number)
- [ ] Task 6: <Test> Write unit tests for Tower of Hanoi algorithm (3, 5, and 7 tiers with various backup counts)
- [ ] Task 7: <Test> Write integration test for Tower of Hanoi over a simulated multi-week backup cycle
- [ ] Task 8: <Documentation> Document Tower of Hanoi behavior with examples for admin guide; include comparison table vs. FIFO and GFS

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — N/A (no additional UI fields)
- [ ] API contract defined — uses `backupsToKeep` as tier count
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — N/A
- [ ] On-prem install/upgrade impact assessed — algorithm selection only; no schema change beyond rotation policy field
- [ ] Sizing impact assessed — Tower of Hanoi uses fixed slot count; predictable disk usage
- [ ] Dependencies identified and unblocked — S2.1 (backups-to-keep), S2.5 (cleanup trigger); architecture team confirmation of algorithm variant
- [ ] NFRs applicable to this story are noted — Reliability: never delete most recent; correct tier assignment
- [ ] Estimated by team

---

### STORY: S2.5 — Automatic Cleanup of Expired Backups per Rotation Policy

**Parent Epic**: Epic 2 — Retention & Rotation Policy Management
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want expired backup files to be automatically cleaned up after each successful backup according to the configured rotation policy,
So that disk space is managed without manual intervention and I am not surprised by disk-full conditions.

**Context & notes**:
- This story implements the orchestration layer that triggers the appropriate rotation engine (FIFO / GFS / Tower of Hanoi) after each successful backup.
- The cleanup runs as a post-backup step within the backup scheduler flow (S1.5 / S1.6).
- If cleanup fails (e.g., file locked), the backup itself is still considered successful; the cleanup failure is logged as a warning and retried on the next cycle.
- Must acquire a mutex/lock on the backup directory to prevent race conditions with concurrent backup or restore operations.
- On-prem consideration: disk-full conditions can cause BLSS application failures; proactive cleanup is critical.

**Acceptance Criteria**

AC1:
  Given: A backup has completed successfully
  When: The post-backup cleanup step runs
  Then: The configured rotation policy engine is invoked; expired backups are identified and deleted

AC2:
  Given: Rotation policy = FIFO
  When: Cleanup runs
  Then: The FIFO engine (S2.2) is invoked; oldest backups beyond retention count are deleted

AC3:
  Given: Rotation policy = GFS
  When: Cleanup runs
  Then: The GFS engine (S2.3) is invoked; backups not satisfying daily/weekly/monthly criteria are deleted

AC4:
  Given: Rotation policy = Tower of Hanoi
  When: Cleanup runs
  Then: The Tower of Hanoi engine (S2.4) is invoked; overwritten tier slots are cleaned up

AC5 (failure handling):
  Given: Cleanup encounters a locked file that cannot be deleted
  When: The deletion attempt fails
  Then: The error is logged as a warning; the cleanup continues with remaining files; the locked file deletion is retried on the next cycle

AC6 (concurrency):
  Given: A restore operation is in progress
  When: The cleanup step attempts to run
  Then: The cleanup waits for the lock to be released or skips this cycle; it does not interfere with the restore

AC7:
  Given: Remote backup is enabled (Epic 3)
  When: Cleanup runs
  Then: The same rotation policy is applied to both local and remote backup files

**Tasks**
- [ ] Task 1: <Backend> Implement post-backup cleanup orchestrator that reads rotation policy from config and dispatches to the correct engine
- [ ] Task 2: <Backend> Implement mutex/lock acquisition on backup directory before cleanup; release after completion
- [ ] Task 3: <Backend> Implement cleanup error handling: log warning on individual file deletion failure; continue with remaining files
- [ ] Task 4: <Backend> Implement retry logic for failed deletions on next cleanup cycle
- [ ] Task 5: <Backend> Integrate cleanup orchestrator into backup scheduler flow (post-backup hook)
- [ ] Task 6: <Test> Write integration test for cleanup orchestration with each rotation policy
- [ ] Task 7: <Test> Write concurrency test: simultaneous backup and cleanup, simultaneous restore and cleanup
- [ ] Task 8: <Test> Write failure test: locked file during cleanup

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — N/A (backend orchestration)
- [ ] API contract defined — post-backup hook in scheduler
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — N/A
- [ ] On-prem install/upgrade impact assessed — cleanup is automatic; no admin action needed
- [ ] Sizing impact assessed — cleanup frees disk space; net positive
- [ ] Dependencies identified and unblocked — S2.2, S2.3, S2.4 (rotation engines); S1.5/S1.6 (backup scheduler)
- [ ] NFRs applicable to this story are noted — Performance: ≤ 60s; Reliability: mutex/lock, retry on failure
- [ ] Estimated by team

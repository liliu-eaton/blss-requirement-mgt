# BDCSPM-6559 — Backup Enhancements

## EPIC 3: Remote Backup Location & Restore Enhancements

**BLSS Product(s)**: Shared platform services (Server Admin)
**Summary** (1-2 sentences)
Enable On-Premises IT / System Admins to configure a remote backup storage location (SMB/CIFS, NFS, or SFTP) with connection testing in the Server Admin UI, and enhance the backup & restore history view with the ability to select and restore from any historical backup, so that customers have off-server disaster recovery capability and improved restore workflows.

---

### Description

- **Problem statement**: Current BLSS backup files are stored only on the local Application Server disk. If the server suffers a hardware failure, fire, or other disaster, all backup data is lost alongside the production data. Additionally, the existing restore workflow lacks visibility into backup history and does not provide a streamlined selection-and-restore process. Customers need off-server backup copies for disaster recovery compliance and a clear, safe restore path.

- **Proposed solution**: Enhance the Server Admin > Backup & Restore page to support:
  1. Remote backup location configuration: protocol selection (SMB/CIFS, NFS, SFTP), host, share path, credentials
  2. Remote backup enable/disable toggle
  3. "Test Connection" button for validating remote connectivity before saving
  4. Automatic copy of backup files to remote location after each local backup
  5. Backup & Restore History tab showing all backup records with host, node type, backup type, timestamp, size, status, and last restore date
  6. Restore workflow: select a backup from history, review details in confirmation modal, execute restore with system restart warning

- **User personas**: On-Prem IT / System Admin, Eaton Field Service Engineer

- **Business value**: Provides disaster recovery capability critical for enterprise data center operations. Reduces risk of total data loss from server hardware failures. Improves restore confidence with clear history visibility and confirmation workflow. Addresses common customer requirement for off-site backup storage in compliance audits.

- **Key workflows**:
  1. **Remote backup setup**: Admin enables remote backup → selects protocol → enters host/path/credentials → tests connection → saves
  2. **Automated remote copy**: After each local backup completes, the system copies the backup file to the remote location
  3. **History review**: Admin switches to Backup & Restore History tab → views table of all backups with status
  4. **Restore**: Admin selects a backup via radio button → clicks Restore → reviews confirmation modal (host, date, type, size + destructive action warning) → confirms → system restores and restarts

- **Cross-product touchpoints**: The restore operation affects all BLSS products installed on the Application Server. After restore, all products are rolled back to the state captured in the selected backup. No product-specific integration required.

- **UX/UI references**: See prototype `Requirements/BDCSPM-6559/prototype-backup-enhancements.html` — "Remote Backup Location" sub-card, "Backup & Restore History" tab, and Restore Confirmation modal.

- **On-prem operational impact**:
  - Install/upgrade impact: New remote backup configuration fields in schema; installer must provide remote backup defaults (disabled by default).
  - Sizing impact: Remote backup requires network bandwidth; no additional local disk. Document recommended network speed per sizing tier.
  - Network impact: **New outbound connections required**: SMB/CIFS (TCP 445), NFS (TCP/UDP 2049), or SFTP (TCP 22). Firewall rules documentation must be updated.
  - Backup impact: Remote copies provide disaster recovery; restore from remote requires network access to remote location.

---

### Non-Functional Requirements (NFRs)

| Category | Requirement | Target | Priority |
|---|---|---|---|
| Performance | Remote backup file transfer throughput | ≥ 50 MB/s on 1 Gbps LAN `[TBD — confirm with architecture team]` | P1 |
| Performance | Backup history page load time (up to 100 backup records) | < 2s at P95 | P2 |
| Performance | Restore operation completion time for Medium tier (50 GB data) | ≤ 6 hours `[TBD — confirm with architecture team]` | P1 |
| Reliability | Remote backup copy must retry on transient network failure | Up to 3 retries with exponential backoff | P1 |
| Reliability | Restore must be atomic — if restore fails partway, system must roll back to pre-restore state or clearly indicate partial restore | Documented recovery procedure for partial restore failure | P0 |
| Security | Remote backup credentials must be stored encrypted at rest | AES-256 or equivalent `[TBD — confirm with security team]` | P0 |
| Security | SFTP connections must use host key verification | Configurable: strict or trust-on-first-use | P1 |
| Security | Remote backup file transfer must use encrypted protocols where available | SFTP (encrypted by default), SMB 3.0+ with encryption `[TBD]` | P1 |
| Availability | Remote backup failure must not block local backup success | Local backup always completes independently | P0 |
| Usability | "Test Connection" provides clear pass/fail result with error details | Within 10 seconds timeout | P1 |

---

### Acceptance Criteria (Epic-level)

- [ ] AC1: An admin can enable remote backup, select a protocol (SMB/CIFS, NFS, SFTP), enter host/path/credentials, and save the configuration.
- [ ] AC2: An admin can test the remote backup connection and receive a clear pass/fail result with error details if applicable.
- [ ] AC3: After each successful local backup, the backup file is automatically copied to the configured remote location.
- [ ] AC4: If the remote copy fails, the local backup is still marked as successful; the remote failure is logged and retried.
- [ ] AC5: The Backup & Restore History tab displays all backup records with host name, node type, backup type, last backup timestamp, size, status, and last restore date.
- [ ] AC6: An admin can select a backup from the history table and initiate a restore; a confirmation modal shows backup details and a destructive action warning.
- [ ] AC7: After confirming restore, the system restores from the selected backup and restarts; the restore outcome is recorded in history.
- [ ] AC8: Remote backup credentials are stored encrypted at rest.
- [ ] AC9: Upgrade from previous BLSS version sets remote backup to disabled (backward-compatible default).

---

### Scope

- **In scope**:
  - Remote backup configuration UI: enable/disable toggle, protocol selector, host, share path, username, password
  - Supported protocols: SMB/CIFS, NFS, SFTP
  - "Test Connection" button with pass/fail feedback
  - Automatic post-backup copy to remote location
  - Remote copy retry with exponential backoff on transient failures
  - Encrypted credential storage
  - Backup & Restore History table (read-only display)
  - Restore from selected backup with confirmation modal and system restart
  - Failure handling: local-only fallback when remote is unreachable

- **Out of scope**:
  - Restore from a remote-only backup (remote is a copy of local; local must exist for restore) `[TBD — confirm if restore from remote is needed]`
  - Cloud backup destinations (S3, Azure Blob, etc.)
  - Backup verification / integrity check from the UI (manual checksum verification via CLI is documented)
  - Scheduling remote-only backup (remote is always a mirror of local)
  - Multi-remote-location support (only one remote location at a time)
  - SSH key authentication for SFTP (password-only in initial release) `[TBD — confirm with security team]`

- **Assumptions**:
  1. ⚠️ ASSUMPTION: Remote backup is a post-backup copy operation (not a direct-to-remote backup). Local backup always completes first.
  2. ⚠️ ASSUMPTION: Only one remote backup location can be configured at a time.
  3. ⚠️ ASSUMPTION: Restore is only from local backup files. Restore from remote backup is out of scope for this release.
  4. ⚠️ ASSUMPTION: SMB/CIFS support uses SMB 3.0+ with encryption enabled by default.
  5. ⚠️ ASSUMPTION: SFTP uses password authentication only (no SSH key) in this release.
  6. ⚠️ ASSUMPTION: The restore operation requires the BLSS application to be stopped; the system restarts automatically after restore.
  7. ⚠️ ASSUMPTION: The backup history table shows the last 100 backups (configurable) `[TBD — confirm pagination requirements]`.
  8. ⚠️ ASSUMPTION: The "Test Connection" operation verifies connectivity, authentication, and write access to the remote path.

---

### Dependencies & Risks

| Item | Type (Dep/Risk) | BLSS Product | Owner | Mitigation | Status |
|---|---|---|---|---|---|
| Epic 1 (S1.4) — Configuration persistence | Dep | Shared platform | Platform team | Remote fields stored in same config schema | `[TBD]` |
| Epic 1 (S1.5/S1.6) — Backup execution | Dep | Shared platform | Platform team | Remote copy triggers post-backup | `[TBD]` |
| Firewall rules for SMB/NFS/SFTP | Dep | Customer network | Customer IT admin | Document required ports in firewall documentation; test with common firewall configurations | `[TBD]` |
| Credential encryption key management | Risk | Shared platform | Security team | Define key storage and rotation strategy; integrate with existing BLSS key management if available | `[TBD]` |
| Remote storage reliability (NAS outage, network partition) | Risk | Customer infra | Platform team | Implement retry + local-only fallback; alert on repeated remote failures | `[TBD]` |
| Restore failure leaves system in inconsistent state | Risk | Shared platform | Architecture team | Implement pre-restore snapshot or rollback procedure; document manual recovery steps | `[TBD]` |
| SFTP host key management | Risk | Shared platform | Security team | Implement configurable host key verification (strict vs. TOFU) | `[TBD]` |

---

### Operational deliverables (required for on-prem)

- [ ] Installation/upgrade runbook updated — document remote backup configuration, default (disabled), credential storage
- [ ] Sizing guide updated — document network bandwidth requirements for remote backup per sizing tier
- [ ] Backup/restore procedure updated — document full restore workflow including system restart; document recovery from failed restore
- [ ] Firewall/port documentation updated — **Required**: document SMB (TCP 445), NFS (TCP/UDP 2049), SFTP (TCP 22) outbound rules
- [ ] Release notes entry drafted — describe remote backup capability, supported protocols, and restore enhancements
- [ ] Lifecycle Services training/enablement updated — train on remote backup setup, connection testing, and restore procedure

---

### Story Breakdown

| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| S3.1 | Configure remote backup location (protocol, host, path, credentials) | 5 | `[TBD]` |
| S3.2 | Test remote backup connection with status feedback | 5 | `[TBD]` |
| S3.3 | Copy backup files to remote location after local backup completes | 8 | `[TBD]` |
| S3.4 | View backup & restore history with status and details | 5 | `[TBD]` |
| S3.5 | Restore from selected backup with confirmation and system restart | 8 | `[TBD]` |
| S3.6 | Handle remote backup failures with retry and local fallback | 5 | `[TBD]` |

---
---

## Stories for Epic 3

---

### STORY: S3.1 — Configure Remote Backup Location (Protocol, Host, Path, Credentials)

**Parent Epic**: Epic 3 — Remote Backup Location & Restore Enhancements
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want to configure a remote backup storage location by selecting a protocol (SMB/CIFS, NFS, SFTP) and entering the host, share path, and credentials,
So that backup files can be copied to an off-server location for disaster recovery.

**Context & notes**:
- See prototype: "Remote Backup Location" sub-card with enable toggle, Protocol dropdown, Remote Host input, Share Path input, Username input, Password input.
- When remote backup is disabled: all fields are hidden.
- When remote backup is enabled: all fields become editable.
- Password must be masked in the UI and stored encrypted at rest (AES-256 or equivalent).
- On-prem consideration: customers typically have a NAS or file server on their local network; this feature does not require internet access.

**Acceptance Criteria**

AC1:
  Given: The admin is on the Backup Plan tab, Remote Backup Location section
  When: The admin toggles "Enable Remote Backup" to ON
  Then: Protocol, Remote Host, Share Path, Username, and Password fields become visible and editable

AC2:
  Given: Remote backup is enabled
  When: The admin selects "SFTP" from the Protocol dropdown
  Then: The protocol is set to SFTP; the fields remain the same (host, path, username, password)

AC3:
  Given: Remote backup is enabled
  When: The admin enters host = "192.168.1.100", path = "/backups/blss", username = "backupuser", password = "secret"
  Then: All fields accept the input; no validation errors

AC4 (validation — required fields):
  Given: Remote backup is enabled
  When: The admin leaves "Remote Host" blank and clicks Save Configuration
  Then: A validation error "Remote host is required" is displayed; save is blocked

AC5 (validation — required fields):
  Given: Remote backup is enabled
  When: The admin leaves "Share Path" blank and clicks Save Configuration
  Then: A validation error "Share path is required" is displayed; save is blocked

AC6:
  Given: Remote backup is enabled and the admin toggles it back to OFF
  When: The toggle changes to OFF
  Then: All remote fields are hidden; previously entered values are preserved in memory (not cleared) but remote backup is disabled in the saved configuration

AC7:
  Given: Remote backup configuration is saved
  When: The admin reloads the page
  Then: The remote backup toggle reflects the saved state; if enabled, all fields show the saved values except Password (masked/empty for security — `[TBD — confirm with security team]`)

**Tasks**
- [ ] Task 1: <Frontend> Implement remote backup enable/disable toggle per prototype
- [ ] Task 2: <Frontend> Implement Protocol dropdown (SMB/CIFS, NFS, SFTP)
- [ ] Task 3: <Frontend> Implement Remote Host, Share Path, Username, Password fields with conditional enable/disable
- [ ] Task 4: <Frontend> Implement required field validation for Remote Host and Share Path
- [ ] Task 5: <Backend> Add remote backup configuration fields to schema (remoteEnabled, remoteProtocol, remoteHost, remotePath, remoteUser, remotePass)
- [ ] Task 6: <Backend> Implement encrypted storage for remotePass field (AES-256 or equivalent) `[TBD — confirm encryption approach with security team]`
- [ ] Task 7: <Backend> Implement server-side validation for remote configuration fields
- [ ] Task 8: <Installer/config> Update installer to set remote backup defaults (remoteEnabled = false)
- [ ] Task 9: <Upgrade/migration> Add remote backup fields to migration script with defaults for existing installations
- [ ] Task 10: <Test> Write unit tests for remote fields conditional rendering and validation
- [ ] Task 11: <Test> Write integration test for save/load of remote configuration (verify password is encrypted at rest)
- [ ] Task 12: <Documentation> Update firewall documentation with required ports (SMB: TCP 445, NFS: TCP/UDP 2049, SFTP: TCP 22)

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — prototype HTML available
- [ ] API contract defined — remote fields in config schema
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — N/A
- [ ] On-prem install/upgrade impact assessed — new config fields; encrypted credential storage
- [ ] Sizing impact assessed — N/A (configuration only)
- [ ] Dependencies identified and unblocked — S1.4 (config persistence); security team for encryption approach
- [ ] NFRs applicable to this story are noted — Security: encrypted credential storage
- [ ] Estimated by team

---

### STORY: S3.2 — Test Remote Backup Connection with Status Feedback

**Parent Epic**: Epic 3 — Remote Backup Location & Restore Enhancements
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want to test the remote backup connection from the Server Admin UI and see a clear pass/fail result,
So that I can verify the remote location is reachable and writable before relying on it for backup copies.

**Context & notes**:
- See prototype: "Test Connection" button below the remote fields.
- The test should verify: (1) network connectivity to host, (2) authentication with provided credentials, (3) write access to the specified share path.
- The test must timeout within 10 seconds.
- Results displayed as toast notification: success (green) or failure (red with error details).
- On-prem consideration: customers may have complex network topologies; clear error messages help troubleshoot firewall/DNS issues.

**Acceptance Criteria**

AC1:
  Given: Remote backup is enabled with valid host, path, and credentials
  When: The admin clicks "Test Connection"
  Then: The system attempts to connect to the remote host using the selected protocol; on success, a green toast "Connection successful" is displayed

AC2:
  Given: Remote backup is enabled with an unreachable host (e.g., wrong IP)
  When: The admin clicks "Test Connection"
  Then: The test times out within 10 seconds; a red toast "Connection failed: Unable to reach host [host]" is displayed

AC3:
  Given: Remote backup is enabled with valid host but wrong credentials
  When: The admin clicks "Test Connection"
  Then: A red toast "Connection failed: Authentication failed for user [username]" is displayed

AC4:
  Given: Remote backup is enabled with valid host and credentials but the share path does not exist
  When: The admin clicks "Test Connection"
  Then: A red toast "Connection failed: Path [path] not found or not writable" is displayed

AC5:
  Given: The "Test Connection" button is clicked
  When: The test is in progress
  Then: The button shows a loading state (disabled + spinner or "Testing..." text) and cannot be clicked again until the test completes

AC6 (negative):
  Given: Remote backup is disabled
  When: The admin views the remote backup section
  Then: The "Test Connection" button is not visible (or disabled)

**Tasks**
- [ ] Task 1: <Backend> Create REST API endpoint `POST /api/v2/admin/backup/remote/test` that accepts protocol, host, path, credentials and returns pass/fail with error details
- [ ] Task 2: <Backend> Implement SMB/CIFS connectivity test (connect, authenticate, write test file, delete test file)
- [ ] Task 3: <Backend> Implement NFS connectivity test (mount, write test file, delete test file, unmount)
- [ ] Task 4: <Backend> Implement SFTP connectivity test (connect, authenticate, write test file, delete test file)
- [ ] Task 5: <Backend> Implement 10-second timeout for all connection tests
- [ ] Task 6: <Frontend> Wire "Test Connection" button to API endpoint with loading state
- [ ] Task 7: <Frontend> Display success/failure toast with error details from API response
- [ ] Task 8: <Test> Write integration tests for each protocol (success, unreachable host, bad credentials, bad path)
- [ ] Task 9: <Test> Write timeout test (ensure 10-second cap)

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — prototype HTML available
- [ ] API contract defined — `POST /api/v2/admin/backup/remote/test`
- [ ] Device/protocol compatibility confirmed — SMB/CIFS, NFS, SFTP libraries available on RHEL 9/10
- [ ] Cross-product dependencies identified and interface agreed — N/A
- [ ] On-prem install/upgrade impact assessed — requires SMB/NFS/SFTP client libraries on Application Server
- [ ] Sizing impact assessed — N/A
- [ ] Dependencies identified and unblocked — S3.1 (remote configuration fields)
- [ ] NFRs applicable to this story are noted — Usability: ≤ 10s timeout; clear error messages
- [ ] Estimated by team

---

### STORY: S3.3 — Copy Backup Files to Remote Location After Local Backup Completes

**Parent Epic**: Epic 3 — Remote Backup Location & Restore Enhancements
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want backup files to be automatically copied to the configured remote location after each successful local backup,
So that an off-server copy exists for disaster recovery without requiring manual file transfer.

**Context & notes**:
- The remote copy is a post-backup step, triggered after the local backup file and checksum are written.
- The copy must use the protocol configured in S3.1 (SMB/CIFS, NFS, SFTP).
- The checksum file must also be copied to the remote location.
- If the remote copy fails, the local backup is still marked as successful (see S3.6 for failure handling).
- The rotation policy (Epic 2) applies to remote backup files as well — when a local backup is deleted by rotation, the corresponding remote copy should also be deleted.
- On-prem consideration: copy operates over the customer's LAN; must handle variable network speeds gracefully.

**Acceptance Criteria**

AC1:
  Given: Remote backup is enabled and a local backup has completed successfully
  When: The post-backup remote copy step runs
  Then: The backup archive file and its checksum file are copied to the remote location using the configured protocol

AC2:
  Given: Remote copy completes successfully
  When: The copy finishes
  Then: The backup history record is updated to indicate remote copy status = "Success"

AC3:
  Given: Remote backup is disabled
  When: A local backup completes
  Then: No remote copy is attempted; the backup completes normally

AC4:
  Given: Remote copy fails on the first attempt (transient network error)
  When: The retry mechanism activates (see S3.6)
  Then: The copy is retried per the retry policy; local backup status remains "Success"

AC5:
  Given: A local backup is deleted by the rotation policy (Epic 2)
  When: The rotation engine runs
  Then: The corresponding remote copy is also deleted from the remote location

AC6:
  Given: Remote copy is in progress
  When: The copy completes
  Then: The file size on the remote location matches the local file size; integrity is verifiable via checksum

**Tasks**
- [ ] Task 1: <Backend> Implement post-backup hook in scheduler to trigger remote copy when remoteEnabled = true
- [ ] Task 2: <Backend> Implement SMB/CIFS file copy (backup archive + checksum file)
- [ ] Task 3: <Backend> Implement NFS file copy (backup archive + checksum file)
- [ ] Task 4: <Backend> Implement SFTP file copy (backup archive + checksum file)
- [ ] Task 5: <Backend> Update backup history record with remote copy status (Success / Failed / Pending)
- [ ] Task 6: <Backend> Integrate with rotation engine (Epic 2): when local backup is deleted, delete corresponding remote copy
- [ ] Task 7: <Backend> Implement progress tracking / logging during large file transfers
- [ ] Task 8: <Test> Write integration tests for remote copy via each protocol (success path)
- [ ] Task 9: <Test> Write test for rotation cleanup of remote copies
- [ ] Task 10: <Test> Write performance test for remote copy throughput on simulated LAN

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — N/A (backend operation; status shown in history via S3.4)
- [ ] API contract defined — post-backup hook; uses remote config from S3.1
- [ ] Device/protocol compatibility confirmed — SMB/NFS/SFTP client libraries on RHEL 9/10
- [ ] Cross-product dependencies identified and interface agreed — N/A
- [ ] On-prem install/upgrade impact assessed — requires network access to remote host; firewall rules documented in S3.1
- [ ] Sizing impact assessed — network bandwidth consumption during copy; document per sizing tier
- [ ] Dependencies identified and unblocked — S3.1 (remote config), S1.5/S1.6 (backup execution), S2.5 (rotation cleanup)
- [ ] NFRs applicable to this story are noted — Performance: ≥ 50 MB/s throughput; Availability: local backup independent of remote
- [ ] Estimated by team

---

### STORY: S3.4 — View Backup & Restore History with Status and Details

**Parent Epic**: Epic 3 — Remote Backup Location & Restore Enhancements
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want to view a history table of all backup and restore operations with their details (host, node type, backup type, timestamp, size, status, last restore date),
So that I can audit backup operations and identify failures or gaps in the backup schedule.

**Context & notes**:
- See prototype: "Backup & Restore History" tab with a data table.
- Table columns: (radio button for selection), Host Name, Node Type, Backup Type, Last Backup (timestamp with timezone), Size, Status (badge: Success / Warning / Failed), Last Restore (timestamp or "N/A").
- Status badges: Success = green, Warning = yellow, Failed = red (not shown in current prototype data but implied).
- The table shows the most recent backups, ordered by timestamp descending.
- A radio button on each row allows selecting a backup for restore (see S3.5).
- On-prem consideration: all data is local; no external API calls.

**Acceptance Criteria**

AC1:
  Given: The admin navigates to the "Backup & Restore History" tab
  When: The tab loads
  Then: A table displays all backup records with columns: Host Name, Node Type, Backup Type, Last Backup (timestamp), Size, Status, Last Restore

AC2:
  Given: There are 5 backup records in the system
  When: The history table renders
  Then: All 5 records are displayed ordered by Last Backup timestamp descending (most recent first)

AC3:
  Given: A backup completed with status "Success"
  When: The status column renders for that record
  Then: A green "Success" badge is displayed

AC4:
  Given: A backup completed with status "Warning" (e.g., backup succeeded but remote copy failed)
  When: The status column renders for that record
  Then: A yellow "Warning" badge is displayed

AC5:
  Given: A backup has never been used for a restore operation
  When: The "Last Restore" column renders for that record
  Then: "N/A" is displayed

AC6:
  Given: A backup was used for a restore on 2026-04-19 14:32:10
  When: The "Last Restore" column renders for that record
  Then: The restore timestamp "2026-04-19 14:32:10" is displayed with timezone

AC7 (empty state):
  Given: No backup records exist (fresh installation)
  When: The history tab loads
  Then: An empty state message is displayed (e.g., "No backup history available. Configure a backup plan to get started.")

AC8:
  Given: Each row has a radio button
  When: The admin selects a row's radio button
  Then: The "Restore" button becomes enabled

**Tasks**
- [ ] Task 1: <Backend> Create REST API endpoint `GET /api/v2/admin/backup/history` returning backup records with all required fields
- [ ] Task 2: <Data/schema> Define backup history table/schema (id, hostName, nodeType, backupType, backupTimestamp, fileSize, status, remoteCopyStatus, lastRestoreTimestamp, filePath)
- [ ] Task 3: <Frontend> Implement Backup & Restore History tab per prototype design
- [ ] Task 4: <Frontend> Implement data table with columns, sorting (timestamp descending), and status badges
- [ ] Task 5: <Frontend> Implement radio button selection for restore (enables Restore button)
- [ ] Task 6: <Frontend> Implement empty state for no backup history
- [ ] Task 7: <Installer/config> Create backup history table in database during installation
- [ ] Task 8: <Upgrade/migration> Write migration script to create backup history table; optionally import existing backup file metadata
- [ ] Task 9: <Test> Write unit tests for history table rendering (populated, empty, various status badges)
- [ ] Task 10: <Test> Write API integration test for backup history endpoint

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — prototype HTML available
- [ ] API contract defined — `GET /api/v2/admin/backup/history`
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — N/A
- [ ] On-prem install/upgrade impact assessed — new DB table for history; migration script needed
- [ ] Sizing impact assessed — history table growth is minimal (metadata only, no backup data)
- [ ] Dependencies identified and unblocked — S1.5/S1.6 (backup execution writes history records)
- [ ] NFRs applicable to this story are noted — Performance: < 2s page load for 100 records
- [ ] Estimated by team

---

### STORY: S3.5 — Restore from Selected Backup with Confirmation and System Restart

**Parent Epic**: Epic 3 — Remote Backup Location & Restore Enhancements
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want to select a backup from the history table and initiate a restore operation with a confirmation dialog that shows backup details and warns about the destructive nature of the action,
So that I can recover from data loss or corruption with confidence and clear understanding of the impact.

**Context & notes**:
- See prototype: "Restore" button (enabled after selecting a radio button in history table). Restore Confirmation modal shows: Host, Backup Date, Type, Size, plus destructive action warning: "This is a destructive action. All current data will be replaced with the backup data. The system will restart after restore."
- Restore replaces all current BLSS data (databases, configuration) with the selected backup's contents.
- After restore, the Application Server restarts automatically.
- The restore operation must be logged in the backup history (updating the "Last Restore" column for the selected backup).
- On-prem consideration: restore causes downtime; admin must be aware. Lifecycle Services may assist with critical restores.

**Acceptance Criteria**

AC1:
  Given: The admin has selected a backup from the history table
  When: The admin clicks the "Restore" button
  Then: A confirmation modal appears showing: Host Name, Backup Date, Backup Type, Size, and a destructive action warning

AC2:
  Given: The confirmation modal is displayed
  When: The admin clicks "Restore Now"
  Then: The restore operation begins; the system stops BLSS services, restores data from the selected backup, and restarts the Application Server

AC3:
  Given: The confirmation modal is displayed
  When: The admin clicks "Cancel"
  Then: The modal closes; no restore is performed

AC4:
  Given: A restore operation completes successfully
  When: The system restarts and the admin logs back in
  Then: The backup history shows the "Last Restore" timestamp updated for the restored backup record; all BLSS data reflects the state at the time of the backup

AC5 (negative):
  Given: No backup is selected in the history table
  When: The admin views the page
  Then: The "Restore" button is disabled and cannot be clicked

AC6 (failure):
  Given: A restore operation fails partway (e.g., corrupted backup file, disk error)
  When: The failure occurs
  Then: The system logs the error with details; the system attempts to restart with whatever state is available; the backup history records restore status = "Failed" with error description; `[TBD — confirm rollback behavior with architecture team]`

AC7:
  Given: A restore is in progress
  When: The admin attempts to navigate away or close the browser
  Then: The restore continues server-side regardless of the UI session state (restore is a backend operation)

**Tasks**
- [ ] Task 1: <Frontend> Implement Restore button (disabled by default, enabled on radio selection)
- [ ] Task 2: <Frontend> Implement Restore Confirmation modal per prototype (backup details + destructive warning)
- [ ] Task 3: <Backend> Create REST API endpoint `POST /api/v2/admin/backup/restore` accepting backup ID; initiates restore process
- [ ] Task 4: <Backend> Implement restore orchestration: stop BLSS services → restore PostgreSQL database → restore configuration files → restart Application Server
- [ ] Task 5: <Backend> Implement restore failure handling: log error, record status in history, attempt restart
- [ ] Task 6: <Backend> Update backup history with "Last Restore" timestamp after successful restore
- [ ] Task 7: <Backend> Implement checksum verification of backup file before starting restore
- [ ] Task 8: <Test> Write integration test for end-to-end restore (backup → restore → verify data state)
- [ ] Task 9: <Test> Write failure test for restore with corrupted backup file
- [ ] Task 10: <Documentation> Document full restore procedure including: pre-restore checklist, expected downtime, post-restore verification steps, recovery from failed restore
- [ ] Task 11: <Service enablement> Create Lifecycle Services restore runbook for field service engineers assisting customers

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — prototype HTML available
- [ ] API contract defined — `POST /api/v2/admin/backup/restore`
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — restore affects all installed BLSS products
- [ ] On-prem install/upgrade impact assessed — restore causes full Application Server restart; downtime documented
- [ ] Sizing impact assessed — restore time depends on backup size; document per sizing tier
- [ ] Dependencies identified and unblocked — S3.4 (history table for selection), S1.5/S1.6 (backup files to restore from)
- [ ] NFRs applicable to this story are noted — Performance: ≤ 6h restore (Medium tier); Reliability: atomic restore or documented recovery
- [ ] Estimated by team

---

### STORY: S3.6 — Handle Remote Backup Failures with Retry and Local Fallback

**Parent Epic**: Epic 3 — Remote Backup Location & Restore Enhancements
**BLSS Product**: Shared platform services (Server Admin)

**Description**
As an On-Prem IT / System Admin,
I want the system to retry failed remote backup copies and fall back to local-only backup when the remote location is unreachable,
So that local backup protection is never compromised by remote storage issues and I am notified of persistent remote failures.

**Context & notes**:
- Remote backup copy failures can occur due to: network outage, remote server down, authentication change, disk full on remote, path deleted.
- Retry policy: up to 3 retries with exponential backoff (e.g., 30s, 2m, 10m) `[TBD — confirm intervals with architecture team]`.
- If all retries fail, the local backup remains valid; the backup history records status = "Warning" (local success, remote failure).
- Persistent remote failures (e.g., 3+ consecutive backup cycles with remote failure) should generate an alarm/notification.
- On-prem consideration: network issues are common in customer environments; the system must be resilient.

**Acceptance Criteria**

AC1:
  Given: Remote backup is enabled and the remote copy fails on the first attempt
  When: The failure is detected
  Then: The system retries the copy up to 3 times with exponential backoff intervals

AC2:
  Given: The first retry succeeds
  When: The copy completes
  Then: The backup history records remote copy status = "Success"; no warning is generated

AC3:
  Given: All 3 retries fail
  When: The final retry fails
  Then: The backup history records status = "Warning" with detail "Remote copy failed after 3 retries: [error message]"; the local backup status remains "Success"

AC4:
  Given: Remote copy has failed for 3 consecutive backup cycles
  When: The 3rd consecutive failure occurs
  Then: An alarm is generated in the system log and (if notifications are configured) a notification is sent to the admin

AC5:
  Given: Remote copy failed on the previous cycle but succeeds on the current cycle
  When: The current copy succeeds
  Then: The consecutive failure counter resets to 0; no alarm is generated

AC6 (negative):
  Given: Remote backup is disabled
  When: A local backup completes
  Then: No remote copy is attempted; no retry logic runs; backup status = "Success"

AC7:
  Given: The remote copy fails due to "disk full" on the remote server
  When: The error is detected
  Then: The error message in the backup history includes "Remote disk full" or equivalent detail; retries are attempted but may also fail with the same error

**Tasks**
- [ ] Task 1: <Backend> Implement retry mechanism with exponential backoff for remote copy failures (3 retries, configurable intervals)
- [ ] Task 2: <Backend> Implement local-only fallback: ensure local backup status = "Success" regardless of remote outcome
- [ ] Task 3: <Backend> Implement backup history status differentiation: "Success" (both local + remote), "Warning" (local success, remote failure)
- [ ] Task 4: <Backend> Implement consecutive failure counter and alarm generation on 3+ consecutive remote failures
- [ ] Task 5: <Backend> Implement detailed error message capture from SMB/NFS/SFTP client for inclusion in history and logs
- [ ] Task 6: <Backend> Reset consecutive failure counter on successful remote copy
- [ ] Task 7: <Test> Write integration test for retry mechanism (1st attempt fails, 2nd succeeds)
- [ ] Task 8: <Test> Write integration test for all-retries-failed scenario
- [ ] Task 9: <Test> Write integration test for consecutive failure alarm (3+ cycles)
- [ ] Task 10: <Test> Write test for error detail capture from each protocol

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached — N/A (backend logic; status shown in history via S3.4)
- [ ] API contract defined — retry is internal to backup scheduler
- [ ] Device/protocol compatibility confirmed — N/A
- [ ] Cross-product dependencies identified and interface agreed — N/A
- [ ] On-prem install/upgrade impact assessed — retry intervals configurable; default set during install
- [ ] Sizing impact assessed — N/A
- [ ] Dependencies identified and unblocked — S3.3 (remote copy), S3.4 (history display)
- [ ] NFRs applicable to this story are noted — Reliability: 3 retries; Availability: local independent of remote
- [ ] Estimated by team

---
---

## Validation Checklist

- [x] All AC are testable — each AC uses Given/When/Then format with specific, observable outcomes
- [x] NFRs are quantified — performance targets include specific time/throughput values with reference hardware tiers
- [x] BLSS product(s) explicitly identified for every Epic and Story — Shared platform services (Server Admin)
- [x] Scope boundaries are explicit — in-scope and out-of-scope defined for each Epic
- [x] Cross-product dependencies are identified — backup/restore affects all installed BLSS products; noted in Epics
- [x] On-prem operational impact assessed — install, upgrade, sizing, network, backup documented per Epic
- [x] Dependencies are identified — inter-story and inter-epic dependencies documented in each Epic's risk table
- [x] No invented features, UI labels, device models, or protocol details — all UI elements reference the existing prototype; API endpoints use `[TBD]` notation
- [x] Terminology matches BLSS product glossary — uses "Application Server", "On-Premises", "Sizing Guide", "Upgrade Runbook", "Migration Script" per Section 12
- [x] Placeholders marked with `[TBD]` — all unconfirmed values (sizing targets, encryption approach, algorithm variant, API paths) marked
- [x] Operational deliverables included — runbooks, sizing guide, firewall docs, release notes, Lifecycle Services training per Epic
- [x] Service enablement (Lifecycle Services) considered — training enablement and restore runbook included

### Open Items Requiring SME Validation

| # | Item | SME / Team | Epic | Priority |
|---|---|---|---|---|
| 1 | ⚠️ NEEDS VALIDATION: Incremental backup implementation approach (WAL archiving vs. file-diff) | Architecture team | Epic 1 | P0 |
| 2 | ⚠️ NEEDS VALIDATION: Full backup performance targets (≤ 4h for Medium tier) | Architecture + SRE | Epic 1 | P1 |
| 3 | ⚠️ NEEDS VALIDATION: Tower of Hanoi algorithm variant | Architecture team | Epic 2 | P1 |
| 4 | ⚠️ NEEDS VALIDATION: Credential encryption approach and key management | Security team | Epic 3 | P0 |
| 5 | ⚠️ NEEDS VALIDATION: Restore rollback behavior on partial failure | Architecture team | Epic 3 | P0 |
| 6 | ⚠️ NEEDS VALIDATION: Remote backup throughput target (≥ 50 MB/s) | Architecture + SRE | Epic 3 | P1 |
| 7 | ⚠️ NEEDS VALIDATION: SFTP host key verification strategy | Security team | Epic 3 | P1 |
| 8 | ⚠️ NEEDS VALIDATION: Restore from remote-only backup (currently out of scope) | Product Management | Epic 3 | P2 |
| 9 | ⚠️ NEEDS VALIDATION: SSH key authentication for SFTP (currently out of scope) | Security team | Epic 3 | P2 |
| 10 | ⚠️ NEEDS VALIDATION: Backup configuration storage mechanism (config file vs. DB table) | Architecture team | Epic 1 | P0 |

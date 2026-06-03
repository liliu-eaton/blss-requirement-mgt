# Epic & Story Templates

## Epic template (required output format)

For each Epic, output the following structure:

```
### EPIC: [EPIC-ID] <Epic Title>

**BLSS Product(s)**: <Which product(s) this Epic belongs to>
**Summary** (1-2 sentences)
<What is being delivered and why it matters to the user/business>

**Description**
- **Problem statement**: <What pain point or opportunity does this address?>
- **Proposed solution**: <High-level approach>
- **User personas**: <Who benefits?>
- **Business value**: <Quantified if possible>
- **Key workflows**: <End-to-end flow summary>
- **Cross-product touchpoints**: <Does this Epic interact with other BLSS products? How?>
- **UX/UI references**: <Link to mockups, Brightlayer Design System components>
- **On-prem operational impact**:
  - Install/upgrade impact: <New service, schema change, config change?>
  - Sizing impact: <Additional CPU/RAM/disk requirements?>
  - Network impact: <New ports, protocols, firewall rules?>
  - Backup impact: <New data to back up? Schema migration?>

**Non-Functional Requirements (NFRs)**
| Category | Requirement | Target | Priority |
|---|---|---|---|
| ... | ... | ... | ... |

**Acceptance Criteria (Epic-level)**
- [ ] AC1: <Measurable, testable criterion>
- [ ] AC2: ...
- [ ] AC3: ...

**Scope**
- **In scope**: <Bulleted list>
- **Out of scope**: <Bulleted list>
- **Assumptions**: <Numbered list; mark with ⚠️ if unvalidated>

**Dependencies & Risks**
| Item | Type (Dep/Risk) | BLSS Product | Owner | Mitigation | Status |
|---|---|---|---|---|---|
| ... | ... | ... | ... | ... | ... |

**Operational deliverables** (required for on-prem)
- [ ] Installation/upgrade runbook updated
- [ ] Sizing guide updated (if resource requirements change)
- [ ] Backup/restore procedure updated (if schema changes)
- [ ] Firewall/port documentation updated (if new network requirements)
- [ ] Release notes entry drafted
- [ ] Lifecycle Services training/enablement updated

**Story Breakdown** (summary list — details below)
| Story ID | Story Title | Points (est.) | Sprint Target |
|---|---|---|---|
| ... | ... | ... | ... |
```

---

## User Story template (required output format)

For each Story, output the following structure:

```
### STORY: [STORY-ID] <Story Title>

**Parent Epic**: [EPIC-ID] <Epic Title>
**BLSS Product**: <Product this story is implemented in>

**Description**
As a [persona/role],
I want to [action/capability],
So that [business value / outcome].

**Context & notes**:
- <Additional context, design decisions, edge cases>
- <Link to mockup / wireframe if UI story>
- <Link to API spec if integration story>
- <Device / protocol considerations if hardware-related>
- <Cross-product data flow if applicable>
- <On-prem considerations: install, config, sizing impact>

**Acceptance Criteria**
Given [precondition],
When [action],
Then [expected result].

AC1:
  Given: <precondition>
  When: <user action or system event>
  Then: <observable outcome>

AC2:
  Given: ...
  When: ...
  Then: ...

AC3: (negative / edge case)
  Given: ...
  When: ...
  Then: ...

**Tasks** (technical decomposition for dev team)
- [ ] Task 1: <Backend> (e.g., "Create REST endpoint GET /api/v2/pue/sites/{siteId}/realtime")
- [ ] Task 2: <Data/schema> (e.g., "Add pue_reading table migration with rollback script")
- [ ] Task 3: <Frontend> (e.g., "Implement PUE gauge component per Brightlayer Design System")
- [ ] Task 4: <Device/protocol> (e.g., "Implement Modbus register mapping for Eaton meter model X")
- [ ] Task 5: <Integration> (e.g., "Consume device inventory API from Brightlayer Power service")
- [ ] Task 6: <Test> (e.g., "Write integration tests for PUE calculation service")
- [ ] Task 7: <Installer/config> (e.g., "Update installer to create new DB table; add config entry to settings.xml")
- [ ] Task 8: <Upgrade/migration> (e.g., "Write DB migration script from v25.1 schema to v25.2; test rollback")
- [ ] Task 9: <Documentation> (e.g., "Update API docs for /pue endpoint; update install guide")
- [ ] Task 10: <Service enablement> (e.g., "Create Lifecycle Services deployment runbook for this feature")

**Definition of Ready checklist**
- [ ] AC are reviewed and agreed by PO + dev team
- [ ] UX mockups attached (if UI-impacting)
- [ ] API contract defined (if integration)
- [ ] Device/protocol compatibility confirmed (if hardware-related)
- [ ] Cross-product dependencies identified and interface agreed
- [ ] On-prem install/upgrade impact assessed
- [ ] Sizing impact assessed (if resource requirements change)
- [ ] Dependencies identified and unblocked
- [ ] NFRs applicable to this story are noted
- [ ] Estimated by team
```

---

## Cross-product requirement patterns

When requirements span multiple BLSS products, use these patterns:

### Pattern A: Data producer → consumer
```
Product A produces data → internal API / shared DB → Product B consumes
Example: Brightlayer Power (device inventory) → internal API → DCPM (asset database)
```
**Required in Epic**: Define API contract or DB schema contract, data ownership, sync frequency, failure handling, and impact on install/upgrade order.

### Pattern B: Unified user experience
```
User sees consolidated view across products in a single on-prem dashboard
Example: NOC dashboard shows DCPM alarms + EPMS power events + Distributed IT site status
```
**Required in Epic**: Define which product owns the UI, data aggregation mechanism, latency requirements, and whether all products must be installed or if graceful degradation is supported.

### Pattern C: Service enablement
```
Software feature requires Lifecycle Services to deploy/configure for customer
Example: EPMS meter commissioning requires on-site Eaton service engineer
```
**Required in Epic**: Define self-service vs. service-assisted boundary, runbook/documentation deliverables, training materials.

### Pattern D: Coordinated upgrade
```
Cross-product feature requires specific upgrade order or version alignment
Example: DCPM 26.1 requires Brightlayer Power 26.1+ for new device inventory API
```
**Required in Epic**: Define version compatibility matrix, upgrade sequence, rollback procedure if partial upgrade fails.

---

## Example usage

**Input from stakeholder:**
> "We need facility managers to see real-time PUE across all sites, and also correlate PUE spikes with power quality events from EPMS. Also need the Lifecycle Services team to be able to configure PUE thresholds during initial deployment. Target Q3."

**Expected PO output sequence:**
1. Identify products: **DCPM** (PUE dashboard), **EPMS** (power event data), **Lifecycle Services** (deployment config) → Cross-product initiative
2. Assess on-prem impact: schema changes, inter-product API, install order, sizing
3. Clarifying questions (gaps in required sections)
4. Assumptions list
5. Epic outline with proposed Story titles, tagged by product
6. Full Epic (template format) after confirmation
7. Full Stories (template format) for each Story
8. Validation checklist
9. Write confirmed output to `Requirements/<JIRA_ISSUE_KEY>/`:
   - `<JIRA_ISSUE_KEY>-Epic_1.md`, `<JIRA_ISSUE_KEY>-Epic_2.md`, ... (one file per Epic with its child Stories)

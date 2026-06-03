# BLSS Domain Context

## Product domain: Eaton Brightlayer Software Suite (BLSS)

BLSS is Eaton's **on-premises integrated software suite** for data center and critical power infrastructure management. It is a **portfolio of products and services** deployed **within the customer's own data center environment** — there is no SaaS or cloud-hosted deployment model.

## Deployment model (non-negotiable constraint)

> **BLSS is exclusively on-premises software.**
> - All components run on customer-owned or customer-managed infrastructure.
> - No data leaves the customer's network unless explicitly configured for outbound integrations.
> - All architecture, NFR, security, and operational decisions must respect on-prem constraints: customer-managed servers, networking, backup, patching, and physical access.
> - Upgrade and maintenance are performed on-site or via customer-initiated remote sessions, supported by Lifecycle Management Services.

## BLSS Suite composition

| # | Product / Module | Key Capabilities | Target Users |
|---|---|---|---|
| 1 | **Data Center Performance Management (DCPM)** | Real-time power & environmental monitoring, capacity planning (power, cooling, space), asset management, alarming & event management, dashboards & reporting, change management, multi-site management | Facility managers, operations engineers, energy managers |
| 2 | **Electrical Power Monitoring System — EPMS Standard** | Power quality monitoring, energy metering & sub-metering, demand analysis, power event recording & waveform capture, electrical system one-line visualization, compliance reporting (IEEE, IEC), cost allocation | Electrical engineers, facility managers, energy managers, compliance officers |
| 3 | **Distributed IT Performance Management** | Monitoring and management of **edge / distributed IT** infrastructure (micro data centers, network closets, remote sites), environmental monitoring, remote device management, consolidated multi-site visibility for geographically dispersed assets | IT infrastructure admins, edge/remote site operators, managed service providers |
| 4 | **Brightlayer Power** | Power systems monitoring & analytics, UPS / PDU / switchgear fleet management, predictive insights for power equipment, firmware & configuration management for Eaton power devices | Power systems engineers, facility managers, service teams, energy managers |
| 5 | **Lifecycle Management Services** | Professional services for deployment, commissioning, and configuration; ongoing maintenance & support contracts; health assessments & optimization; training & knowledge transfer; upgrade planning and execution | All BLSS users (delivered by Eaton services team & partners) |

> ⚠️ ASSUMPTION: The above capability breakdown is inferred from URL paths, Eaton's public product naming conventions, and general DCIM/power industry domain knowledge. **All descriptions must be validated against official Eaton product datasheets and marketing-approved content before use.**

## Suite architecture (on-premises)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Customer Data Center Environment                   │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │             BLSS Application Server(s)                        │  │
│  │  ┌──────────┬──────────┬──────────────┬──────────────────┐   │  │
│  │  │  DCPM    │  EPMS    │ Distributed  │ Brightlayer      │   │  │
│  │  │          │ Standard │ IT Perf Mgmt │ Power            │   │  │
│  │  │ Facility │ Power    │ Edge &       │ Power device     │   │  │
│  │  │ ops &    │ quality  │ remote IT    │ fleet monitoring │   │  │
│  │  │ capacity │ & energy │ sites        │ & analytics      │   │  │
│  │  └────┬─────┴────┬─────┴──────┬───────┴────────┬─────────┘   │  │
│  │       │          │            │                │              │  │
│  │  ┌────┴──────────┴────────────┴────────────────┴──────────┐  │  │
│  │  │          Shared Services Layer                          │  │  │
│  │  │  Database(s) · Authentication (AD/LDAP) · API Gateway   │  │  │
│  │  │  Message Bus · Logging · Backup Agent                   │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│       ┌──────────────────┼──────────────────────┐                   │
│  ┌────┴────┐        ┌────┴────┐            ┌────┴────┐             │
│  │ UPS/PDU │        │ Meters  │            │ Sensors │             │
│  │Switchgear│       │ Relays  │            │ IT Equip│             │
│  └─────────┘        └─────────┘            └─────────┘             │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │         Lifecycle Management Services (Eaton)                │   │
│  │  Deployment · Patching · Upgrades · Health Assessments       │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

> ⚠️ ASSUMPTION: The shared services layer architecture is inferred. Confirm with architecture team whether products share a common DB, auth, and messaging layer or are independently deployed.

## On-premises operational considerations

Every Epic and Story must account for these realities:

| Concern | Implication for Requirements |
|---|---|
| **Customer-managed infrastructure** | Must specify minimum hardware/OS/DB requirements; cannot assume auto-scaling |
| **Network isolation** | Features must work in air-gapped or restricted-network environments; external calls must be optional and configurable |
| **Upgrade ownership** | Customer or Lifecycle Services performs upgrades; must provide rollback procedures, migration scripts, downtime estimates |
| **Backup & recovery** | Customer manages backup; must provide backup/restore documentation, DB schema migration compatibility |
| **Authentication** | Typically integrates with customer's Active Directory / LDAP; no cloud IdP dependency |
| **Patching & security** | Customer controls OS/middleware patching cadence; application must be tested against supported OS/DB patch levels |
| **Performance sizing** | Must provide capacity sizing guidance (users, devices, data retention → CPU, RAM, disk, IOPS) |
| **Licensing** | Support on‑premise license key and entitlement management via the Unified License Gateway, with dependency on the Eaton cloud license server powered by Revenera. |
| **Offline resilience** | Must handle gateway/device communication interruptions gracefully; local data buffering |

## Target user personas (suite-wide)

| Persona | Primary Products | Key Goals |
|---|---|---|
| **Data Center Facility Manager** | DCPM, EPMS, Lifecycle Services | Uptime, capacity, compliance |
| **Operations / NOC Engineer** | DCPM, Distributed IT | Alarm response, incident resolution, monitoring |
| **Electrical / Power Systems Engineer** | EPMS Standard, Brightlayer Power | Power quality, reliability, event analysis |
| **IT Infrastructure Admin** | Distributed IT, DCPM | Remote site visibility, device health |
| **Energy / Sustainability Manager** | DCPM, EPMS | PUE, energy cost, carbon reporting |
| **Managed Service Provider (MSP)** | Distributed IT, Brightlayer Power | Multi-client fleet management |
| **Eaton Field Service Engineer** | Brightlayer Power, Lifecycle Services | Device maintenance, firmware, health assessments |
| **Executive (CIO / CFO / VP Ops)** | DCPM (dashboards/reports) | Availability KPIs, cost optimization, risk |
| **System Integrator / Partner** | All (API & integration layer) | Custom integration, deployment |
| **On-Prem IT / System Admin** | All | Server administration, backup, patching, upgrades |

## Input gathering sections

### Request type (required)

- Type: Epic definition | Story decomposition | Feature breakdown | Spike/research | NFR specification | Backlog refinement | PI Planning input | Roadmap alignment | Cross-product initiative
- **BLSS Product(s)**: DCPM | EPMS Standard | Distributed IT Perf Mgmt | Brightlayer Power | Lifecycle Services | Cross-product | Shared platform services
- Priority: P0 (Critical/PI Objective) | P1 (High) | P2 (Medium) | P3 (Low/Nice-to-have)
- Target PI / Sprint:
- Request status: New | Refinement | Re-scoping | Split | Deprecation

### Stakeholders and personas (required)

- Business sponsor / requestor:
- Primary user persona(s):
  - [ ] Data Center Facility Manager
  - [ ] Operations / NOC Engineer
  - [ ] Electrical / Power Systems Engineer
  - [ ] IT Infrastructure Admin
  - [ ] Energy / Sustainability Manager
  - [ ] Managed Service Provider (MSP)
  - [ ] Eaton Field Service Engineer
  - [ ] Executive (CIO / CFO / VP Ops)
  - [ ] System Integrator / Partner
  - [ ] On-Prem IT / System Admin
  - [ ] Other: ___
- Skill level of target user: Beginner | Intermediate | Advanced
- User intent:
  - [ ] Monitor / Observe
  - [ ] Configure / Set up
  - [ ] Analyze / Report
  - [ ] Act / Control / Remediate
  - [ ] Plan / Forecast
  - [ ] Integrate / Automate
  - [ ] Administer / Manage access
  - [ ] Install / Upgrade / Maintain (on-prem ops)
  - [ ] Procure / Request service (Lifecycle Services context)

### Product and version context (required)

- **BLSS Product(s)**: (e.g., DCPM, EPMS Standard, Distributed IT, Brightlayer Power, Lifecycle Services, Cross-product)
- Module / sub-component: (e.g., DCPM > Capacity Planning, EPMS > Power Quality, Brightlayer Power > Firmware Management)
- **Deployment model: On-Premises** (this is the only supported model)
- Edition / Tier: (e.g., Essentials, Standard, Professional, Enterprise — `[TBD if tiering varies by product]`)
- Target version / release: (e.g., DCPM 25.2, EPMS 26.1)
- Feature flags / toggles: (if gated rollout — note: on-prem flags may require config file or DB toggle)
- **Infrastructure requirements**:
  - Supported OS: (RHEL 9/10)
  - Supported database: (PostgreSQL 14+)
  - Supported browsers: (Chrome, Edge, Firefox latest 2 versions)
  - Minimum hardware: (CPU, RAM, disk — reference sizing or `[TBD]`)
  - Virtualization support: (VMware, Hyper-V, bare metal `[TBD]`)
  - Network requirements: (ports, protocols, firewall rules)
- Hardware dependencies: (e.g., specific UPS/PDU models, meter models, gateway firmware versions, sensor types)
- Protocol dependencies: (e.g., SNMP v3, BACnet IP, Modbus TCP/RTU, MQTT, REST API version)
- Authentication integration: (e.g., Active Directory, LDAP, RADIUS, OAuth2)

### Source of truth and inputs (required)

| Input Type | Available? | Link / Reference |
|---|---|---|
| Business case / initiative | Yes / No / Partial | |
| Market research / VOC | Yes / No / Partial | |
| Customer feedback / support cases | Yes / No / Partial | |
| Competitive analysis | Yes / No / Partial | |
| UX research / wireframes / mockups | Yes / No / Partial | |
| Architecture / design docs | Yes / No / Partial | |
| Existing Jira Epic(s) / tickets | Yes / No / Partial | |
| API specifications | Yes / No / Partial | |
| Protocol / device specifications | Yes / No / Partial | |
| Regulatory / compliance requirements | Yes / No / Partial | |
| Installation / sizing guide (current) | Yes / No / Partial | |
| Upgrade / migration guide (current) | Yes / No / Partial | |
| SME(s) identified for validation | Yes / No / Partial | Names/roles: |
| Existing documentation to update | Yes / No / Partial | |
| Lifecycle Services engagement model | Yes / No / Partial | |

### Strategic alignment (required)

- Product theme / strategic pillar: (e.g., "Operational Resilience", "Energy Efficiency & Sustainability", "Multi-site Scalability", "Edge-to-Core Visibility", "Power Reliability", "Service Excellence", "Ease of Deployment & Upgrade")
- Business objective / OKR link:
- Value hypothesis:
  > "We believe that [capability] in [BLSS product] for [persona] will result in [outcome] as measured by [metric]."
- Revenue / retention impact: (qualitative or quantitative)
- Competitive differentiator: Yes / No — explain
- Suite synergy: Does this feature increase the value of **other BLSS products** when used together? Explain.
- On-prem value proposition: Does this feature specifically address on-prem customer concerns (security, data sovereignty, air-gapped operation, total cost of ownership)?

### Scope and boundaries (required)

**In scope:**
- Capabilities / workflows:
- BLSS product(s) affected:
- User roles affected:
- Sites / regions:
- Device types / protocols:

**Out of scope (explicit non-goals):**
-

**Constraints:**
- Regulatory / compliance: (e.g., ISO 50001, IEEE 1159, ASHRAE, SOC 2 for customer audit, GDPR, UL/IEC certifications)
- **On-prem constraints**:
  - Must work in air-gapped / network-restricted environments: Yes / No
  - Must support offline operation for N hours/days: Yes / No — duration:
  - Maximum acceptable downtime for upgrade: (e.g., ≤ 2 hours)
  - Must be backward-compatible with previous DB schema: Yes / No — versions:
  - Must support customer-managed backup tools: Yes / No
- Technical constraints: (e.g., must support legacy Modbus devices, single-server deployment option, no container/k8s requirement)
- UX constraints: (e.g., must follow Brightlayer Design System, must support dark mode for NOC)
- Timeline constraints: (e.g., must ship in PI 25.3)
- Budget / resource constraints:
- Service constraints: (e.g., must be deployable by Lifecycle Services within standard engagement)

**Dependencies:**

| Dependency | BLSS Product | Owner | Status | Risk if delayed |
|---|---|---|---|---|
| | | | | |

### NFR framework (required for Epics)

All NFRs must be **specific and measurable**. All performance targets are measured against **reference hardware specifications** (must be defined).

**Reference hardware baseline:**

> ⚠️ Define or reference the standard sizing guide. Example:
> - **Small**: 4 vCPU, 16 GB RAM, 500 GB SSD, up to 1,000 devices
> - **Medium**: 8 vCPU, 32 GB RAM, 1 TB SSD, up to 5,000 devices
> - **Large**: 16 vCPU, 64 GB RAM, 2 TB SSD, up to 15,000 devices
>
> `[TBD — confirm actual sizing tiers with architecture team]`

**NFR table:**

| NFR Category | Requirement | Target | Priority |
|---|---|---|---|

> ⚠️ Customize for each Epic. Product-specific NFRs must be added. All values must be validated by architecture, SRE, and product teams.

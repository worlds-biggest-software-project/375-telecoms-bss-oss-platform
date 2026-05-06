# Telecoms BSS/OSS Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open, AI-native, standards-aligned BSS/OSS platform for telecommunications operators — unifying subscriber management, network inventory, and service fulfilment in a single modular stack.

Telecoms operators run two parallel software stacks — Business Support Systems (BSS) for billing, ordering, and CRM, and Operational Support Systems (OSS) for network inventory, provisioning, and assurance. This project specifies a modern, integrated platform that supports rapid service introduction, automated fulfilment, and a unified view of network and customer across the full subscriber lifecycle, aimed at MVNOs, regional carriers, and digital-first operators who are underserved by Tier-1 vendor offerings.

---

## Why Telecoms BSS/OSS Platform?

- Incumbents (Amdocs, NetCracker/Nokia, Ericsson, Huawei) are priced for Tier-1 carriers and require heavy professional-services engagements, putting them out of reach for greenfield MVNOs and smaller regional operators.
- Cloud-native challengers like Wavelo are strong on BSS but thin on OSS depth (network inventory, fault management); pure-play OSS specialists like Sunvizion are thin on billing and CRM. There is no affordable full-stack option.
- Despite TM Forum Open API standards, data-model lock-in remains pervasive — operators cannot move between vendors without major migration projects.
- Most platforms treat BSS and OSS as separate domains with separate AI models; cross-domain reasoning (linking a network fault directly to billing impact or churn risk) is absent or nascent.
- Open-source alternatives are narrow: BillRun (AGPLv3) covers BSS only with no enterprise-grade OSS and no 3GPP Nchf 5G charging; OpenSlice covers slice management only.

---

## Key Features

### BSS Core

- Product catalogue management with version control and multi-channel distribution (TMF620)
- Convergent charging engine for prepaid, postpaid, and hybrid on a single platform (3GPP Nchf-compliant)
- Order management orchestrating sales channel through network fulfilment (TMF622, TMF641)
- Subscriber and CRM module with 360-degree customer view
- Self-service portals for consumer subscribers and enterprise B2B buyers

### OSS Core

- Network inventory covering physical and logical resources with service-to-resource linking (TMF638)
- Service fulfilment and automated provisioning, including zero-touch activation
- Fault management with alarm ingestion, correlation, and trouble-ticket generation
- Performance management with KPI collection, threshold alerting, and trend reporting
- 5G network slice lifecycle management with per-slice SLA billing

### Platform & Integration

- Real-time event-driven architecture on Apache Kafka for sub-second billing and provisioning event propagation
- TM Forum Open API compliance and ODA component alignment as first-class architecture principles
- Role-based access control, multi-tenancy, and audit logging
- GDPR and telecommunications data retention compliance tooling
- Cloud-native deployment on Kubernetes across public, private, and hybrid clouds

### AI-Native Capabilities

- Anomaly detection across CDR and network event streams
- Natural-language product catalogue configuration via LLM-assisted authoring
- Automated root-cause analysis linking OSS fault events to BSS subscriber impact
- Predictive churn detection combining network quality and billing behavioural signals
- Optional Agent2Agent (A2A) multi-agent orchestration across BSS and OSS domains

---

## AI-Native Advantage

Unlike incumbents that bolt AI onto separate BSS and OSS silos, this platform is designed for cross-domain causal reasoning from day one — connecting network faults, subscriber experience, and billing outcomes in a single model. AI-augmentation candidates include intelligent per-subscriber offer personalisation in real time, natural-language catalogue configuration so business users can describe new offers in plain English, and explainable-AI surfaces for regulator-auditable bill adjustments and fraud flags.

---

## Tech Stack & Deployment

- Cloud-native, container-first design targeting Kubernetes on public, private, or hybrid cloud
- Apache Kafka event backbone for real-time mediation, rating, and provisioning
- TM Forum Open APIs (TMF620, TMF622, TMF638, TMF641 and others) and ODA component alignment
- 3GPP standards for charging (Nchf, TS 32.290 / 32.291) and ETSI NFV MANO interfaces for network function lifecycle management
- REST and GraphQL northbound APIs with a developer portal for operator engineering teams and partners

---

## Market Context

The BSS/OSS market is dominated by Amdocs, NetCracker/Nokia, Ericsson, and Huawei serving Tier-1 carriers, with cloud-native challengers (Wavelo, Cerillion, MATRIXX) and OSS specialists (Sunvizion, Comarch) competing in the mid-market. Primary buyers are MVNOs, greenfield digital operators, and regional carriers priced out of large vendor deployments and unable to assemble full-stack capability from existing open-source projects (BillRun, OpenSlice, Kuwaiba). The candidate is rated complexity 10/10 with low domain availability and low demand in the project catalogue, reflecting a heavily incumbent-defended but structurally underserved segment.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.

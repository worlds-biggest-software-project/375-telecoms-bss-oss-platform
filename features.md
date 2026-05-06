# Telecoms BSS/OSS Platform — Feature & Functionality Survey

> Candidate #375 · Researched: 2026-05-06

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Amdocs CES26 (aOS) | Commercial — large enterprise | Proprietary | https://www.amdocs.com/products-services/bss-oss/customer-experience-suite |
| NetCracker Digital BSS/OSS | Commercial — large enterprise | Proprietary | https://www.netcracker.com/portfolio/products/ |
| Ericsson Digital BSS / Telco DataOps | Commercial — large enterprise | Proprietary | https://www.ericsson.com/en/oss-bss |
| Huawei Agentic BSS / AUTINOps | Commercial — large enterprise | Proprietary | https://carrier.huawei.com/en/products/service-and-software/software-business |
| Wavelo | Commercial — cloud-native SaaS | Proprietary | https://www.wavelo.com/ |
| Cerillion Enterprise BSS/OSS | Commercial — mid-market | Proprietary | https://www.cerillion.com/ |
| MATRIXX Software | Commercial — charging specialist | Proprietary | https://www.matrixx.com/ |
| Salesforce Communications Cloud | Commercial — CRM-led BSS | Proprietary SaaS | https://www.salesforce.com/communications/cloud/ |
| Comarch BSS/OSS | Commercial — mid-market | Proprietary | https://www.comarch.com/telecommunications/ |
| Sunvizion | Commercial — OSS specialist | Proprietary | https://www.sunvizion.com/ |
| BillRun | Open source BSS | AGPLv3 | https://bill.run/ |
| OpenSlice | Open source OSS | Apache 2.0 | https://github.com/openslice |
| Kuwaiba | Open source network inventory | Apache 2.0 | https://www.kuwaiba.org/ |

---

## Feature Analysis by Solution

### Amdocs CES26 (aOS Cognitive Core)

**Core features**
- End-to-end agentic BSS-OSS-Network suite launched at MWC 2026; branded as CES26 inside the aOS platform
- Billing and revenue management including convergent charging, multi-biller aggregated invoice generation, and enterprise-scale billing
- Product catalogue and order management driving commercial and technical fulfilment across consumer and enterprise segments
- CRM and digital self-service portal with low-code / no-code configuration tooling
- AI-native OSS covering inventory, assurance, and orchestration with digital-twin-backed closed-loop automation
- Network performance management and fault assurance with predict-diagnose-recommend-resolve agentic workflows

**Differentiating features**
- aOS Cognitive Core: purpose-built AI fabric embedding domain-specific agents across every BSS and OSS domain, enabling multi-agent collaboration without custom integration
- Zero-touch operations targeting autonomous-network L4/L5 maturity via policy-driven closed loops
- Self-managed digital BSS tooling that lets operators reconfigure products and offers without raising vendor service requests
- connectX developer portal providing API access for third-party ecosystem builders

**UX patterns**
- Low-code/no-code studio for business configurators to adjust offers and pricing
- AI-assisted care agents surface billing explanations and order status without transferring to a human
- Progressive autonomy model: operators can dial back AI recommendations and approve changes before they are enacted

**Integration points**
- TMF Open API–compliant (product catalogue TMF620, product ordering TMF621, service inventory TMF638)
- 3GPP and ETSI standards alignment for network-layer interfaces
- API-first architecture with connectX developer portal (devportal.amdocs-dbs.com)

**Known gaps**
- Platform scale and licensing cost put it out of reach for greenfield MVNOs or smaller regional operators
- Heavy reliance on Amdocs professional services for initial configuration; customisation lock-in risk
- Limited public API documentation; developer portal access may require commercial agreement

**Licence / IP notes**
- Fully proprietary; no open-source components publicly disclosed. Certified compliant with TM Forum Open APIs and ODA standards.

---

### NetCracker Digital BSS/OSS

**Core features**
- Lead-to-cash automation spanning marketing & commerce, sales & customer service, and revenue management clouds
- Customer Engagement module for omnichannel journey management and personalised experience
- Advanced Analytics (AIOps) with ML-based scenarios for customer, BSS, and OSS automation
- Network inventory and resource management aligned to TM Forum ODA
- Service orchestration covering hybrid (physical + virtualised + cloud) network lifecycles
- Partner management and ecosystem billing for wholesale and digital marketplace models

**Differentiating features**
- Qubership APIHUB: open-source API registry and developer portal supporting OpenAPI, GraphQL, AsyncAPI, RESTCONF, NETCONF/YANG, and TOSCA — published to GitHub (github.com/Netcracker/qubership-apihub)
- Strong TM Forum ODA Component Directory certification, providing operators with verifiable interoperability assurance
- Three separate cloud offerings (Marketing & Commerce Cloud, Sales & Customer Service Cloud, Revenue Management Cloud) that can be adopted independently

**UX patterns**
- Modular product structure allows operators to adopt one cloud layer at a time, reducing transformation risk
- Dashboard-driven AIOps surfacing ML recommendations with human-in-the-loop override controls
- Omnichannel customer journey designer for marketing and sales teams

**Integration points**
- TM Forum Open APIs (TMF620, TMF622, TMF638, TMF641, and others)
- REST, SOAP, gRPC, GraphQL, AsyncAPI, RESTCONF, NETCONF/YANG, TOSCA supported
- ODA Component Directory Partner (certified March 2026)

**Known gaps**
- Enterprise-only pricing; smaller operators cannot afford deployment
- Deep integration with Nokia hardware stacks can create hardware-vendor dependency
- OSS capabilities are stronger than BSS for operators not already on Nokia networks

**Licence / IP notes**
- Core platform is proprietary; Qubership APIHUB is open-sourced on GitHub under a permissive licence.

---

### Ericsson Digital BSS / Telco DataOps Platform

**Core features**
- Cloud-native OSS and BSS spanning data, cloud & IT, monetisation, service orchestration, and core commerce
- Telco DataOps Platform: re-engineered mediation layer with LLM pipeline to AWS Bedrock for AI-enriched data operations
- More than 20 cloud-native AI and Gen-AI applications deployable on Ericsson's Telco IT AI Engine or partner hyperscaler platforms
- Telco Agentic AI Studio: low-code platform for building GenAI BSS/OSS applications on Amazon Bedrock
- Service orchestration covering 5G slice lifecycle management and cross-domain intent translation
- Deployment flexibility: on-premise, public cloud, or hybrid

**Differentiating features**
- First major vendor to formally reposition its mediation product as a "Telco DataOps Platform" with native LLM pipeline support
- Gen-AI Lab with AWS providing a structured co-innovation programme for CSPs
- Broad AI application catalogue (20+) spanning network, customer, and billing domains

**UX patterns**
- Developer-friendly AI Studio with drag-and-drop GenAI application builder
- Human-in-the-loop controls for agentic automation with configurable escalation thresholds
- Cloud-agnostic deployment wizard reducing infrastructure dependency

**Integration points**
- AWS Bedrock as primary AI/GenAI model provider
- TM Forum Open APIs and 3GPP standards compliance
- ETSI NFV MANO interfaces for network function lifecycle management

**Known gaps**
- AWS partnership creates de facto cloud-vendor dependency for GenAI features
- Mediation platform rebranding may obscure legacy integration complexity for existing customers
- Smaller CSPs may struggle to justify the AI Studio investment without scale

**Licence / IP notes**
- Fully proprietary. Open API compliance certified by TM Forum.

---

### Huawei Agentic BSS / AUTINOps

**Core features**
- Agentic BSS with Digital Manager Agent continuously segmenting users and coordinating Offer, Product, and Assessment agents to launch new offerings within hours
- "Empathetic Marketing and Care": AI-driven personalisation matching the right offer to the right subscriber at the right time
- SmartCare Intelligence: proactive customer care powered by network and behavioural analytics
- AUTINOps: AI-Native framework for autonomous network operations (announced MWC 2026)
- Convergent billing system (CBS) fully cloudified for public, private, hybrid, or on-premise deployment
- Supports B2C, B2B, IoT, and 5G business scenarios with rapid service rollout templates

**Differentiating features**
- Multi-agent orchestration framework enabling agents to negotiate and execute product launches end-to-end without human initiation
- Industry's first AI-Native intelligent operations framework (per Huawei's own claim, MWC 2026)
- Strong Asia-Pacific and emerging-market operator relationships providing real-world at-scale validation

**UX patterns**
- Agent-driven product launch workflow: business users specify intent, AI agents handle decomposition and execution
- Unified 360-degree subscriber view linking network, billing, and care interactions
- Mobile-optimised self-care and B2B portal templates

**Integration points**
- Deep integration with Huawei network equipment (5G core, radio, transport) for tight service-network correlation
- Open ecosystem APIs for third-party partners and app marketplace integration
- TM Forum Open API alignment

**Known gaps**
- Geopolitical restrictions limit deployments in North America and parts of Europe
- Tight coupling with Huawei network equipment disadvantages multi-vendor network operators
- Limited transparency around AI model governance and data sovereignty practices

**Licence / IP notes**
- Fully proprietary. Subject to export control regulations in certain markets.

---

### Wavelo

**Core features**
- Cloud-native BSS platform purpose-built for digital-first and MVNO operators
- Converged billing: mobile, broadband, OTT, and IoT services rated and billed on a single platform
- Event-driven architecture (EDA) on Apache Kafka: network events stream in real time, triggering billing, provisioning, and care updates simultaneously
- Customer order management with automated provisioning and zero-touch network activation
- Product catalogue supporting rapid offer configuration and multi-channel distribution
- Experience Orchestration module for personalised 5G and cloud customer journeys

**Differentiating features**
- Modular "pick and mix" adoption: operators can adopt billing, provisioning, or catalogue independently without a full platform replacement
- TMF API compatibility built in as a first-class architecture principle, not a retrofit
- Real-time EDA architecture eliminates batch billing cycles; events propagate sub-second to all downstream systems
- Developer portal (dev.wavelo.com) designed for operator engineering teams, not just vendor consultants

**UX patterns**
- Self-service operator portal with configuration wizards for launching new products without coding
- Real-time event monitor giving operations teams live visibility into network, billing, and care events
- API-first design allowing digital channels (app, web, call centre) to be built on the same platform APIs

**Integration points**
- TMF Open APIs (TMF620, TMF622, and others)
- Apache Kafka event backbone for streaming integration
- REST API with developer portal and documentation at dev.wavelo.com
- Designed for integration with third-party CRM, payment gateways, and network management systems

**Known gaps**
- Narrower enterprise and wholesale BSS capability compared to Amdocs or NetCracker
- OSS depth is limited — network inventory and fault management require third-party OSS integration
- Primarily targets MVNOs and greenfield operators; complex legacy migration tooling is limited

**Licence / IP notes**
- Fully proprietary SaaS. Subsidiary of Tucows Inc.

---

### Cerillion Enterprise BSS/OSS

**Core features**
- Integrated BSS/OSS suite covering CRM, product catalogue, order management, convergent charging, billing, analytics, and network management
- 3GPP-compliant Convergent Charging System (CCS) with Nchf interface for 5G prepaid/postpaid/hybrid charging
- Self-service module with customer and partner portals
- Service catalogue driving technical provisioning alongside commercial product catalogue
- Agent2Agent (A2A) AI coordination introduced in release 26.1: AI agents communicate and execute tasks across internal and external systems collaboratively
- "Ready for ODA" certified by TM Forum (2025); Diamond-level API conformance

**Differentiating features**
- A2A AI coordination enabling multi-step cross-system automation without a central orchestrator — distinguishes it from single-agent platforms
- TM Forum Diamond certification: the highest conformance tier for Open APIs, important for operator procurement scoring
- Mid-market price point with full-stack capability, competing with large vendors at lower total cost of ownership
- Active product release cadence (annual versioned releases, e.g. 26.1 in 2026)

**UX patterns**
- Unified operator console integrating BSS and OSS workflows in a single interface
- Role-based dashboards for billing, care, operations, and network teams
- AI Hub: centralised management surface for configuring and monitoring AI agents

**Integration points**
- TM Forum Open APIs (Diamond certified)
- 3GPP TS 32.290 / 32.291 for 5G convergent charging
- REST APIs for third-party CRM, ERP, mediation, payment, and provisioning integration

**Known gaps**
- Smaller ecosystem partner network than Amdocs or NetCracker
- Limited public API documentation; integration requires vendor engagement
- AI capabilities (A2A) are newly released; real-world production maturity unproven

**Licence / IP notes**
- Fully proprietary. TM Forum Open API certified (Diamond level). Listed on London Stock Exchange (CER).

---

### MATRIXX Software

**Core features**
- Convergent Charging System (CCS): single real-time platform for rating, balance management, and charging across all access types, services, devices, and payment methods
- 5G 3GPP-compliant CCS: first vendor to deliver a charging system meeting 3GPP Release 15 Nchf interface specification
- Flexible billing modes: on-demand, recurring subscription, and traditional billing cycles on the same account
- Cloud-native on Google Cloud and other hyperscalers
- API-first architecture for rapid integration with BSS partners and third-party applications
- Dynamic pricing engine enabling real-time tariff and allowance adjustment

**Differentiating features**
- Pure-play charging and monetisation specialist — deeper than the charging modules of full-stack BSS platforms
- First-mover 5G 3GPP compliance provides customer confidence for 5G monetisation deployments
- Deployed by Tier-1 operators (e.g. Telstra); proven at hyperscale

**UX patterns**
- Charging configuration UI designed for product and pricing teams without engineering involvement
- Real-time balance dashboard for operations teams to monitor subscriber quotas and spending
- Google Cloud marketplace deployment reducing procurement friction

**Integration points**
- 3GPP Nchf REST-based interface (TS 32.290 / 32.291)
- Diameter interface for 4G/LTE backward compatibility
- Extensive BSS API library for integration with upstream billing and CRM
- Google Cloud native with marketplace listing

**Known gaps**
- Charging specialist only — operators still need a separate billing, CRM, and OSS platform
- Not a full BSS/OSS stack; integration complexity with other vendor modules
- Pricing and contract terms are enterprise-only; not suitable for smaller operators

**Licence / IP notes**
- Fully proprietary. 5G charging patents held; specific patent claims not publicly disclosed.

---

### Salesforce Communications Cloud

**Core features**
- Catalog-driven digital BSS from commerce to cash: product catalogue, order management, billing inquiry, and revenue operations
- Agentforce for Communications: AI agents for 24/7 automated customer support, order status, and billing troubleshooting
- Billing Inquiry Manager: AI-assisted billing explanation tool for care agents (predictive + generative AI)
- Enterprise product catalogue supporting both commercial and technical products, with third-party catalogue integration
- Sales quoting automation (MWC 2026 release)
- 360-degree customer view unifying network, billing, and CRM data on the Salesforce platform

**Differentiating features**
- Largest CRM ecosystem globally — operators gain access to the full Salesforce AppExchange partner network
- Agentforce 360 for Communications: cross-domain AI agent orchestration covering sales, care, and operations from a single platform
- Low total-cost-of-ownership for BSS layers that overlap with standard Salesforce CRM capabilities
- No-code/low-code configuration via Salesforce Flow for business users

**UX patterns**
- Familiar Salesforce Lightning UI reduces operator training overhead
- Self-service portal built on Salesforce Experience Cloud
- Agentforce AI surfaced directly in agent console and customer portal without separate AI management layer

**Integration points**
- Salesforce REST and GraphQL APIs with OpenAPI documentation
- AppExchange ecosystem for third-party OSS and network integration (e.g. Enxoo + CROSS Network Intelligence)
- TMF Open API alignment for product catalogue and ordering

**Known gaps**
- OSS capabilities absent — network inventory, provisioning, and fault management require third-party integration
- Charging depth limited compared to MATRIXX or Cerillion; suitable mainly for postpaid bill presentment
- Vendor lock-in to Salesforce platform; data portability constraints
- Per-user Salesforce licensing can become expensive at large agent scale

**Licence / IP notes**
- Fully proprietary SaaS. Salesforce platform data residency and sovereignty terms apply.

---

### Comarch BSS/OSS

**Core features**
- BSSSmart: all-in-one pre-configured platform covering billing, rating, charging, pre-sales, product catalogue, fulfilment, and site installation management
- Resource Inventory: federated, AI-enhanced inventory spanning physical and logical network resources, supporting network digital twins
- Orchestration Management: SMO/MDO engine for zero-touch autonomous network operations with model-driven tool layer and agentic AI
- Fault Management with AI/ML for proactive fault resolution and automated root-cause analysis
- Performance Management with AI-driven analytics, self-healing automation, and ROI reporting
- Integration Platform: data lake architecture processing petabytes in real time, enabling closed-loop agentic automation
- 20 TM Forum–compliant Open APIs aligned to ODA

**Differentiating features**
- ODA-aligned portfolio covering both BSS and OSS with 20 certified TM Forum Open APIs — breadth matched by few mid-market vendors
- Data-lake-first integration architecture positioning Comarch as an orchestration hub, not just a BSS point solution
- Orchestration Management as a central MDO that can govern third-party vendor network functions without requiring full platform replacement

**UX patterns**
- Role-specific portals for billing teams, network operations, and field technicians
- AI-powered service desk with reinforcement-learning ticket resolution and human-in-the-loop audit controls
- Zero-touch configuration change management with policy-driven guardrails

**Integration points**
- 20 TM Forum Open APIs (ODA-aligned)
- Agile microservices architecture with containerised deployment on Kubernetes
- Interoperability with third-party NMS vendors via standard interfaces
- Cloud-native with support for public, private, and hybrid cloud

**Known gaps**
- Less brand recognition than Amdocs, Ericsson, or Nokia in large Tier-1 operator procurement shortlists
- Professional services dependency for complex configuration and customisation
- AI capabilities (agentic orchestration) appear newer and lack published independent validation

**Licence / IP notes**
- Fully proprietary. TM Forum ODA-compliant (20 Open APIs).

---

### Sunvizion (OSS Specialist)

**Core features**
- Network Inventory: physical, logical, outside plant, inside plant, and service inventory across FTTH, copper, coax, FTTx, DWDM, SDH/PDH, Ethernet, IP/MPLS, and ATM
- Service Fulfilment: automated end-to-end service activation for B2C and B2B; direct NMS integration for zero-touch provisioning
- Service Order Management: full process automation from order receipt through network configuration
- Network Roll-Out Management: project tracking and resource scheduling for network build programmes
- Workflow Management: built-in workflow engine automating business processes across network management and service monetisation
- Broad NMS interoperability: aggregates data from all NMS systems regardless of vendor

**Differentiating features**
- Multi-technology network inventory depth unmatched by BSS-led platforms (FTTH through IP/MPLS in one model)
- End-to-end service-to-resource correlation: every service instance is linked to the physical resources delivering it
- Strong focus on fixed-network operators; differentiator for fibre and broadband-heavy CSPs

**UX patterns**
- Map-based network topology viewer for field and planning teams
- Automated workflow notifications keeping fulfilment teams informed without manual chasing
- Service verification views linking logical service state to physical resource state

**Integration points**
- NMS-agnostic southbound integration (supports all major NMS platforms)
- TM Forum-aligned service and resource API model
- REST APIs for integration with BSS platforms (billing, CRM, order management)

**Known gaps**
- Billing and CRM capabilities are minimal; Sunvizion is an OSS complement, not a full BSS/OSS stack
- AI and machine-learning features are less developed compared to Amdocs or Comarch
- Limited global brand presence outside European fixed-network operators

**Licence / IP notes**
- Fully proprietary. Developed by Suntech S.A. (Poland).

---

### BillRun (Open Source BSS)

**Core features**
- CDR mediation processing call detail records from network elements
- Real-time OCS (Online Charging System) for prepaid balance management
- Rating and charging engine supporting prepaid, postpaid, roaming, and wholesale
- Fraud detection module
- CRM and automated customer portal
- Number Portability Gateway
- IoT billing support

**Differentiating features**
- Only significant open-source BSS platform with a complete stack including OCS, mediation, billing, CRM, and customer portal
- AGPLv3 licence ensures downstream modifications remain open, protecting community contributions
- IoT billing support in an open-source context is unusual and valuable for smaller IoT-focused operators

**UX patterns**
- Web-based operator console with standard billing workflow
- Customer self-service portal included out of the box
- API-first design enabling integration with third-party systems

**Integration points**
- REST APIs for third-party integration
- CDR file ingestion supporting industry-standard formats
- Modular architecture supporting custom plugin development

**Known gaps**
- No enterprise-grade OSS (network inventory, fault management, performance management)
- Community support only; professional SLA support requires third-party vendors
- Limited 5G charging capability; 3GPP Nchf interface not implemented
- UI/UX dated compared to commercial platforms
- TM Forum Open API compliance not certified

**Licence / IP notes**
- AGPLv3. Use in commercial SaaS products requires either open-sourcing the full deployment or a commercial licence from BillRun Technologies.

---

### OpenSlice (Open Source OSS)

**Core features**
- Network Slice as a Service (NSaaS) delivery using ETSI ZSM (Zero-touch network and Service Management) architecture
- TM Forum Open APIs for service ordering and service inventory
- Cross-domain service orchestration spanning multiple administrative domains
- REST-based northbound API for external BSS integration

**Differentiating features**
- Purpose-built for 5G network slicing lifecycle management — the only open-source platform in this niche
- Implements ETSI ZSM and TM Forum Open APIs concurrently, providing standards-complete reference implementation

**Known gaps**
- Narrow scope (slicing only); not a general-purpose OSS
- Limited production deployments; primarily used in research and trial environments
- Requires significant engineering effort to integrate with commercial BSS or network management systems

**Licence / IP notes**
- Apache 2.0.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Convergent charging supporting prepaid, postpaid, and hybrid on a single platform
- Product catalogue management with version control and multi-channel distribution
- Order management from sales channel through network provisioning
- CRM and 360-degree customer view
- Network inventory covering physical and logical resources
- Fault management with alarm correlation and trouble-ticket integration
- TM Forum Open API compliance (TMF620, TMF622, TMF638, TMF641)
- Cloud-native deployment (Kubernetes, public/private/hybrid cloud)
- Role-based access control and multi-tenancy for operator and partner isolation
- GDPR and telecommunications data retention compliance tooling

### Differentiating Features
- Agentic AI frameworks with domain-specific agents collaborating across BSS and OSS domains (Amdocs, Huawei, Cerillion)
- Real-time event-driven architecture replacing batch billing cycles (Wavelo, MATRIXX)
- Digital-twin-backed network inventory enabling simulation and impact analysis (Comarch, Amdocs)
- Agent2Agent (A2A) coordination for cross-platform agentic task execution (Cerillion 26.1)
- Pure-play charging depth with 3GPP Nchf compliance (MATRIXX, Cerillion CCS)
- Modular "pick and mix" adoption paths reducing transformation risk (Wavelo, NetCracker)
- Open-source API management platform alongside proprietary BSS/OSS (NetCracker Qubership APIHUB)

### Underserved Areas / Opportunities
- Unified BSS-OSS intelligence layer: most platforms treat BSS and OSS as separate domains with separate AI models; cross-domain causal reasoning (e.g. linking a network fault directly to billing impact) is absent or nascent
- Small and mid-size operator affordability: the full-stack AI-native platforms are priced for Tier-1 carriers; MVNOs and regional operators are underserved by capable, affordable full-stack options
- Explainable AI for regulatory compliance: BSS decisions (bill adjustments, fraud flags) need auditable explanations for regulators; current AI layers do not surface this adequately
- Open data model and interoperability: despite TM Forum Open API standards, data model lock-in remains pervasive; an operator cannot move from Amdocs to NetCracker without a major data migration project
- Real-time slice-aware billing: 5G network slicing with per-slice SLA billing is theoretically supported but practically complex; end-to-end automated slice billing is not production-proven across vendors
- Self-service for enterprise customers: B2B self-service portals for enterprise buyers to configure, order, and monitor services are weak or absent in most platforms
- Carbon and energy reporting: sustainability KPIs tied to network operations and billing are not natively surfaced in any platform

### AI-Augmentation Candidates
- Anomaly detection in CDR streams: rule-based fraud detection is table-stakes; AI pattern recognition across billions of events per day is systematically underexplored in open offerings
- Intelligent offer personalisation: moving from segment-based pricing to individual subscriber AI-generated offers in real time
- Automated root-cause analysis linking OSS fault events to BSS churn signals — enabling proactive retention rather than reactive repair
- Natural-language product catalogue configuration: business users describing a new offer in plain language and having the AI generate the catalogue entity, pricing rules, and provisioning workflow
- Predictive capacity planning: correlating network performance trends, subscriber growth, and billing data to forecast capacity requirements and trigger automated procurement

---

## Legal & IP Summary

All major commercial platforms (Amdocs, Ericsson, NetCracker/Nokia, Huawei, Wavelo, Cerillion, MATRIXX, Salesforce, Comarch, Sunvizion) are fully proprietary. Huawei's platform is subject to export control restrictions in several jurisdictions. MATRIXX Software holds patents in the convergent charging domain, though specific claims are not publicly enumerated. TM Forum Open APIs are published under a royalty-free licence for implementation, but the API specifications themselves are TM Forum copyright. The open-source alternatives (BillRun under AGPLv3, OpenSlice and Kuwaiba under Apache 2.0) can serve as reference implementations or starting points, but BillRun's AGPLv3 licence imposes copyleft obligations on SaaS deployments. No specific patent blocking concerns were identified for building a new AI-native BSS/OSS platform using TM Forum Open APIs and 3GPP specifications, provided no proprietary algorithm or data model is copied directly from a commercial vendor.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Product catalogue management with TM Forum TMF620 API compliance
- Convergent charging engine supporting prepaid, postpaid, and hybrid (3GPP Nchf-compliant)
- Order management and automated service provisioning (TMF622, TMF641)
- Subscriber / CRM module with 360-degree customer view
- Basic network inventory (physical and logical) with service-to-resource linking (TMF638)
- Fault management with alarm ingestion, correlation, and trouble-ticket generation
- Role-based access control, multi-tenancy, and audit logging

**Should-have (v1.1)**
- Real-time event-driven architecture (Kafka or equivalent) for sub-second billing and provisioning event propagation
- AI-native anomaly detection on CDR and network event streams
- Self-service portal for subscribers and enterprise B2B buyers
- 5G network slice lifecycle management with per-slice SLA billing
- Natural-language product catalogue configuration (LLM-assisted)
- ODA component alignment and TM Forum Open API certification path

**Nice-to-have (backlog)**
- Digital twin of the network estate for simulation and what-if planning
- Agent2Agent (A2A) multi-agent orchestration across BSS and OSS domains
- Carbon and energy consumption reporting tied to network operations
- Automated root-cause analysis linking OSS fault events to BSS subscriber impact
- Predictive churn detection combining network quality and billing behavioural signals
- Marketplace for third-party service components and partner billing settlement

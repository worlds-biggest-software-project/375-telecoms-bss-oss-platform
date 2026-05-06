# Standards & API Reference

> Project: Telecoms BSS/OSS Platform · Generated: 2026-05-06

---

## Industry Standards & Specifications

### TM Forum Open APIs and Open Digital Architecture

**TM Forum Open APIs (TMF Open API suite)**
- URL: https://www.tmforum.org/oda/open-apis/
- GitHub: https://github.com/tmforum-apis
- Over 100 REST-based, event-driven, and domain-specific APIs collaboratively developed by TM Forum members; more than 1,100,000 downloads by 60,000+ developers from 2,900 organisations. The most commercially significant APIs for a BSS/OSS platform are: TMF620 (Product Catalogue Management), TMF621 (Product Ordering Management), TMF622 (Product Ordering), TMF638 (Service Inventory), TMF641 (Service Ordering), and TMF724 (Event Management).

**TM Forum Gen5 Open APIs**
- URL: https://www.tmforum.org/open-digital-architecture/open-apis/
- Fifth-generation TMF APIs adding event-driven architecture support, a simplified developer experience, and intent-based automation capabilities aligned with autonomous-network goals. Gen5 APIs are actively being adopted in 2026 and are the target for new platform implementations.

**TM Forum Open Digital Architecture (ODA)**
- URL: https://www.tmforum.org/open-digital-architecture/
- GitHub: https://github.com/tmforum-oda
- Standardised cloud-native enterprise architecture blueprint for all elements of the telecom industry. ODA defines reusable, standardised software components grouped into five functional blocks: Party Management, Core Commerce Management, Production, Intelligence & Insights, and Common Functionality. The ODA Component Certification process (launched 2025) allows vendors to certify their solutions via the TM Forum Conformance Test Kit (CTK). Being listed as an ODA Component Directory Partner signals interoperability assurance to operator procurement teams.

---

### 3GPP Standards (Charging and 5G Service Architecture)

**3GPP TS 32.290 — 5G Charging: Services, Operations and Procedures (SBI)**
- URL: https://www.etsi.org/deliver/etsi_ts/132200_132299/132290/
- Defines the service-based interface (SBI) for convergent charging in 5G, specifying the Nchf interface exposed by the Charging Function (CHF). Core operations are Nchf_ConvergedCharging_Create, Nchf_ConvergedCharging_Update, and Nchf_ConvergedCharging_Release — the foundation for any 3GPP-compliant charging system.

**3GPP TS 32.291 — 5G Charging Service (Stage 3)**
- URL: https://www.etsi.org/deliver/etsi_ts/132200_132299/132291/
- Stage 3 protocol specification for 5G charging, covering the Nchf REST-based interface in detail. Any platform targeting 5G operators must implement this spec to integrate with the 5G core network's session management function (SMF) and other network functions.

**3GPP Convergent Charging System (CCS) Architecture**
- URL: https://www.3gpp.org/technologies/slicing-charging
- The CCS consolidates online (prepaid) and offline (postpaid) charging into a single platform. Core components defined by 3GPP: Charging Function (CHF), Rating Function (RF), Account and Balance Management Function (ABMF), and Charging Gateway Function (CGF). This architecture is required for 5G monetisation and slice-aware billing.

---

### ETSI Standards

**ETSI NFV-MAN 001 — Network Functions Virtualisation: Management and Orchestration**
- URL: https://www.etsi.org/technologies/nfv
- Specification: https://www.etsi.org/deliver/etsi_gs/NFV-MAN/001_099/001/
- Defines the NFV MANO framework covering service lifecycle management: service definition, automation, error correlation, monitoring, and VNF lifecycle management. The reference point Os-Ma-nfvo connects OSS/BSS to the NFV Orchestrator (NFVO), specified in NFV-SOL005. Essential for any OSS platform managing virtualised or cloud-native network functions.

**ETSI NFV-SOL 003 and SOL 005 — REST APIs for MANO Interfaces**
- URL: https://www.etsi.org/committee/nfv
- SOL003 specifies the Or-Vnfm interface (NFVO to VNFM); SOL005 specifies the Os-Ma-nfvo interface (OSS/BSS to NFVO). Both use REST-based APIs enabling OSS platforms to orchestrate virtualised network function lifecycles without proprietary integration.

**ETSI ZSM (Zero-touch network and Service Management)**
- URL: https://www.etsi.org/technologies/zero-touch-network-service-management
- Defines end-to-end automation architecture for cross-domain network management with closed-loop automation. ZSM is the standards basis for autonomous network operations (L4/L5 maturity) and is implemented by open-source projects such as OpenSlice.

---

### GSMA Open Gateway and CAMARA

**GSMA Open Gateway**
- URL: https://www.gsma.com/solutions-and-impact/gsma-open-gateway/
- Developer documentation: https://open-gateway.gsma.com/docs
- A global framework of common network APIs exposing core operator capabilities (identity, location, fraud prevention, communication services) to developers and cloud providers. Aligned with 86 operator groups representing 300+ networks and 80% of global mobile connections as of 2026. A modern BSS platform should support GSMA Open Gateway API exposure for enterprise and developer monetisation.

**CAMARA (Linux Foundation)**
- URL: https://www.gsma.com/solutions-and-impact/technologies/networks/operator-platform-hp/camara-2/
- GitHub: https://github.com/camaraproject
- Open-source project defining RESTful API specifications, data models, and security patterns for accessing network services. CAMARA is the technical standards body underpinning GSMA Open Gateway, published under Apache 2.0. Reference implementations provide freely usable API definitions for network capability exposure.

---

### Security and Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect 1.0**
- OAuth 2.0: https://datatracker.ietf.org/doc/html/rfc6749
- OpenID Connect: https://openid.net/connect/
- Industry-standard authorisation and identity federation protocols required for BSS API security, subscriber self-service portal authentication, and partner API access management. TM Forum Open APIs specify OAuth 2.0 as the mandatory authentication mechanism.

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- The standard reference for API security risk in REST-based platforms. Directly applicable to BSS/OSS API layers exposed to partners, developers, and enterprise customers. Key risks include broken object-level authorisation, excessive data exposure, and injection attacks on query parameters.

**NIST SP 800-207 — Zero Trust Architecture**
- URL: https://doi.org/10.6028/NIST.SP.800-207
- Increasingly required by operator security policies for cloud-native BSS/OSS deployments; mandates continuous verification of all internal and external API calls rather than perimeter-based trust. Particularly relevant for multi-vendor ODA component environments where component boundaries require explicit authorisation.

**GDPR (EU 2016/679) and Telecommunications Data Retention Directives**
- GDPR: https://gdpr-info.eu/
- Imposes subscriber data minimisation, right-to-erasure, purpose limitation, and breach notification requirements on any BSS platform storing European subscriber data. Supplemented by national telecommunications data retention laws (e.g. the UK Investigatory Powers Act) which require operators to retain certain CDR metadata for law enforcement purposes — creating a tension with GDPR minimisation that the platform data model must resolve.

---

### Data Model and API Specification Standards

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The de facto standard for documenting REST APIs. All TM Forum Open APIs and CAMARA APIs are published as OpenAPI 3.x specifications. A new BSS/OSS platform should publish its APIs using OAS 3.1 to enable automated SDK generation and integration tooling.

**AsyncAPI 2.x / 3.x**
- URL: https://www.asyncapi.com/
- Standard for documenting event-driven and message-based APIs (Kafka, WebSocket, AMQP). TM Forum Gen5 APIs add AsyncAPI support to complement REST specifications, directly relevant to real-time billing and provisioning event streams.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/
- Used within OpenAPI and AsyncAPI specifications to define data models. TM Forum Open API data models are expressed as JSON Schema; a platform's internal data model should align with these schemas to minimise integration friction.

**Protocol Buffers (Protobuf) / gRPC**
- URL: https://protobuf.dev/
- Used in high-throughput internal service communication (e.g. between mediation and charging layers). NetCracker's Qubership APIHUB documents gRPC interfaces alongside REST. Consider for internal OSS microservice communication where latency is critical.

---

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- Baseline information security management standard required by most Tier-1 and Tier-2 operators in vendor procurement. A BSS/OSS platform must be deployable in an ISO 27001-certified environment and its own software development lifecycle should comply.

**ISO/IEC 19086-1 to 19086-4 — Cloud Computing SLA Framework**
- URL: https://www.iso.org/standard/67545.html
- Defines common SLA building blocks, metric models, conformance requirements, and security/PII components for cloud services. Relevant when defining SLA terms for a cloud-hosted BSS/OSS offering; operators will expect alignment with this framework in commercial contracts.

**ISO/IEC 20000-1:2018 — IT Service Management**
- URL: https://www.iso.org/standard/70636.html
- IT service management standard frequently referenced in operator procurement for BSS/OSS managed service engagements. Defines requirements for planning, establishing, implementing, operating, monitoring, maintaining, and improving a service management system.

---

## Similar Products — Developer Documentation & APIs

### Amdocs connectX Developer Portal

- **Description:** Developer portal for Amdocs' BSS/OSS platform, providing API documentation, integration guides, and sandbox access for operator engineering teams and ecosystem partners.
- **API Documentation:** https://devportal.amdocs-dbs.com/
- **SDKs/Libraries:** Not publicly disclosed; API access requires commercial agreement.
- **Developer Guide:** Available via developer portal (registration required).
- **Standards:** TM Forum Open APIs, 3GPP, ETSI; REST/JSON.
- **Authentication:** OAuth 2.0.

---

### NetCracker / Qubership APIHUB

- **Description:** Kubernetes-native API registry and developer portal supporting OpenAPI, GraphQL, AsyncAPI, RESTCONF, NETCONF/YANG, and TOSCA specifications. Published open-source by NetCracker under the Qubership brand.
- **API Documentation:** https://www.netcracker.com/portfolio/products/netcracker-api-management-and-integration.html
- **SDKs/Libraries:** https://github.com/Netcracker/qubership-apihub (Go backend; UI repo at github.com/Netcracker/qubership-apihub-ui)
- **Developer Guide:** https://tmforum-oda.github.io/oda-ca-docs/ (ODA component context)
- **Standards:** REST/JSON, GraphQL, AsyncAPI, RESTCONF, NETCONF/YANG, TOSCA, OpenAPI 2.0 and 3.0.
- **Authentication:** Internal authentication via Go-based backend; OAuth 2.0 for external API access.

---

### Wavelo Developer Portal

- **Description:** Cloud-native BSS developer portal for CSPs, MVNOs, and engineering teams building on Wavelo's event-driven BSS platform.
- **API Documentation:** https://dev.wavelo.com/
- **SDKs/Libraries:** Not publicly disclosed; REST API access documented on developer portal.
- **Developer Guide:** https://dev.wavelo.com/ (registration may be required for full access).
- **Standards:** TMF Open APIs (TMF620, TMF622 and others), REST/JSON, Apache Kafka event streaming.
- **Authentication:** OAuth 2.0 / API key.

---

### Cerillion BSS/OSS API Suite

- **Description:** TM Forum Diamond-certified REST API layer for Cerillion's BSS/OSS suite, supporting product catalogue, ordering, billing, charging, and network management integration. Release 26.1 (2026) adds A2A agent API interfaces.
- **API Documentation:** https://www.cerillion.com/products/bssoss-suite/integration-layer/
- **SDKs/Libraries:** Not publicly available; integration via REST APIs documented under commercial agreement.
- **Developer Guide:** https://www.cerillion.com/products/bssoss-suite/integration-layer/
- **Standards:** TM Forum Open APIs (Diamond certified), 3GPP TS 32.290/32.291 (Nchf), ODA-aligned. REST/JSON.
- **Authentication:** OAuth 2.0.

---

### MATRIXX Software API

- **Description:** 5G-compliant convergent charging API exposing real-time rating, balance management, and billing interfaces for integration with BSS partners and network functions.
- **API Documentation:** https://www.matrixx.com/5g-bss-technology/
- **SDKs/Libraries:** Not publicly available; access requires commercial engagement.
- **Developer Guide:** https://www.matrixx.com/converged-charging-and-monetization/
- **Standards:** 3GPP TS 32.290/32.291 (Nchf REST interface), Diameter (4G legacy), REST/JSON, Google Cloud native.
- **Authentication:** OAuth 2.0; mTLS for network function-to-charging function interfaces per 3GPP security architecture.

---

### Salesforce Communications Cloud API

- **Description:** Salesforce REST and GraphQL APIs exposing product catalogue, order management, billing inquiry, and AI agent (Agentforce) capabilities for telecom BSS integration and digital channel development.
- **API Documentation:** https://developer.salesforce.com/docs/communications/
- **SDKs/Libraries:** Salesforce SDKs available for JavaScript, Python, Java, Apex, and others: https://developer.salesforce.com/tools/
- **Developer Guide:** https://developer.salesforce.com/docs/communications/
- **Standards:** REST/JSON, GraphQL, OpenAPI, TMF Open API alignment for product catalogue and ordering. Salesforce Apex for custom logic.
- **Authentication:** OAuth 2.0 (Salesforce Connected App flow).

---

### TM Forum Open API Reference Implementations (GitHub)

- **Description:** Official TM Forum GitHub organisation publishing Open API specifications, Postman collections, and reference implementation artefacts for all TMF APIs.
- **API Documentation:** https://www.tmforum.org/oda/open-apis/
- **SDKs/Libraries:** https://github.com/tmforum-apis — individual repos per API (e.g. TMF620, TMF622, TMF638, TMF641).
- **Developer Guide:** https://www.tmforum.org/open-digital-architecture/implementation/open-apis/
- **Standards:** REST/JSON, AsyncAPI (Gen5), OpenAPI 3.x; all specs published under TM Forum royalty-free licence.
- **Authentication:** OAuth 2.0 (specified per-API in the security scheme).

---

### GSMA Open Gateway / CAMARA API Reference

- **Description:** Standardised network capability APIs (identity, location, fraud prevention, communication services) exposed by 300+ mobile networks globally through a common API framework.
- **API Documentation:** https://open-gateway.gsma.com/docs
- **SDKs/Libraries:** https://github.com/camaraproject — Apache 2.0 licensed reference implementations per network capability.
- **Developer Guide:** https://open-gateway.gsma.com/docs; operator-specific onboarding via channel partners (hyperscalers, aggregators).
- **Standards:** REST/JSON, OpenAPI 3.x, OAuth 2.0 / OIDC for API authorisation (CAMARA security scheme).
- **Authentication:** OAuth 2.0 / OpenID Connect (CAMARA-specified authorisation flow using operator identity federation).

---

### BillRun Open Source BSS

- **Description:** Open-source telecom BSS providing CDR mediation, real-time OCS, convergent rating/charging, CRM, self-service portal, and IoT billing. AGPLv3 community project.
- **API Documentation:** https://bill.run/
- **SDKs/Libraries:** https://github.com/billrun — community-maintained; primarily PHP/Node.js.
- **Developer Guide:** https://bill.run/ (community documentation).
- **Standards:** REST/JSON; CDR file ingestion (industry-standard formats). TM Forum Open API compliance not certified.
- **Authentication:** API key; session-based authentication for operator portal.

---

## Notes

**Emerging standards to watch:**
- **TM Forum Gen5 APIs**: The transition from REST-only (Gen4) to event-driven (Gen5) APIs is actively underway in 2026. New platforms should target Gen5 from inception rather than retrofitting.
- **3GPP Release 18 and 19**: Future 3GPP releases are extending the charging and service architecture for advanced 5G features (RedCap IoT, 5G-Advanced network slicing enhancements). Platform architects should track the evolving 3GPP charging specifications.
- **AI / Agentic API standards**: As AI agents increasingly orchestrate BSS/OSS operations, there is no current industry standard for agent-to-agent (A2A) communication in the telecom domain. Cerillion's A2A implementation (2026) is vendor-proprietary. TM Forum's AI-native ODA initiative may produce standards in 2026–2027.
- **CAMARA adoption velocity**: CAMARA is growing rapidly but API coverage is still limited to a subset of network capabilities. Operators and BSS platforms building for enterprise developer markets should monitor CAMARA roadmap releases closely.
- **Data sovereignty and localisation**: Several jurisdictions (India, Brazil, Saudi Arabia, Indonesia) impose strict data localisation requirements on subscriber data processed by BSS platforms. Multi-region deployment architecture with data residency controls is becoming a procurement requirement, not a differentiator.

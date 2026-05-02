# 375 – Telecoms BSS/OSS Platform

**Date:** 2026-05-02

---

## 1. Problem Statement

Telecommunications operators run two parallel software stacks that must work in close coordination: Business Support Systems (BSS) handle the commercial relationship with subscribers—billing, ordering, CRM, and revenue management—while Operational Support Systems (OSS) manage the network itself—inventory, provisioning, fault management, and performance assurance. Historically these stacks were separate, heavily customised, and built on monolithic architectures that take years to change. As operators roll out 5G, expand IoT connectivity, and launch cloud-based services, they need to activate new products in hours rather than months and provide seamless, real-time customer experiences. The core challenge is a modern, integrated BSS/OSS platform that supports rapid service introduction, automated fulfilment, and a unified view of network and customer across the full subscriber lifecycle.

---

## 2. Market Landscape

The BSS/OSS market is served by a mix of large established vendors and cloud-native challengers:

- **Amdocs:** among the largest BSS/OSS vendors; serves major carriers globally with billing, ordering, and network management suites
- **Ericsson (BSCS, OSS):** deep OSS heritage from its network equipment business; offers integrated operations and business support for mobile operators
- **Nokia (NetCracker):** full-stack BSS/OSS from network inventory through billing, with a strong TM Forum Open API alignment
- **Huawei BSS:** significant in Asia-Pacific and emerging markets; integrated with Huawei network infrastructure
- **Wavelo:** cloud-native BSS platform targeting digital-first operators with billing, orchestration, and cloud provisioning built for rapid launch
- **Sunvizion:** specialist OSS platform covering network inventory and service fulfilment
- **Circles (TaaS model):** Telecom-as-a-Service approach that wraps BSS/OSS capabilities into an operator-in-a-box for MVNOs and greenfield launches

---

## 3. Key Features & Capabilities

A full-stack BSS/OSS platform covers two intersecting domains:

**BSS capabilities:**
- **Subscriber management:** customer profiles, contract management, lifecycle events (activation, upgrade, churn)
- **Product catalogue:** centralised catalogue driving consistent offers across channels; rapid product configuration without code changes
- **Order management:** end-to-end order orchestration from sales channel through network fulfilment
- **Billing and revenue management:** convergent billing for postpaid, prepaid, and hybrid; mediation, rating, invoicing, and revenue assurance
- **CRM and self-service:** 360-degree customer view, customer support tooling, digital self-service portal and mobile app

**OSS capabilities:**
- **Network inventory management:** physical and logical network resource records; topology modelling; capacity tracking
- **Service fulfilment and provisioning:** automated configuration of network elements on order activation; zero-touch provisioning for 5G network slices
- **Fault management:** alarm correlation, event suppression, root-cause analysis, trouble-ticket integration
- **Performance management:** KPI collection, threshold alerting, and trend reporting across the network estate
- **Workforce management:** field technician scheduling and work-order management for network operations crews

---

## 4. Technical Considerations

BSS/OSS platforms present distinctive architectural challenges:

- **TM Forum Open APIs:** industry-standard API framework (TMF APIs) is increasingly required for interoperability with partner systems, cloud marketplaces, and wholesale settlement; non-compliant platforms face integration cost penalties
- **Microservices migration:** operators are under pressure to decompose monolithic stacks into loosely coupled services; vendors must support hybrid deployments where legacy and cloud-native components coexist during extended migration periods
- **5G network slicing:** end-to-end slice lifecycle management (creation, modification, teardown) requires tight OSS integration with the 5G core and radio access network; this is a key differentiator in 2026
- **Real-time rating:** IoT and usage-based services require real-time or near-real-time rating rather than batch billing cycles; event streaming platforms (Kafka) underpin modern mediation layers
- **Data privacy:** GDPR, CCPA, and country-specific telecommunications data retention laws impose strict requirements on subscriber data handling, retention, and erasure
- **Scalability:** large operators serve hundreds of millions of subscribers; the platform must horizontally scale both transaction processing and reporting without sacrificing consistency

---

## 5. Citations

1. Wikipedia – "OSS/BSS" — [https://en.wikipedia.org/wiki/OSS/BSS](https://en.wikipedia.org/wiki/OSS/BSS)
2. Infraon – "OSS in Telecom – What is it and why is it important in 2026?" — [https://infraon.io/blog/what-are-operational-support-systems-oss-in-telecom/](https://infraon.io/blog/what-are-operational-support-systems-oss-in-telecom/)
3. Circles – "BSS/OSS & TaaS: How Digital Transformation is Reshaping Telecom Operations" — [https://circles.co/in-the-loop/oss-bss-telco-transformation](https://circles.co/in-the-loop/oss-bss-telco-transformation)
4. Salesforce – "What Is a Business Support System?" — [https://www.salesforce.com/communications/customer-service-software/bss-telecom/](https://www.salesforce.com/communications/customer-service-software/bss-telecom/)
5. Microsoft AI – "What Are OSS and BSS?" — [https://www.microsoft.com/en-us/ai/telecommunications/resources/discover-oss-bss-solutions](https://www.microsoft.com/en-us/ai/telecommunications/resources/discover-oss-bss-solutions)
6. Wavelo – "Telecom BSS & OSS Billing, Orchestration, Cloud Provisioning Software for CSPs" — [https://www.wavelo.com/](https://www.wavelo.com/)
7. Sunvizion – "Telecom OSS BSS Solutions" — [https://www.sunvizion.com/](https://www.sunvizion.com/)
8. EarnBill – "Difference Between BSS And OSS In Telecom" — [https://earnbill.com/difference-between-bss-and-oss-in-telecom/](https://earnbill.com/difference-between-bss-and-oss-in-telecom/)

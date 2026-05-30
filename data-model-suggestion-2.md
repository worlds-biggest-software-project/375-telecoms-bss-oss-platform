# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Telecoms BSS/OSS Platform · Created: 2026-05-26

## Philosophy

This model collapses the BSS/OSS domain into five core tables by using JSONB columns to aggregate related data that would otherwise require separate tables. Each table represents a primary aggregate boundary: `tenants` (operator configuration), `customers` (the full subscriber lifecycle including subscriptions, balances, orders, and billing in nested JSONB), `network_resources` (physical and logical inventory with embedded service instances, alarms, and performance data), `charging_events` (high-volume partitioned CDR stream), and `ai_suggestions`. An `audit_log` table completes the set.

The key insight is that telecoms operators query by subscriber or by network element — rarely across both simultaneously at the transactional level. By embedding subscription, billing, and order data inside the customer aggregate and embedding service instances, alarms, and performance data inside the network resource aggregate, the most common operational queries (subscriber 360° view, network element status dashboard) become single-row reads with no JOINs. The BSS-OSS link is maintained through reference IDs in both directions (customer → service_instance_ids, resource → subscriber_ids).

This approach is ideal for MVNOs and greenfield operators who need rapid time-to-market. The flexible JSONB structure accommodates jurisdiction-specific billing rules, varied network equipment configurations, and rapid product iteration without schema migrations. TM Forum API compliance is achieved through API-layer transformation rather than direct table mapping.

**Best for:** MVNOs and digital-first operators prioritising development speed, flexible product configuration, and minimal schema migration overhead.

**Trade-offs:**
- Pro: Subscriber 360° view is a single-row read — no JOINs across 6+ tables
- Pro: Network element status with alarms and services is a single-row read
- Pro: New product types, billing rules, or equipment types require no schema changes
- Pro: Jurisdiction-specific fields (tax rules, regulatory identifiers) live in JSONB without column explosion
- Con: Cross-domain analytical queries (all customers affected by a specific alarm type) require JSONB extraction
- Con: JSONB fields bypass CHECK constraints — validation must happen at the application layer
- Con: Large customer aggregates (enterprise accounts with thousands of subscriptions) may hit row-size limits
- Con: TMF API entity boundaries don't align 1:1 with tables — API layer must decompose aggregates

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| TMF620 (Product Catalogue) | Product definitions stored in `customers.subscriptions_json[].product` — API layer decomposes to TMF620 entities |
| TMF622 (Product Ordering) | Order lifecycle embedded in `customers.orders_json[]` — API maps to TMF622 ProductOrder |
| TMF638 (Service Inventory) | Service instances embedded in `network_resources.services_json[]` — API maps to TMF638 Resource |
| TMF641 (Service Ordering) | Service provisioning tracked in `customers.orders_json[].provisioning` — API maps to TMF641 |
| 3GPP TS 32.290/32.291 | `charging_events` table models Nchf sessions with all 3GPP charging fields |
| 3GPP CCS Architecture | Balance management embedded in `customers.balances_json` per ABMF pattern |
| ETSI NFV MANO | VNF/CNF lifecycle fields in `network_resources.nfv_json` |
| ETSI ZSM | Slice management embedded in `network_resources` for slice-type resources |
| GSMA Open Gateway / CAMARA | API capability exposure tracked in `customers.subscriptions_json[].camara_apis` |
| ODA Component Model | Aggregate boundaries loosely align with ODA blocks: customers ↔ Party/Commerce, resources ↔ Production |
| GDPR | Consent and anonymisation fields in `customers` top level; erasure is a single-row update |
| ISO 27001 | `audit_log` with tenant isolation and correlation tracking |

---

## Tenant Configuration

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    tenant_type     TEXT NOT NULL CHECK (tenant_type IN (
                        'mvno', 'regional_carrier', 'digital_operator',
                        'enterprise_reseller', 'iot_operator', 'wholesale'
                    )),
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                        'provisioning', 'active', 'suspended', 'terminated'
                    )),
    country_code    TEXT NOT NULL,
    data_residency  TEXT NOT NULL,
    -- All operator config in JSONB
    identity_json   JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "mcc_mnc": "234-15",
    --   "mvno_type": "full",
    --   "host_operator": "Vodafone UK",
    --   "telecom_licence": "UK-OFCOM-2024-1234",
    --   "tax_id": "GB123456789"
    -- }
    billing_json    JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "currency": "GBP",
    --   "billing_cycle_day": 1,
    --   "invoice_format": "pdf",
    --   "tax_regime": "uk_vat",
    --   "tax_rates": [{"name": "VAT", "rate": 0.20, "applies_to": ["recurring", "usage"]}],
    --   "dunning_policy": {"levels": [7, 14, 28], "actions": ["reminder", "warning", "suspend"]},
    --   "payment_methods": ["direct_debit", "card", "topup_voucher"]
    -- }
    catalogue_json  JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"id": "prod-001", "name": "Essential 10GB", "type": "mobile_bundle", "status": "active",
    --    "pricing": {"recurring_cents": 1499, "period": "month"},
    --    "allowances": {"data_mb": 10240, "voice_min": 300, "sms": 500},
    --    "charging": {"rating_group": 100, "service_id": "data-4g-5g", "type": "convergent"},
    --    "provisioning": {"apn": "internet", "qos": "default"},
    --    "channels": ["web", "app", "call_centre"],
    --    "valid_from": "2026-01-01", "valid_to": null},
    --   {"id": "prod-002", "name": "Business Unlimited", "type": "enterprise_mpls", ...}
    -- ]
    network_json    JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "oda_components": ["party_management", "core_commerce", "production"],
    --   "kafka_topics": {"charging": "tenant-abc.charging", "provisioning": "tenant-abc.provisioning"},
    --   "slice_types_enabled": ["embb", "urllc"],
    --   "camara_apis_enabled": ["device-location", "number-verification"]
    -- }
    users_json      JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"id": "uuid", "email": "admin@mvno.com", "name": "Jane Ops", "role": "admin",
    --    "status": "active", "idp": "okta", "mfa": true, "last_login": "2026-05-25T18:00:00Z"}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_status ON tenants (status);
CREATE INDEX idx_tenants_country ON tenants (country_code);
CREATE INDEX idx_tenants_catalogue ON tenants USING GIN (catalogue_json);
```

## Customer Aggregate

```sql
CREATE TABLE customers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    external_id         TEXT,
    customer_type       TEXT NOT NULL CHECK (customer_type IN (
                            'individual', 'enterprise', 'wholesale',
                            'iot_fleet', 'government', 'internal'
                        )),
    status              TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                            'prospect', 'active', 'suspended', 'terminated',
                            'churned', 'anonymised'
                        )),
    -- Identity
    identity_json       JSONB NOT NULL DEFAULT '{}',
    -- Example (individual): {
    --   "first_name": "Alice", "last_name": "Smith",
    --   "email": "alice@example.com", "phone": "+447700900123",
    --   "dob": "1990-03-15",
    --   "billing_address": {"line1": "10 Fleet St", "city": "London", "postcode": "EC4Y 1AU", "country": "GB"},
    --   "identity_verified": true, "identity_method": "kyc_document",
    --   "credit_class": "standard", "credit_limit_cents": 50000
    -- }
    -- Example (enterprise): {
    --   "company_name": "Acme Corp", "registration_number": "12345678",
    --   "tax_id": "GB999999973", "contact_email": "telecom@acme.com",
    --   "billing_address": {...},
    --   "credit_class": "enterprise", "credit_limit_cents": 5000000,
    --   "account_manager": "uuid"
    -- }
    account_type        TEXT NOT NULL CHECK (account_type IN (
                            'prepaid', 'postpaid', 'hybrid', 'wholesale'
                        )),
    -- Subscriptions (the core BSS aggregate)
    subscriptions_json  JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {
    --     "id": "sub-001", "product_id": "prod-001", "product_name": "Essential 10GB",
    --     "status": "active", "msisdn": "+447700900123", "imsi": "234150999999999",
    --     "iccid": "8944...", "imei": "35...",
    --     "account_type": "postpaid", "billing_cycle_day": 1,
    --     "allowances": {
    --       "data_mb": {"total": 10240, "used": 3072, "reset_date": "2026-06-01"},
    --       "voice_min": {"total": 300, "used": 45},
    --       "sms": {"total": 500, "used": 12}
    --     },
    --     "rating_group": 100, "apn": "internet", "qos_profile": "default",
    --     "roaming_profile": "eu_included",
    --     "network_slice_id": null,
    --     "service_instance_ids": ["si-uuid-1", "si-uuid-2"],
    --     "activated_at": "2026-01-15T10:00:00Z",
    --     "contract_end_date": "2027-01-14", "auto_renew": true,
    --     "camara_apis": ["device-location"]
    --   }
    -- ]
    -- Balances (3GPP ABMF)
    balances_json       JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"type": "monetary", "unit": "cents", "currency": "GBP", "current": 0, "reserved": 0, "credit_limit": 50000},
    --   {"type": "data", "unit": "megabyte", "current": 7168, "valid_to": "2026-06-01T00:00:00Z"},
    --   {"type": "promotional_credit", "unit": "cents", "currency": "GBP", "current": 500, "valid_to": "2026-07-01"}
    -- ]
    -- Orders
    orders_json         JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {
    --     "id": "ord-001", "type": "new_subscription", "status": "completed",
    --     "channel": "web", "items": [{"product_id": "prod-001", "action": "add"}],
    --     "provisioning": {
    --       "steps": [
    --         {"step": 1, "action": "allocate_msisdn", "status": "completed"},
    --         {"step": 2, "action": "provision_hlr", "status": "completed"},
    --         {"step": 3, "action": "activate_sim", "status": "completed"}
    --       ]
    --     },
    --     "fulfilment": {"msisdn": "+447700900123", "imsi": "234150999999999"},
    --     "created_at": "2026-01-14T14:30:00Z", "completed_at": "2026-01-15T10:00:00Z"
    --   }
    -- ]
    -- Billing
    billing_json        JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "billing_cycle_day": 1,
    --   "payment_method": "direct_debit",
    --   "payment_reference": "DD-BACS-12345",
    --   "current_period": {
    --     "start": "2026-05-01", "end": "2026-05-31",
    --     "recurring_cents": 1499, "usage_cents": 230, "tax_cents": 346, "total_cents": 2075
    --   },
    --   "invoices": [
    --     {"id": "inv-001", "number": "INV-2026-04-001", "period": "2026-04",
    --      "total_cents": 1799, "status": "paid", "paid_at": "2026-05-03"}
    --   ],
    --   "outstanding_cents": 0,
    --   "dunning_level": 0
    -- }
    -- GDPR
    gdpr_json           JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "consent": {"marketing": true, "analytics": false, "third_party": false, "date": "2026-01-14"},
    --   "data_retention_until": "2028-01-14T00:00:00Z",
    --   "erasure_requests": [],
    --   "portability_exports": []
    -- }
    anonymised_at       TIMESTAMPTZ,
    -- Engagement & AI
    engagement_json     JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "acquisition_channel": "web",
    --   "segment": "young_professional",
    --   "churn_risk_score": 0.15,
    --   "lifetime_value_cents": 180000,
    --   "nps_score": 8,
    --   "last_contact": "2026-05-20T09:00:00Z",
    --   "tags": ["loyal", "data_heavy", "5g_early_adopter"]
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customers_tenant ON customers (tenant_id);
CREATE INDEX idx_customers_status ON customers (tenant_id, status);
CREATE INDEX idx_customers_type ON customers (tenant_id, customer_type);
CREATE INDEX idx_customers_account ON customers (tenant_id, account_type);
CREATE INDEX idx_customers_external ON customers (tenant_id, external_id);
CREATE INDEX idx_customers_subscriptions ON customers USING GIN (subscriptions_json);
CREATE INDEX idx_customers_engagement ON customers USING GIN (engagement_json);
CREATE INDEX idx_customers_billing ON customers USING GIN (billing_json);
```

## Network Resource Aggregate

```sql
CREATE TABLE network_resources (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    resource_type       TEXT NOT NULL CHECK (resource_type IN (
                            'site', 'rack', 'router', 'switch', 'olt', 'ont',
                            'bts', 'enodeb', 'gnodeb', 'antenna',
                            'fibre_segment', 'vnf', 'cnf', 'network_function',
                            'radio_cell', 'network_slice',
                            'amf', 'smf', 'upf', 'nrf', 'nssf', 'pcf', 'chf'
                        )),
    name                TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'planned' CHECK (status IN (
                            'planned', 'installing', 'available', 'in_service',
                            'degraded', 'maintenance', 'retired'
                        )),
    parent_id           UUID REFERENCES network_resources(id),
    -- Equipment identity & location
    equipment_json      JSONB NOT NULL DEFAULT '{}',
    -- Example (gNodeB): {
    --   "vendor": "Nokia", "model": "AirScale 5G BTS",
    --   "serial": "NK-2026-12345", "firmware": "24.3.1",
    --   "management_ip": "10.1.2.100",
    --   "location": {"site": "London-East-01", "lat": 51.5074, "lon": -0.1278, "address": "..."},
    --   "sectors": 3, "carriers": 2, "bandwidth_mhz": 100,
    --   "frequency_bands": ["n78", "n258"]
    -- }
    -- Capacity & performance
    capacity_json       JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "max_users": 1200, "current_users": 450,
    --   "throughput_gbps": {"max": 10, "current": 3.2},
    --   "ports": {"total": 48, "used": 32, "available": 16},
    --   "utilisation_pct": 37.5
    -- }
    -- NFV / cloud-native
    nfv_json            JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "descriptor_id": "vnfd-upf-001", "instance_id": "vnfi-upf-london-01",
    --   "orchestrator": "kubernetes",
    --   "cluster": "k8s-prod-east", "namespace": "5g-core",
    --   "replicas": {"desired": 3, "ready": 3},
    --   "etsi_lcm_state": "instantiated"
    -- }
    -- Service instances hosted on this resource
    services_json       JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {
    --     "id": "si-uuid-1", "type": "mobile_connectivity",
    --     "customer_id": "cust-uuid", "subscription_id": "sub-001",
    --     "status": "active",
    --     "params": {"msisdn": "+447700900123", "apn": "internet", "qos_class": 9},
    --     "sla": {"availability_pct": 99.9, "latency_ms": 20},
    --     "sla_status": "met",
    --     "provisioned_at": "2026-01-15T10:00:00Z"
    --   }
    -- ]
    -- Active alarms
    alarms_json         JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {
    --     "id": "alarm-uuid", "type": "high_cpu", "severity": "warning",
    --     "status": "active", "specific_problem": "CPU utilisation 87%",
    --     "raised_at": "2026-05-26T08:30:00Z",
    --     "affected_subscribers": 120,
    --     "affected_service_ids": ["si-uuid-1", "si-uuid-2"],
    --     "correlation_id": "corr-001"
    --   }
    -- ]
    -- 5G slice (for slice-type resources)
    slice_json          JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "s_nssai": "1-000001", "sst": 1, "sd": "000001",
    --   "slice_type": "embb",
    --   "sla": {"throughput_mbps": 1000, "latency_ms": 10, "reliability_pct": 99.99},
    --   "sla_status": "met",
    --   "billing_model": "sla_tiered",
    --   "max_subscribers": 5000, "current_subscribers": 1200,
    --   "resource_ids": ["uuid-upf", "uuid-smf", "uuid-gnb"],
    --   "zsm_ref": "zsm-domain-001",
    --   "activated_at": "2026-03-01T00:00:00Z"
    -- }
    -- Trouble tickets
    tickets_json        JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {
    --     "id": "tt-uuid", "number": "TT-2026-0042", "source": "alarm_auto",
    --     "status": "assigned", "priority": "p2_major",
    --     "summary": "High CPU on gNodeB London-East-01",
    --     "assigned_to": "user-uuid", "assigned_team": "noc",
    --     "reported_at": "2026-05-26T08:31:00Z",
    --     "sla_response_target": "30 minutes", "sla_resolve_target": "4 hours"
    --   }
    -- ]
    -- Health
    health_status       TEXT CHECK (health_status IN (
                            'healthy', 'warning', 'critical', 'unknown'
                        )),
    last_polled_at      TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_resources_tenant ON network_resources (tenant_id);
CREATE INDEX idx_resources_type ON network_resources (tenant_id, resource_type);
CREATE INDEX idx_resources_status ON network_resources (tenant_id, status);
CREATE INDEX idx_resources_parent ON network_resources (parent_id);
CREATE INDEX idx_resources_health ON network_resources (tenant_id, health_status);
CREATE INDEX idx_resources_services ON network_resources USING GIN (services_json);
CREATE INDEX idx_resources_alarms ON network_resources USING GIN (alarms_json);
CREATE INDEX idx_resources_equipment ON network_resources USING GIN (equipment_json);
CREATE INDEX idx_resources_slice ON network_resources USING GIN (slice_json);
```

## Charging Events (High Volume)

```sql
CREATE TABLE charging_events (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    customer_id         UUID NOT NULL,
    subscription_id     TEXT NOT NULL,          -- reference into customers.subscriptions_json[].id
    event_type          TEXT NOT NULL CHECK (event_type IN (
                            'voice_mo', 'voice_mt', 'voice_forward',
                            'sms_mo', 'sms_mt', 'mms',
                            'data_session', 'data_topup',
                            'content_purchase', 'roaming_voice', 'roaming_data',
                            'iot_event', 'api_call',
                            'recurring_charge', 'one_off_charge',
                            'adjustment', 'refund', 'balance_topup',
                            'slice_usage', 'wholesale_interconnect'
                        )),
    -- 3GPP Nchf fields
    nchf_json           JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "session_id": "nchf-sess-001",
    --   "operation": "update",
    --   "rating_group": 100,
    --   "service_id": "data-4g-5g",
    --   "charging_type": "convergent",
    --   "quota_granted_mb": 100,
    --   "quota_consumed_mb": 23.5
    -- }
    -- Usage
    usage_json          JSONB NOT NULL DEFAULT '{}',
    -- Example (data): {"bytes": 24641536, "duration_sec": 3600}
    -- Example (voice): {"a_party": "+447700900123", "b_party": "+447700900456", "duration_sec": 180}
    -- Charging result
    rated_amount_cents  BIGINT NOT NULL DEFAULT 0,
    currency            TEXT NOT NULL DEFAULT 'GBP',
    tax_cents           BIGINT NOT NULL DEFAULT 0,
    balance_before_cents BIGINT,
    balance_after_cents  BIGINT,
    -- Context
    location_json       JSONB NOT NULL DEFAULT '{}',
    -- Example: {"mcc_mnc": "234-15", "cell_id": "cell-london-e-03", "roaming": false}
    source_system       TEXT NOT NULL CHECK (source_system IN (
                            'mediation', 'ocs', 'ofcs', 'provisioning',
                            'partner_settlement', 'manual', 'api'
                        )),
    network_slice_id    TEXT,
    invoiced            BOOLEAN NOT NULL DEFAULT false,
    event_timestamp     TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (event_timestamp);

CREATE INDEX idx_charging_tenant ON charging_events (tenant_id);
CREATE INDEX idx_charging_customer ON charging_events (customer_id);
CREATE INDEX idx_charging_type ON charging_events (tenant_id, event_type);
CREATE INDEX idx_charging_event_time ON charging_events (event_timestamp DESC);
CREATE INDEX idx_charging_uninvoiced ON charging_events (tenant_id, invoiced) WHERE invoiced = false;
CREATE INDEX idx_charging_nchf ON charging_events USING GIN (nchf_json);
```

## AI Suggestions & Audit

```sql
CREATE TABLE ai_suggestions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    suggestion_type     TEXT NOT NULL CHECK (suggestion_type IN (
                            'churn_prediction', 'offer_personalisation',
                            'fraud_detection', 'anomaly_alert',
                            'root_cause_analysis', 'capacity_forecast',
                            'catalogue_generation', 'bill_adjustment',
                            'network_optimisation', 'slice_scaling',
                            'proactive_care', 'revenue_assurance'
                        )),
    status              TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                            'pending', 'accepted', 'rejected',
                            'auto_applied', 'expired'
                        )),
    confidence          REAL NOT NULL CHECK (confidence BETWEEN 0.0 AND 1.0),
    target_entity_type  TEXT NOT NULL CHECK (target_entity_type IN (
                            'customer', 'network_resource', 'tenant'
                        )),
    target_entity_id    UUID NOT NULL,
    summary             TEXT NOT NULL,
    detail_json         JSONB NOT NULL DEFAULT '{}',
    explanation         TEXT,
    model_id            TEXT,
    model_version       TEXT,
    reviewed_by         TEXT,
    reviewed_at         TIMESTAMPTZ,
    expires_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_tenant ON ai_suggestions (tenant_id);
CREATE INDEX idx_ai_type ON ai_suggestions (tenant_id, suggestion_type);
CREATE INDEX idx_ai_status ON ai_suggestions (tenant_id, status);
CREATE INDEX idx_ai_target ON ai_suggestions (target_entity_type, target_entity_id);

CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user', 'system', 'ai', 'api_client',
                            'kafka_consumer', 'scheduler', 'provisioning_engine',
                            'mediation', 'ocs', 'nfv_orchestrator'
                        )),
    actor_id            TEXT NOT NULL,
    action              TEXT NOT NULL,
    entity_type         TEXT NOT NULL,
    entity_id           UUID NOT NULL,
    changes_json        JSONB NOT NULL DEFAULT '{}',
    ip_address          TEXT,
    tmf_api             TEXT,
    oda_component       TEXT,
    gdpr_relevant       BOOLEAN NOT NULL DEFAULT false,
    correlation_id      TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_tenant ON audit_log (tenant_id);
CREATE INDEX idx_audit_actor ON audit_log (actor_type, actor_id);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log (created_at DESC);
CREATE INDEX idx_audit_gdpr ON audit_log (tenant_id) WHERE gdpr_relevant = true;
```

---

## Aggregate Query Examples

### Subscriber 360° view (single-row read)

```sql
SELECT identity_json, account_type, subscriptions_json, balances_json,
       orders_json, billing_json, engagement_json, gdpr_json
FROM customers
WHERE tenant_id = $1 AND id = $2;
```

### Find all customers with churn risk above threshold

```sql
SELECT id, identity_json->>'email' AS email,
       (engagement_json->>'churn_risk_score')::REAL AS churn_risk,
       subscriptions_json
FROM customers
WHERE tenant_id = $1
  AND status = 'active'
  AND (engagement_json->>'churn_risk_score')::REAL > 0.7
ORDER BY churn_risk DESC;
```

### Network element status dashboard (single-row read)

```sql
SELECT name, resource_type, status, health_status,
       equipment_json, capacity_json, services_json, alarms_json, tickets_json
FROM network_resources
WHERE tenant_id = $1 AND id = $2;
```

### All network elements with active critical alarms

```sql
SELECT id, name, resource_type, alarms_json
FROM network_resources
WHERE tenant_id = $1
  AND alarms_json @> '[{"severity": "critical", "status": "active"}]';
```

### Cross-domain: subscribers affected by degraded resources

```sql
SELECT nr.name AS resource_name,
       jsonb_array_length(nr.services_json) AS service_count,
       nr.alarms_json
FROM network_resources nr
WHERE nr.tenant_id = $1
  AND nr.health_status IN ('warning', 'critical')
  AND jsonb_array_length(nr.services_json) > 0;
-- Application layer resolves customer_ids from services_json[].customer_id
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant Configuration | 1 | Operator config, product catalogue, users all in JSONB |
| Customer Aggregate | 1 | Full subscriber lifecycle: identity, subscriptions, balances, orders, billing, GDPR |
| Network Resource Aggregate | 1 | Equipment, services, alarms, trouble tickets, slice config all embedded |
| Charging Events | 1 | Partitioned high-volume CDR stream with Nchf fields |
| AI & Audit | 2 | Cross-domain AI suggestions + partitioned audit log |
| **Total** | **6** | |

---

## Key Design Decisions

1. **Customer as the BSS aggregate root** — subscriptions, balances, orders, billing, and GDPR consent are all embedded within the customer row because the overwhelming majority of BSS operations (subscriber lookup, balance check, order status, invoice query) target a single customer. This eliminates the 6-table JOIN that a normalized model requires for a 360° view.

2. **Network resource as the OSS aggregate root** — service instances, active alarms, trouble tickets, and slice configuration are embedded within the network resource row because NOC operators primarily work resource-by-resource. A single GET returns the full operational picture for any network element.

3. **Product catalogue in tenant JSONB** — for MVNOs with 10-50 products, the catalogue fits comfortably in a JSONB array on the tenant row. This avoids a separate table and enables atomic catalogue updates (replace the entire array in a transaction). For operators with 500+ products, this would need promotion to a separate table.

4. **Charging events remain a separate partitioned table** — CDR volume (millions per day even for small MVNOs) makes embedding impractical. The `charging_events` table is the one high-volume write path that must remain independent for partitioning and archival.

5. **Cross-aggregate references by ID** — `customers.subscriptions_json[].service_instance_ids` references into `network_resources.services_json[].id`, and vice versa. The application layer resolves these references; the database does not enforce referential integrity across aggregates. This is the explicit trade-off for read performance.

6. **GDPR erasure is a single-row operation** — anonymising a customer means updating one row (nullify `identity_json`, set `anonymised_at`). No cascading updates across subscription, order, and billing tables.

7. **TMF API decomposition at the application layer** — TMF620 ProductOffering maps to an element of `tenants.catalogue_json[]`, TMF622 ProductOrder maps to an element of `customers.orders_json[]`, TMF638 Resource maps to `network_resources` top-level fields, and TMF641 Service maps to `network_resources.services_json[]`. The API layer performs this projection.

8. **Alarm correlation across resources** — `alarms_json[].correlation_id` groups related alarms across multiple network resources. The application layer performs correlation analysis by querying all resources with the same correlation_id using a GIN index containment query.

9. **5G slice as a resource type** — rather than a separate table, network slices are modeled as `network_resources` with `resource_type = 'network_slice'` and rich `slice_json` containing S-NSSAI, SLA, billing model, and subscriber counts. This keeps the resource graph unified.

10. **Balance management in customer JSONB** — follows the 3GPP ABMF pattern with multiple balance buckets (monetary, data, voice, promotional) as a JSONB array. Online charging quota operations update the relevant bucket atomically using JSONB path operations.

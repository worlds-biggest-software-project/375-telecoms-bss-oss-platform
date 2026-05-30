# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Telecoms BSS/OSS Platform · Created: 2026-05-26

## Philosophy

This model assigns a dedicated table to every core domain concept — tenants, customers, products, orders, subscriptions, charging records, invoices, network resources, service instances, alarms, trouble tickets, and network slices — connected by foreign keys that enforce referential integrity across the full BSS-OSS boundary. The design aligns directly with TM Forum Open API entity models (TMF620 Product Catalogue, TMF622 Product Ordering, TMF638 Service Inventory, TMF641 Service Ordering) so that API resources map one-to-one to database tables, minimising the impedance mismatch between API contract and persistence layer.

Convergent charging follows the 3GPP CCS architecture (TS 32.290/32.291), with separate tables for real-time charging records and billing invoices, and a dedicated balance table supporting prepaid, postpaid, and hybrid accounts on a single platform. Network inventory covers both physical and logical resources in a single self-referencing table with type discrimination, enabling service-to-resource linking as specified by TMF638. Multi-tenancy is enforced at the row level with a `tenant_id` foreign key on every operational table, supporting MVNO and regional operator isolation.

The normalized structure makes cross-domain queries natural — linking a network fault (alarm) through service instance and subscription to a specific customer and their billing impact requires only standard JOINs, not denormalized aggregation. This is the safest choice for teams with strong relational database skills who need auditable, standards-compliant data integrity.

**Best for:** Operators requiring strict TM Forum API compliance, regulatory auditability, and clear BSS-OSS data lineage.

**Trade-offs:**
- Pro: Direct mapping to TM Forum Open API entity models; straightforward API implementation
- Pro: Referential integrity enforced at the database level across BSS and OSS domains
- Pro: Cross-domain JOINs (fault → service → subscription → customer → billing) are natural
- Pro: GDPR data subject operations (erasure, portability) target well-defined tables
- Con: 15 tables with foreign keys increases migration complexity and JOIN depth for reporting queries
- Con: Schema changes require coordinated migrations across related tables
- Con: High-throughput charging (millions of CDRs/hour) may stress the normalized structure without partitioning

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| TMF620 (Product Catalogue) | `products` table structure mirrors TMF620 ProductOffering entity with lifecycle status, validity periods, and channel distribution |
| TMF622 (Product Ordering) | `orders` table follows TMF622 ProductOrder lifecycle states and item structure |
| TMF638 (Service Inventory) | `network_resources` table models physical and logical resources per TMF638 Resource entity |
| TMF641 (Service Ordering) | `service_instances` table links services to resources per TMF641 ServiceOrder fulfilment |
| 3GPP TS 32.290/32.291 | `charging_records` table models Nchf sessions with rating groups, quota grants, and convergent charging types |
| 3GPP CCS Architecture | `balances` table implements ABMF (Account and Balance Management Function) with prepaid/postpaid/hybrid support |
| ETSI NFV MANO | `network_resources` includes VNF/CNF lifecycle fields; `service_instances` tracks virtualised service components |
| ETSI ZSM | `network_slices` models slice lifecycle per ZSM closed-loop automation principles |
| GSMA Open Gateway / CAMARA | `products` supports API-as-product exposure; `service_instances` tracks CAMARA capability subscriptions |
| ODA Component Model | Table boundaries align with ODA functional blocks: Party Management, Core Commerce, Production |
| ISO 27001 | `audit_log` provides security event trail; `users` tracks authentication and role changes |
| GDPR | `customers` includes consent tracking and anonymisation flags; charging records support retention policies |
| OAuth 2.0 / OIDC | `users` stores identity provider references and role assignments per tenant |

---

## Tenant & Identity Management

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
    country_code    TEXT NOT NULL,              -- ISO 3166-1 alpha-2
    data_residency  TEXT NOT NULL,              -- region where subscriber data must reside
    mcc_mnc         TEXT,                       -- Mobile Country Code + Mobile Network Code
    mvno_type       TEXT CHECK (mvno_type IN (
                        'full', 'light', 'branded_reseller', 'esp'
                    )),
    host_operator   TEXT,                       -- MNO providing network access
    regulatory_ids  JSONB NOT NULL DEFAULT '{}',
    -- Example: {"telecom_licence": "UK-OFCOM-2024-1234", "tax_id": "GB123456789"}
    billing_config  JSONB NOT NULL DEFAULT '{}',
    -- Example: {"currency": "GBP", "billing_cycle_day": 1, "invoice_format": "pdf", "tax_regime": "uk_vat"}
    oda_components  TEXT[] NOT NULL DEFAULT '{}', -- ODA functional blocks enabled
    kafka_topics    JSONB NOT NULL DEFAULT '{}',
    -- Example: {"charging": "tenant-abc.charging", "provisioning": "tenant-abc.provisioning"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_status ON tenants (status);
CREATE INDEX idx_tenants_country ON tenants (country_code);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN (
                        'admin', 'billing_manager', 'catalogue_manager',
                        'order_manager', 'network_engineer', 'noc_operator',
                        'care_agent', 'field_technician', 'partner_manager',
                        'finance_analyst', 'compliance_officer', 'api_developer',
                        'ai_operator', 'read_only'
                    )),
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                        'active', 'suspended', 'deactivated'
                    )),
    idp_provider    TEXT,                       -- OAuth 2.0 / OIDC provider
    idp_subject     TEXT,                       -- external identity reference
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"can_approve_orders": true, "max_credit_limit": 10000, "allowed_oda_blocks": ["commerce", "production"]}
    mfa_enabled     BOOLEAN NOT NULL DEFAULT false,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users (tenant_id);
CREATE INDEX idx_users_role ON users (tenant_id, role);
```

## Customer Management

```sql
CREATE TABLE customers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    external_id         TEXT,                   -- CRM or legacy system reference
    customer_type       TEXT NOT NULL CHECK (customer_type IN (
                            'individual', 'enterprise', 'wholesale',
                            'iot_fleet', 'government', 'internal'
                        )),
    status              TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                            'prospect', 'active', 'suspended', 'terminated',
                            'churned', 'anonymised'
                        )),
    -- Individual fields
    first_name          TEXT,
    last_name           TEXT,
    date_of_birth       DATE,
    -- Enterprise fields
    company_name        TEXT,
    registration_number TEXT,
    tax_id              TEXT,
    -- Common fields
    email               TEXT,
    phone               TEXT,
    billing_address     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"line1": "10 Fleet St", "city": "London", "postcode": "EC4Y 1AU", "country": "GB"}
    identity_verified   BOOLEAN NOT NULL DEFAULT false,
    identity_method     TEXT CHECK (identity_method IN (
                            'kyc_document', 'credit_check', 'enterprise_registration',
                            'self_declared', 'operator_verified'
                        )),
    credit_class        TEXT CHECK (credit_class IN (
                            'prepaid_only', 'low', 'standard', 'premium', 'enterprise', 'unlimited'
                        )),
    credit_limit_cents  BIGINT,                -- in tenant billing currency
    account_type        TEXT NOT NULL CHECK (account_type IN (
                            'prepaid', 'postpaid', 'hybrid', 'wholesale'
                        )),
    -- GDPR / data privacy
    gdpr_consent        JSONB NOT NULL DEFAULT '{}',
    -- Example: {"marketing": true, "analytics": false, "third_party": false, "consent_date": "2026-01-15"}
    data_retention_until TIMESTAMPTZ,          -- computed from telecoms retention directive
    anonymised_at       TIMESTAMPTZ,
    -- Churn & engagement
    acquisition_channel TEXT,
    churn_risk_score    REAL,                  -- AI-computed 0.0-1.0
    lifetime_value_cents BIGINT,
    -- Segment
    segment             TEXT,
    tags                TEXT[] NOT NULL DEFAULT '{}',
    metadata            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customers_tenant ON customers (tenant_id);
CREATE INDEX idx_customers_status ON customers (tenant_id, status);
CREATE INDEX idx_customers_type ON customers (tenant_id, customer_type);
CREATE INDEX idx_customers_email ON customers (tenant_id, email);
CREATE INDEX idx_customers_segment ON customers (tenant_id, segment);
CREATE INDEX idx_customers_tags ON customers USING GIN (tags);
```

## Product Catalogue (TMF620)

```sql
CREATE TABLE products (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    tmf_product_id      TEXT,                   -- TMF620 ProductOffering.id
    name                TEXT NOT NULL,
    description         TEXT,
    product_type        TEXT NOT NULL CHECK (product_type IN (
                            'mobile_voice', 'mobile_data', 'mobile_bundle',
                            'broadband', 'iptv', 'ott', 'iot_connectivity',
                            'enterprise_mpls', 'sd_wan', 'ucaas',
                            'network_slice', 'api_product', 'add_on',
                            'device_bundle', 'roaming_pack', 'wholesale'
                        )),
    category            TEXT NOT NULL CHECK (category IN (
                            'consumer', 'enterprise', 'wholesale', 'iot', 'developer'
                        )),
    status              TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
                            'draft', 'test', 'active', 'sunset', 'retired'
                        )),
    version             INTEGER NOT NULL DEFAULT 1,
    valid_from          TIMESTAMPTZ,
    valid_to            TIMESTAMPTZ,
    channels            TEXT[] NOT NULL DEFAULT '{}',
    -- e.g. {'web', 'app', 'call_centre', 'partner', 'self_service', 'ndc'}
    pricing_json        JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "recurring": {"amount_cents": 2999, "currency": "GBP", "period": "month"},
    --   "one_off": {"amount_cents": 0},
    --   "usage": [
    --     {"unit": "mb", "rate_cents": 1, "included": 20480},
    --     {"unit": "minute", "rate_cents": 5, "included": 500}
    --   ],
    --   "overage": {"mb_rate_cents": 2, "minute_rate_cents": 8}
    -- }
    allowances_json     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"data_mb": 20480, "voice_minutes": 500, "sms": 1000, "roaming_zones": ["eu"]}
    charging_config     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"charging_type": "convergent", "rating_group": 100, "service_id": "data-4g-5g", "quota_threshold_pct": 80}
    provisioning_spec   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"apn": "internet", "qos_profile": "default", "network_slice_type": "eMBB", "activation_workflow": "zero_touch"}
    eligibility_rules   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"min_credit_class": "standard", "required_identity": true, "excluded_segments": ["prepaid_only"]}
    dependencies        UUID[] NOT NULL DEFAULT '{}', -- prerequisite product IDs
    camara_apis         TEXT[] NOT NULL DEFAULT '{}', -- CAMARA API capabilities exposed
    created_by          UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_products_tenant ON products (tenant_id);
CREATE INDEX idx_products_status ON products (tenant_id, status);
CREATE INDEX idx_products_type ON products (tenant_id, product_type);
CREATE INDEX idx_products_category ON products (tenant_id, category);
CREATE INDEX idx_products_channels ON products USING GIN (channels);
CREATE INDEX idx_products_valid ON products (tenant_id, valid_from, valid_to);
```

## Order Management (TMF622)

```sql
CREATE TABLE orders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    tmf_order_id        TEXT,                   -- TMF622 ProductOrder.id
    order_type          TEXT NOT NULL CHECK (order_type IN (
                            'new_subscription', 'modify', 'upgrade', 'downgrade',
                            'add_on', 'port_in', 'port_out', 'suspend',
                            'resume', 'terminate', 'device_swap',
                            'slice_provision', 'wholesale_activation',
                            'number_change', 'sim_swap'
                        )),
    status              TEXT NOT NULL DEFAULT 'acknowledged' CHECK (status IN (
                            'acknowledged', 'validated', 'in_progress',
                            'pending_provisioning', 'provisioning',
                            'pending_activation', 'activated',
                            'completed', 'failed', 'cancelled',
                            'rolled_back'
                        )),
    priority            TEXT NOT NULL DEFAULT 'normal' CHECK (priority IN (
                            'low', 'normal', 'high', 'critical'
                        )),
    channel             TEXT NOT NULL CHECK (channel IN (
                            'web', 'app', 'call_centre', 'retail_store',
                            'partner', 'self_service', 'api', 'bulk_import'
                        )),
    items_json          JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"product_id": "uuid", "action": "add", "quantity": 1, "pricing_override": null},
    --   {"product_id": "uuid", "action": "remove", "quantity": 1}
    -- ]
    provisioning_plan   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"steps": [
    --   {"step": 1, "action": "allocate_msisdn", "status": "completed"},
    --   {"step": 2, "action": "provision_hlr", "status": "in_progress"},
    --   {"step": 3, "action": "activate_sim", "status": "pending"}
    -- ]}
    fulfilment_json     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"msisdn": "+447700900123", "imsi": "234150999999999", "iccid": "8944..."}
    requested_date      TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    failed_reason       TEXT,
    rollback_order_id   UUID REFERENCES orders(id),
    created_by          UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orders_tenant ON orders (tenant_id);
CREATE INDEX idx_orders_customer ON orders (customer_id);
CREATE INDEX idx_orders_status ON orders (tenant_id, status);
CREATE INDEX idx_orders_type ON orders (tenant_id, order_type);
CREATE INDEX idx_orders_created ON orders (tenant_id, created_at DESC);
```

## Subscriptions & Charging

```sql
CREATE TABLE subscriptions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    product_id          UUID NOT NULL REFERENCES products(id),
    order_id            UUID REFERENCES orders(id),
    status              TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                            'pending_activation', 'active', 'suspended',
                            'grace_period', 'terminated', 'ported_out'
                        )),
    msisdn              TEXT,                   -- subscriber phone number (E.164)
    imsi                TEXT,
    iccid               TEXT,                   -- SIM card identifier
    imei                TEXT,                   -- device identifier
    account_type        TEXT NOT NULL CHECK (account_type IN (
                            'prepaid', 'postpaid', 'hybrid', 'wholesale'
                        )),
    billing_cycle_day   INTEGER CHECK (billing_cycle_day BETWEEN 1 AND 28),
    allowances_json     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"data_mb": {"total": 20480, "used": 5120, "reset_date": "2026-06-01"}, "voice_minutes": {"total": 500, "used": 123}}
    current_balance_cents BIGINT NOT NULL DEFAULT 0,
    rating_group        INTEGER,                -- 3GPP rating group for Nchf
    apn                 TEXT,
    qos_profile         TEXT,
    network_slice_id    UUID,                   -- FK to network_slices if slice-specific
    roaming_profile     TEXT CHECK (roaming_profile IN (
                            'domestic_only', 'eu_included', 'global', 'barred'
                        )),
    -- Service identifiers for OSS linking
    service_instance_ids UUID[] NOT NULL DEFAULT '{}',
    activated_at        TIMESTAMPTZ,
    suspended_at        TIMESTAMPTZ,
    terminated_at       TIMESTAMPTZ,
    contract_end_date   DATE,
    auto_renew          BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscriptions_tenant ON subscriptions (tenant_id);
CREATE INDEX idx_subscriptions_customer ON subscriptions (customer_id);
CREATE INDEX idx_subscriptions_product ON subscriptions (product_id);
CREATE INDEX idx_subscriptions_status ON subscriptions (tenant_id, status);
CREATE INDEX idx_subscriptions_msisdn ON subscriptions (tenant_id, msisdn);
CREATE INDEX idx_subscriptions_imsi ON subscriptions (tenant_id, imsi);

CREATE TABLE charging_records (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    subscription_id     UUID NOT NULL REFERENCES subscriptions(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    record_type         TEXT NOT NULL CHECK (record_type IN (
                            'voice_mo', 'voice_mt', 'voice_forward',
                            'sms_mo', 'sms_mt', 'mms',
                            'data_session', 'data_topup',
                            'content_purchase', 'roaming_voice', 'roaming_data',
                            'iot_event', 'api_call',
                            'recurring_charge', 'one_off_charge',
                            'adjustment', 'refund', 'balance_topup',
                            'slice_usage', 'wholesale_interconnect'
                        )),
    -- 3GPP Nchf fields (TS 32.290)
    nchf_session_id     TEXT,                   -- Nchf charging session reference
    nchf_operation      TEXT CHECK (nchf_operation IN (
                            'initial', 'update', 'terminate'
                        )),
    rating_group        INTEGER,
    service_id          TEXT,                   -- 3GPP service identifier
    -- Usage
    usage_quantity      BIGINT,                 -- bytes, seconds, messages, API calls
    usage_unit          TEXT CHECK (usage_unit IN (
                            'byte', 'kilobyte', 'megabyte', 'gigabyte',
                            'second', 'minute', 'message', 'event', 'api_call'
                        )),
    -- Charging
    rated_amount_cents  BIGINT NOT NULL DEFAULT 0,
    currency            TEXT NOT NULL DEFAULT 'GBP',
    tax_amount_cents    BIGINT NOT NULL DEFAULT 0,
    charging_type       TEXT NOT NULL CHECK (charging_type IN (
                            'online', 'offline', 'convergent'
                        )),
    quota_granted       BIGINT,                 -- for online charging quota management
    quota_consumed      BIGINT,
    balance_before_cents BIGINT,
    balance_after_cents  BIGINT,
    -- Context
    event_timestamp     TIMESTAMPTZ NOT NULL,
    a_party             TEXT,                   -- calling party (E.164)
    b_party             TEXT,                   -- called party (E.164)
    location_mcc_mnc    TEXT,                   -- serving network for roaming
    cell_id             TEXT,
    roaming_indicator   BOOLEAN NOT NULL DEFAULT false,
    network_slice_id    UUID,
    -- Source
    source_system       TEXT NOT NULL CHECK (source_system IN (
                            'mediation', 'ocs', 'ofcs', 'provisioning',
                            'partner_settlement', 'manual', 'api'
                        )),
    source_file         TEXT,                   -- CDR file reference
    rated_at            TIMESTAMPTZ,
    invoiced            BOOLEAN NOT NULL DEFAULT false,
    invoice_id          UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (event_timestamp);

CREATE INDEX idx_charging_tenant ON charging_records (tenant_id);
CREATE INDEX idx_charging_subscription ON charging_records (subscription_id);
CREATE INDEX idx_charging_customer ON charging_records (customer_id);
CREATE INDEX idx_charging_type ON charging_records (tenant_id, record_type);
CREATE INDEX idx_charging_session ON charging_records (nchf_session_id);
CREATE INDEX idx_charging_event_time ON charging_records (event_timestamp DESC);
CREATE INDEX idx_charging_uninvoiced ON charging_records (tenant_id, invoiced) WHERE invoiced = false;
```

## Billing

```sql
CREATE TABLE invoices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    invoice_number      TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
                            'draft', 'generated', 'sent', 'paid',
                            'partially_paid', 'overdue', 'disputed',
                            'credited', 'written_off'
                        )),
    invoice_type        TEXT NOT NULL CHECK (invoice_type IN (
                            'recurring', 'one_off', 'adjustment',
                            'credit_note', 'final', 'wholesale_settlement'
                        )),
    billing_period_start DATE NOT NULL,
    billing_period_end   DATE NOT NULL,
    currency            TEXT NOT NULL DEFAULT 'GBP',
    subtotal_cents      BIGINT NOT NULL DEFAULT 0,
    tax_cents           BIGINT NOT NULL DEFAULT 0,
    total_cents         BIGINT NOT NULL DEFAULT 0,
    paid_cents          BIGINT NOT NULL DEFAULT 0,
    outstanding_cents   BIGINT NOT NULL DEFAULT 0,
    line_items_json     JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"description": "Mobile Bundle 20GB", "type": "recurring", "amount_cents": 2999},
    --   {"description": "Data overage 2.3GB", "type": "usage", "amount_cents": 460},
    --   {"description": "Roaming EU - 45min voice", "type": "usage", "amount_cents": 0}
    -- ]
    tax_breakdown_json  JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"jurisdiction": "GB", "rate": 0.20, "taxable_cents": 3459, "tax_cents": 692}]
    payment_method      TEXT,
    payment_reference   TEXT,
    paid_at             TIMESTAMPTZ,
    due_date            DATE NOT NULL,
    sent_at             TIMESTAMPTZ,
    dunning_level       INTEGER NOT NULL DEFAULT 0,
    dispute_reason      TEXT,
    credit_note_for     UUID REFERENCES invoices(id),
    pdf_url             TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, invoice_number)
);

CREATE INDEX idx_invoices_tenant ON invoices (tenant_id);
CREATE INDEX idx_invoices_customer ON invoices (customer_id);
CREATE INDEX idx_invoices_status ON invoices (tenant_id, status);
CREATE INDEX idx_invoices_due ON invoices (tenant_id, due_date) WHERE status IN ('sent', 'overdue');
```

## Balance Management (3GPP ABMF)

```sql
CREATE TABLE balances (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    customer_id         UUID NOT NULL REFERENCES customers(id),
    balance_type        TEXT NOT NULL CHECK (balance_type IN (
                            'monetary', 'data', 'voice', 'sms',
                            'loyalty_points', 'promotional_credit'
                        )),
    unit                TEXT NOT NULL CHECK (unit IN (
                            'cents', 'megabyte', 'minute', 'message', 'point'
                        )),
    currency            TEXT,                   -- for monetary balances
    current_value       BIGINT NOT NULL DEFAULT 0,
    reserved_value      BIGINT NOT NULL DEFAULT 0,  -- quota reserved by active sessions
    credit_limit        BIGINT,                 -- for postpaid monetary balances
    threshold_alert_pct INTEGER,                -- notify when balance drops below this %
    valid_from          TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to            TIMESTAMPTZ,            -- expiry for promotional/topup balances
    last_charged_at     TIMESTAMPTZ,
    last_topup_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_balances_tenant ON balances (tenant_id);
CREATE INDEX idx_balances_customer ON balances (customer_id);
CREATE INDEX idx_balances_type ON balances (customer_id, balance_type);
CREATE INDEX idx_balances_expiring ON balances (valid_to) WHERE valid_to IS NOT NULL;
```

## Network Inventory (TMF638) & Service Instances (TMF641)

```sql
CREATE TABLE network_resources (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    parent_id           UUID REFERENCES network_resources(id),
    resource_type       TEXT NOT NULL CHECK (resource_type IN (
                            -- Physical
                            'site', 'rack', 'router', 'switch', 'olt',
                            'ont', 'bts', 'enodeb', 'gnodeb', 'antenna',
                            'sim_card_batch', 'fibre_segment', 'cable',
                            'power_supply', 'cooling_unit',
                            -- Logical
                            'vlan', 'vpn', 'ip_subnet', 'apn', 'bearer',
                            'vnf', 'cnf', 'network_function',
                            'radio_cell', 'frequency_band',
                            -- 5G specific
                            'amf', 'smf', 'upf', 'nrf', 'nssf', 'pcf', 'chf'
                        )),
    name                TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'planned' CHECK (status IN (
                            'planned', 'installing', 'available', 'reserved',
                            'in_service', 'degraded', 'maintenance',
                            'decommissioning', 'retired'
                        )),
    -- Location
    site_name           TEXT,
    latitude            DOUBLE PRECISION,
    longitude           DOUBLE PRECISION,
    address             TEXT,
    -- Identifiers
    serial_number       TEXT,
    vendor              TEXT,
    model               TEXT,
    firmware_version    TEXT,
    management_ip       TEXT,
    -- Capacity
    capacity_json       JSONB NOT NULL DEFAULT '{}',
    -- Example (gNodeB): {"sectors": 3, "carriers": 2, "bandwidth_mhz": 100, "max_users": 1200}
    -- Example (router): {"ports_total": 48, "ports_used": 32, "throughput_gbps": 400}
    -- NFV / cloud-native
    nfv_descriptor_id   TEXT,                   -- ETSI NFV VNF/CNF descriptor reference
    nfv_instance_id     TEXT,
    orchestrator        TEXT,                   -- kubernetes_cluster, openstack, vmware
    -- Monitoring
    health_status       TEXT CHECK (health_status IN (
                            'healthy', 'warning', 'critical', 'unknown'
                        )),
    last_polled_at      TIMESTAMPTZ,
    alarm_count         INTEGER NOT NULL DEFAULT 0,
    metadata            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_resources_tenant ON network_resources (tenant_id);
CREATE INDEX idx_resources_parent ON network_resources (parent_id);
CREATE INDEX idx_resources_type ON network_resources (tenant_id, resource_type);
CREATE INDEX idx_resources_status ON network_resources (tenant_id, status);
CREATE INDEX idx_resources_health ON network_resources (tenant_id, health_status);
CREATE INDEX idx_resources_mgmt_ip ON network_resources (management_ip);
CREATE INDEX idx_resources_metadata ON network_resources USING GIN (metadata);

CREATE TABLE service_instances (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    customer_id         UUID REFERENCES customers(id),
    subscription_id     UUID REFERENCES subscriptions(id),
    tmf_service_id      TEXT,                   -- TMF641 Service.id
    service_type        TEXT NOT NULL CHECK (service_type IN (
                            'mobile_connectivity', 'broadband', 'iptv',
                            'voip', 'vpn', 'sd_wan', 'iot_connectivity',
                            'network_slice_instance', 'api_subscription',
                            'cloud_communication', 'enterprise_trunk',
                            'wholesale_interconnect'
                        )),
    status              TEXT NOT NULL DEFAULT 'designed' CHECK (status IN (
                            'designed', 'reserved', 'provisioning',
                            'active', 'suspended', 'terminating', 'terminated'
                        )),
    -- Resource binding
    resource_ids        UUID[] NOT NULL DEFAULT '{}',
    resource_mapping    JSONB NOT NULL DEFAULT '{}',
    -- Example: {"radio_cell": "uuid", "upf": "uuid", "apn": "internet", "bearer_id": "uuid"}
    -- Service parameters
    service_params      JSONB NOT NULL DEFAULT '{}',
    -- Example (mobile): {"msisdn": "+447700900123", "apn": "internet", "qos_class": 9, "ambr_dl_mbps": 100, "ambr_ul_mbps": 50}
    -- SLA
    sla_json            JSONB NOT NULL DEFAULT '{}',
    -- Example: {"availability_pct": 99.9, "latency_ms": 20, "jitter_ms": 5, "packet_loss_pct": 0.1}
    sla_status          TEXT CHECK (sla_status IN (
                            'met', 'at_risk', 'breached', 'not_measured'
                        )),
    -- Provisioning
    provisioned_at      TIMESTAMPTZ,
    provisioning_log    JSONB NOT NULL DEFAULT '[]',
    -- Monitoring
    last_performance    JSONB NOT NULL DEFAULT '{}',
    -- Example: {"throughput_mbps": 85.2, "latency_ms": 12, "packet_loss_pct": 0.01, "measured_at": "2026-05-26T10:00:00Z"}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_services_tenant ON service_instances (tenant_id);
CREATE INDEX idx_services_customer ON service_instances (customer_id);
CREATE INDEX idx_services_subscription ON service_instances (subscription_id);
CREATE INDEX idx_services_type ON service_instances (tenant_id, service_type);
CREATE INDEX idx_services_status ON service_instances (tenant_id, status);
CREATE INDEX idx_services_resources ON service_instances USING GIN (resource_ids);
```

## Fault Management

```sql
CREATE TABLE alarms (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    resource_id         UUID REFERENCES network_resources(id),
    service_instance_id UUID REFERENCES service_instances(id),
    alarm_type          TEXT NOT NULL CHECK (alarm_type IN (
                            'equipment_failure', 'link_down', 'link_degraded',
                            'high_cpu', 'high_memory', 'high_temperature',
                            'power_failure', 'radio_interference',
                            'capacity_threshold', 'packet_loss',
                            'latency_threshold', 'protocol_error',
                            'security_breach', 'configuration_error',
                            'vnf_failure', 'slice_degradation',
                            'environmental', 'customer_reported'
                        )),
    severity            TEXT NOT NULL CHECK (severity IN (
                            'critical', 'major', 'minor', 'warning', 'indeterminate'
                        )),
    status              TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                            'active', 'acknowledged', 'correlated',
                            'cleared', 'suppressed'
                        )),
    probable_cause      TEXT,
    specific_problem    TEXT NOT NULL,
    additional_info     JSONB NOT NULL DEFAULT '{}',
    -- Correlation
    correlation_id      UUID,                   -- parent alarm if correlated
    correlated_alarm_ids UUID[] NOT NULL DEFAULT '{}',
    root_cause_alarm_id UUID REFERENCES alarms(id),
    is_root_cause       BOOLEAN NOT NULL DEFAULT false,
    -- Impact
    affected_subscribers INTEGER NOT NULL DEFAULT 0,
    affected_services   UUID[] NOT NULL DEFAULT '{}',
    business_impact     TEXT CHECK (business_impact IN (
                            'revenue_loss', 'service_degradation',
                            'compliance_risk', 'safety_risk', 'none'
                        )),
    -- Trouble ticket
    trouble_ticket_id   UUID,
    -- Timeline
    raised_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    acknowledged_at     TIMESTAMPTZ,
    acknowledged_by     UUID REFERENCES users(id),
    cleared_at          TIMESTAMPTZ,
    auto_cleared        BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alarms_tenant ON alarms (tenant_id);
CREATE INDEX idx_alarms_resource ON alarms (resource_id);
CREATE INDEX idx_alarms_status ON alarms (tenant_id, status);
CREATE INDEX idx_alarms_severity ON alarms (tenant_id, severity) WHERE status = 'active';
CREATE INDEX idx_alarms_correlation ON alarms (correlation_id);
CREATE INDEX idx_alarms_raised ON alarms (tenant_id, raised_at DESC);
CREATE INDEX idx_alarms_affected ON alarms USING GIN (affected_services);

CREATE TABLE trouble_tickets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    ticket_number       TEXT NOT NULL,
    source              TEXT NOT NULL CHECK (source IN (
                            'alarm_auto', 'customer_reported', 'noc_manual',
                            'field_tech', 'partner', 'ai_detected', 'api'
                        )),
    status              TEXT NOT NULL DEFAULT 'open' CHECK (status IN (
                            'open', 'assigned', 'in_progress', 'pending_customer',
                            'pending_vendor', 'resolved', 'closed', 'reopened'
                        )),
    priority            TEXT NOT NULL CHECK (priority IN (
                            'p1_critical', 'p2_major', 'p3_minor', 'p4_cosmetic'
                        )),
    category            TEXT NOT NULL CHECK (category IN (
                            'network_fault', 'service_outage', 'billing_dispute',
                            'provisioning_failure', 'quality_complaint',
                            'security_incident', 'change_request'
                        )),
    -- Linked entities
    customer_id         UUID REFERENCES customers(id),
    alarm_ids           UUID[] NOT NULL DEFAULT '{}',
    service_instance_ids UUID[] NOT NULL DEFAULT '{}',
    -- Description
    summary             TEXT NOT NULL,
    description         TEXT,
    resolution_notes    TEXT,
    -- SLA tracking
    sla_target_response INTERVAL,
    sla_target_resolve  INTERVAL,
    sla_response_met    BOOLEAN,
    sla_resolve_met     BOOLEAN,
    -- Assignment
    assigned_to         UUID REFERENCES users(id),
    assigned_team       TEXT,
    escalation_level    INTEGER NOT NULL DEFAULT 0,
    -- Timeline
    reported_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    responded_at        TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    closed_at           TIMESTAMPTZ,
    work_log_json       JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"timestamp": "...", "user": "uuid", "action": "note", "text": "Replaced faulty SFP module"}]
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, ticket_number)
);

CREATE INDEX idx_tickets_tenant ON trouble_tickets (tenant_id);
CREATE INDEX idx_tickets_status ON trouble_tickets (tenant_id, status);
CREATE INDEX idx_tickets_priority ON trouble_tickets (tenant_id, priority) WHERE status NOT IN ('resolved', 'closed');
CREATE INDEX idx_tickets_customer ON trouble_tickets (customer_id);
CREATE INDEX idx_tickets_assigned ON trouble_tickets (assigned_to) WHERE status NOT IN ('resolved', 'closed');
CREATE INDEX idx_tickets_alarms ON trouble_tickets USING GIN (alarm_ids);
```

## 5G Network Slicing

```sql
CREATE TABLE network_slices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    slice_name          TEXT NOT NULL,
    s_nssai             TEXT NOT NULL,          -- Single Network Slice Selection Assistance Information
    sst                 INTEGER NOT NULL,       -- Slice/Service Type (1=eMBB, 2=URLLC, 3=mMTC)
    sd                  TEXT,                   -- Slice Differentiator (optional)
    status              TEXT NOT NULL DEFAULT 'designed' CHECK (status IN (
                            'designed', 'instantiating', 'active',
                            'modifying', 'deactivating', 'terminated'
                        )),
    slice_type          TEXT NOT NULL CHECK (slice_type IN (
                            'embb', 'urllc', 'mmtc', 'enterprise_private',
                            'iot_massive', 'gaming', 'automotive_v2x',
                            'telemedicine', 'custom'
                        )),
    -- SLA
    sla_json            JSONB NOT NULL DEFAULT '{}',
    -- Example: {"throughput_mbps": 1000, "latency_ms": 1, "reliability_pct": 99.999, "density_devices_km2": 1000000}
    sla_status          TEXT CHECK (sla_status IN (
                            'met', 'at_risk', 'breached', 'not_measured'
                        )),
    -- Resources
    resource_ids        UUID[] NOT NULL DEFAULT '{}', -- network resources allocated to this slice
    nssf_config         JSONB NOT NULL DEFAULT '{}',  -- NSSF configuration
    -- Billing
    billing_model       TEXT NOT NULL CHECK (billing_model IN (
                            'flat_rate', 'usage_based', 'sla_tiered',
                            'dynamic', 'enterprise_contract'
                        )),
    pricing_json        JSONB NOT NULL DEFAULT '{}',
    -- Subscribers
    max_subscribers     INTEGER,
    current_subscribers INTEGER NOT NULL DEFAULT 0,
    -- Lifecycle
    requested_by        UUID REFERENCES customers(id),
    approved_by         UUID REFERENCES users(id),
    activated_at        TIMESTAMPTZ,
    deactivated_at      TIMESTAMPTZ,
    etsi_zsm_ref        TEXT,                   -- ETSI ZSM management domain reference
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_slices_tenant ON network_slices (tenant_id);
CREATE INDEX idx_slices_status ON network_slices (tenant_id, status);
CREATE INDEX idx_slices_type ON network_slices (tenant_id, slice_type);
CREATE INDEX idx_slices_snssai ON network_slices (s_nssai);
CREATE INDEX idx_slices_resources ON network_slices USING GIN (resource_ids);
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
                            'customer', 'subscription', 'product', 'order',
                            'network_resource', 'service_instance', 'alarm',
                            'trouble_ticket', 'network_slice', 'invoice'
                        )),
    target_entity_id    UUID NOT NULL,
    summary             TEXT NOT NULL,
    detail_json         JSONB NOT NULL DEFAULT '{}',
    -- Example (churn_prediction): {
    --   "churn_probability": 0.73,
    --   "factors": ["3 service_degradation alarms in 30 days", "bill_increase_22pct", "contract_end_14_days"],
    --   "recommended_action": "offer_retention_bundle",
    --   "estimated_ltv_save_cents": 360000
    -- }
    explanation         TEXT,                   -- human-readable AI reasoning for regulatory audit
    model_id            TEXT,
    model_version       TEXT,
    reviewed_by         UUID REFERENCES users(id),
    reviewed_at         TIMESTAMPTZ,
    review_notes        TEXT,
    expires_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_tenant ON ai_suggestions (tenant_id);
CREATE INDEX idx_ai_type ON ai_suggestions (tenant_id, suggestion_type);
CREATE INDEX idx_ai_status ON ai_suggestions (tenant_id, status);
CREATE INDEX idx_ai_target ON ai_suggestions (target_entity_type, target_entity_id);
CREATE INDEX idx_ai_confidence ON ai_suggestions (tenant_id, confidence DESC);
CREATE INDEX idx_ai_created ON ai_suggestions (tenant_id, created_at DESC);

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
    -- Example: {"status": {"from": "active", "to": "suspended"}, "reason": "non_payment"}
    ip_address          TEXT,
    user_agent          TEXT,
    tmf_api             TEXT,                   -- TMF API that triggered the action (e.g. "TMF622")
    oda_component       TEXT,                   -- ODA component that originated the action
    gdpr_relevant       BOOLEAN NOT NULL DEFAULT false,
    data_residency      TEXT,                   -- region where this event is stored
    correlation_id      TEXT,                   -- distributed tracing
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_tenant ON audit_log (tenant_id);
CREATE INDEX idx_audit_actor ON audit_log (actor_type, actor_id);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_action ON audit_log (tenant_id, action);
CREATE INDEX idx_audit_created ON audit_log (created_at DESC);
CREATE INDEX idx_audit_tmf ON audit_log (tmf_api) WHERE tmf_api IS NOT NULL;
CREATE INDEX idx_audit_gdpr ON audit_log (tenant_id, entity_type) WHERE gdpr_relevant = true;
```

---

## Cross-Domain Query Examples

### Fault-to-billing impact (OSS → BSS)

```sql
-- Find all customers financially affected by an active critical alarm
SELECT c.id, c.company_name, c.email,
       s.msisdn, s.current_balance_cents,
       a.specific_problem, a.raised_at
FROM alarms a
JOIN service_instances si ON si.id = ANY(a.affected_services)
JOIN subscriptions sub ON sub.id = si.subscription_id
JOIN customers c ON c.id = sub.customer_id
JOIN subscriptions s ON s.customer_id = c.id
WHERE a.tenant_id = $1
  AND a.status = 'active'
  AND a.severity = 'critical'
ORDER BY a.raised_at DESC;
```

### Revenue at risk from network degradation

```sql
-- Sum monthly revenue from subscriptions on degraded network resources
SELECT nr.name AS resource_name, nr.resource_type,
       COUNT(DISTINCT sub.id) AS affected_subscriptions,
       SUM(p.pricing_json->'recurring'->>'amount_cents')::BIGINT AS monthly_revenue_at_risk_cents
FROM network_resources nr
JOIN service_instances si ON nr.id = ANY(si.resource_ids)
JOIN subscriptions sub ON sub.id = si.subscription_id
JOIN products p ON p.id = sub.product_id
WHERE nr.tenant_id = $1
  AND nr.health_status IN ('warning', 'critical')
  AND sub.status = 'active'
GROUP BY nr.id, nr.name, nr.resource_type
ORDER BY monthly_revenue_at_risk_cents DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 2 | Multi-tenant isolation, RBAC with ODA-aligned roles |
| Customer Management | 1 | B2C/B2B/wholesale with GDPR consent tracking |
| Product Catalogue | 1 | TMF620-aligned with convergent pricing and CAMARA API products |
| Order Management | 1 | TMF622 lifecycle with provisioning plan tracking |
| Subscriptions & Charging | 2 | Convergent account management + partitioned CDR/charging records (Nchf) |
| Billing | 2 | Invoices + 3GPP ABMF-aligned balance management |
| Network Inventory | 2 | Physical/logical resources (TMF638) + service instances (TMF641) |
| Fault Management | 2 | Alarm correlation + trouble tickets with SLA tracking |
| 5G Slicing | 1 | Slice lifecycle with SLA billing and ETSI ZSM reference |
| AI & Audit | 2 | Cross-domain AI suggestions + partitioned audit log |
| **Total** | **16** | |

---

## Key Design Decisions

1. **Row-level multi-tenancy** — every operational table carries `tenant_id` as a non-nullable FK, enabling MVNO and operator isolation without schema-per-tenant complexity. All queries include tenant_id in WHERE clauses and indexes.

2. **3GPP Nchf-native charging records** — the `charging_records` table models Nchf session lifecycle (initial/update/terminate) with rating groups and quota management, enabling direct integration with 5G core CHF without a translation layer.

3. **Convergent balance management** — a single `balances` table supports monetary (prepaid credit, postpaid ceiling), data, voice, SMS, and loyalty point balances, following the 3GPP ABMF pattern where a subscriber can hold multiple balance buckets simultaneously.

4. **TMF entity mapping** — `products` ↔ TMF620 ProductOffering, `orders` ↔ TMF622 ProductOrder, `network_resources` ↔ TMF638 Resource, `service_instances` ↔ TMF641 Service. Each table includes a `tmf_*_id` field for external API identity.

5. **Self-referencing network hierarchy** — `network_resources.parent_id` models site → rack → equipment → port containment and logical resource composition without a separate hierarchy table.

6. **Service-to-resource linking** — `service_instances.resource_ids` array with GIN index enables the critical OSS query "which resources deliver this service" and its inverse "which services are affected by this resource failure."

7. **Alarm correlation model** — `correlation_id` groups related alarms, `root_cause_alarm_id` identifies the root cause, and `affected_services` array enables instant subscriber impact assessment when a network fault occurs.

8. **5G slice-aware billing** — `network_slices` are first-class entities with their own SLA and billing model, linked to both `subscriptions` (commercial) and `network_resources` (technical), enabling per-slice SLA billing as a differentiated capability.

9. **Partitioned high-volume tables** — `charging_records` and `audit_log` are partitioned by timestamp range to manage the millions of CDRs and audit events generated daily by even a small operator.

10. **GDPR-aware customer model** — `customers` tracks consent granularity (marketing, analytics, third-party), data retention deadlines, and anonymisation timestamps, resolving the tension between GDPR minimisation and telecoms data retention directives.

11. **Cross-domain AI reasoning** — `ai_suggestions.target_entity_type` spans both BSS (customer, subscription, invoice) and OSS (network_resource, alarm, service_instance) domains, enabling the platform's core differentiator of unified BSS-OSS intelligence.

12. **ODA component provenance** — `audit_log.oda_component` tracks which ODA functional block (Party Management, Core Commerce, Production) originated each action, supporting TM Forum ODA certification requirements.

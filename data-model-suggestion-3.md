# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Telecoms BSS/OSS Platform · Created: 2026-05-26

## Philosophy

This model treats every state change across both BSS and OSS domains as an immutable event in a single append-only store. The event store is the sole source of truth — subscriber activations, charging sessions, product catalogue changes, network alarms, trouble ticket updates, order lifecycle transitions, and 5G slice provisioning are all recorded as typed events on domain streams. Materialised read models (CQRS projections) are rebuilt from these events to serve the operational queries that BSS and OSS users need: subscriber 360° view, network operations dashboard, billing summaries, and fault correlation boards.

This architecture is naturally suited to telecoms because the domain is inherently event-driven: CDRs are events, alarms are events, order state transitions are events, and billing cycles are event aggregations. Rather than forcing this event-native domain into a CRUD model and then bolting on audit logging, the event-sourced approach makes the event stream the primary data structure and derives CRUD-like views as projections. The result is a complete, tamper-evident audit trail that satisfies both GDPR data processing records and telecommunications regulatory requirements without any additional logging infrastructure.

The platform's core differentiator — cross-domain BSS-OSS causal reasoning — becomes a stream processing problem rather than a cross-database JOIN problem. An alarm event on a network stream can be correlated in real time with subscriber events on customer streams to produce an AI-driven churn prediction or proactive care action, all within the Kafka-native event backbone that the platform already specifies.

**Best for:** Operators who need complete audit trails, real-time event-driven processing, and the ability to replay history for regulatory compliance, dispute resolution, or AI model training.

**Trade-offs:**
- Pro: Complete, immutable audit trail satisfying GDPR processing records and telecoms regulatory requirements
- Pro: Natural fit for Kafka event backbone — events flow directly from Kafka topics to the event store
- Pro: Cross-domain BSS-OSS correlation is a stream processing problem, not a JOIN problem
- Pro: Temporal queries ("what was this subscriber's plan on March 15?") are trivial — replay events to that point
- Pro: AI model training can replay the full event history to learn from real operational sequences
- Con: Read models must be projected and maintained — adds operational complexity
- Con: Event schema evolution requires careful versioning (upcasters for old events)
- Con: Eventually consistent read models may lag behind the event store during high-volume bursts
- Con: Developers unfamiliar with CQRS/ES face a steeper learning curve than with CRUD

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| TMF620 (Product Catalogue) | `catalogue.*` events model product lifecycle; `rm_product_catalogue` read model serves TMF620 API |
| TMF622 (Product Ordering) | `order.*` events model order lifecycle; `rm_order_status` read model serves TMF622 API |
| TMF638 (Service Inventory) | `network.*` events model resource lifecycle; `rm_network_status` read model serves TMF638 API |
| TMF641 (Service Ordering) | `service.*` events model service provisioning; `rm_order_status` includes fulfilment state for TMF641 |
| TMF724 (Event Management) | TMF724 events map directly to event store entries — the API exposes the native event stream |
| 3GPP TS 32.290/32.291 | `charging.*` events capture Nchf session lifecycle (create/update/terminate) with rating groups |
| 3GPP CCS Architecture | Balance mutations are events; `rm_subscriber_360` read model shows current balance state per ABMF |
| ETSI NFV MANO | `network.vnf_instantiated`, `network.vnf_scaled`, `network.vnf_terminated` events track VNF lifecycle |
| ETSI ZSM | `slice.*` events model closed-loop slice automation per ZSM architecture |
| GSMA Open Gateway / CAMARA | API capability events tracked as `service.camara_capability_activated` |
| CloudEvents 1.0 | Event envelope follows CloudEvents spec (ce_source, ce_type, ce_specversion, ce_time) |
| AsyncAPI 3.0 | Event streams documented as AsyncAPI channels for Kafka topic contracts |
| ODA Component Model | `ce_source` identifies the ODA component that produced each event |
| GDPR | Complete processing record via event stream; `customer.anonymised` event triggers cascade |
| ISO 27001 | Immutable event store provides tamper-evident security audit trail |

---

## Event Store Infrastructure

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN (
                        'tenant', 'customer', 'catalogue', 'order',
                        'subscription', 'charging', 'billing',
                        'network', 'service', 'alarm', 'ticket',
                        'slice', 'ai', 'config'
                    )),
    stream_id       TEXT NOT NULL,              -- e.g. "customer:cust-uuid" or "network:gnb-london-01"
    sequence_num    BIGINT NOT NULL,
    event_type      TEXT NOT NULL,
    event_data      JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- CloudEvents envelope
    ce_source       TEXT NOT NULL,              -- ODA component or system origin
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',
    ce_type         TEXT NOT NULL,              -- CloudEvents type (e.g. "com.platform.customer.activated")
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Tenant isolation
    tenant_id       UUID NOT NULL,
    -- Actor
    actor_type      TEXT NOT NULL CHECK (actor_type IN (
                        'user', 'system', 'ai', 'api_client',
                        'kafka_consumer', 'scheduler', 'provisioning_engine',
                        'mediation', 'ocs', 'ofcs', 'nfv_orchestrator',
                        'nms', 'alarm_manager', 'care_agent_bot'
                    )),
    actor_id        TEXT NOT NULL,
    -- Tracing
    correlation_id  TEXT,                       -- distributed trace ID
    causation_id    UUID,                       -- event that caused this event
    -- TMF / regulatory
    tmf_api         TEXT,                       -- TMF API that triggered the event
    oda_component   TEXT,                       -- ODA functional block origin
    gdpr_relevant   BOOLEAN NOT NULL DEFAULT false,
    kafka_topic     TEXT,                       -- source Kafka topic
    kafka_offset    BIGINT,                     -- source Kafka offset for exactly-once tracking
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_num)
) PARTITION BY RANGE (ce_time);

CREATE INDEX idx_events_stream ON event_store (stream_id, sequence_num);
CREATE INDEX idx_events_tenant ON event_store (tenant_id, ce_time DESC);
CREATE INDEX idx_events_type ON event_store (event_type, ce_time DESC);
CREATE INDEX idx_events_correlation ON event_store (correlation_id);
CREATE INDEX idx_events_causation ON event_store (causation_id);
CREATE INDEX idx_events_actor ON event_store (actor_type, actor_id);
CREATE INDEX idx_events_tmf ON event_store (tmf_api) WHERE tmf_api IS NOT NULL;
CREATE INDEX idx_events_gdpr ON event_store (tenant_id, stream_type) WHERE gdpr_relevant = true;
CREATE INDEX idx_events_kafka ON event_store (kafka_topic, kafka_offset);

CREATE TABLE stream_snapshots (
    stream_id       TEXT NOT NULL,
    snapshot_at     BIGINT NOT NULL,            -- sequence_num at snapshot
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_at)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_sequence   BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Event Taxonomy

### Tenant & Configuration Events
```
tenant.created              — new operator/MVNO onboarded
tenant.configured           — billing, network, or regulatory config updated
tenant.suspended            — operator suspended (non-payment, regulatory)
tenant.terminated           — operator decommissioned
config.user_created         — operator staff account created
config.user_role_changed    — RBAC role assignment changed
config.user_deactivated     — staff account deactivated
config.idp_configured       — OAuth 2.0 / OIDC identity provider set up
```

### Product Catalogue Events
```
catalogue.product_created           — new product/offer defined (TMF620)
catalogue.product_updated           — product attributes modified
catalogue.product_activated         — product made available for sale
catalogue.product_retired           — product removed from sale
catalogue.pricing_changed           — pricing rules updated
catalogue.channel_distribution_set  — distribution channels configured
catalogue.ai_generated              — product created by LLM-assisted authoring
```

### Customer Lifecycle Events
```
customer.created            — subscriber record created (TMF629 Customer)
customer.identity_verified  — KYC/identity check completed
customer.credit_assessed    — credit class assigned
customer.consent_given      — GDPR consent recorded (with scope)
customer.consent_withdrawn  — GDPR consent revoked
customer.segment_updated    — customer segment changed (manual or AI)
customer.churn_risk_updated — AI churn prediction updated
customer.suspended          — account suspended (non-payment, fraud)
customer.terminated         — account closed
customer.anonymised         — GDPR erasure executed
customer.portability_exported — GDPR data export completed
```

### Order Lifecycle Events
```
order.created               — new order placed (TMF622)
order.validated             — order validation passed
order.provisioning_started  — network provisioning initiated
order.provisioning_step     — individual provisioning step completed/failed
order.activated             — service activated, subscription live
order.completed             — order fully fulfilled
order.failed                — order failed (with reason)
order.cancelled             — order cancelled (customer or system)
order.rolled_back           — failed order rolled back
```

### Subscription Events
```
subscription.activated      — subscription goes live
subscription.modified       — plan change, add-on added/removed
subscription.upgraded       — moved to higher tier
subscription.downgraded     — moved to lower tier
subscription.suspended      — temporarily suspended
subscription.resumed        — resumed from suspension
subscription.ported_in      — number ported in from another operator
subscription.ported_out     — number ported out to another operator
subscription.sim_swapped    — SIM card replaced
subscription.terminated     — subscription ended
subscription.renewed        — contract auto-renewed
```

### Charging & Billing Events
```
charging.session_created    — Nchf ConvergedCharging_Create
charging.session_updated    — Nchf ConvergedCharging_Update (quota grant/report)
charging.session_terminated — Nchf ConvergedCharging_Release
charging.cdr_rated          — CDR mediated and rated
charging.balance_topup      — prepaid balance topped up
charging.balance_reserved   — quota reserved for active session
charging.balance_deducted   — balance consumed
charging.adjustment         — manual billing adjustment
charging.refund             — refund issued
billing.cycle_started       — billing period opened
billing.invoice_generated   — invoice created
billing.invoice_sent        — invoice delivered to customer
billing.payment_received    — payment processed
billing.payment_failed      — payment attempt failed
billing.dunning_escalated   — dunning level increased
billing.dispute_raised      — customer disputes a charge
billing.dispute_resolved    — dispute resolved (credit or upheld)
billing.write_off           — balance written off
```

### Network Inventory Events
```
network.resource_created        — new resource registered (TMF638)
network.resource_commissioned   — resource brought into service
network.resource_updated        — resource attributes changed
network.resource_degraded       — resource health degraded
network.resource_maintenance    — resource taken for maintenance
network.resource_decommissioned — resource retired
network.vnf_instantiated        — VNF/CNF instantiated (ETSI NFV)
network.vnf_scaled              — VNF/CNF scaled up/down
network.vnf_terminated          — VNF/CNF terminated
network.capacity_threshold      — utilisation threshold crossed
network.firmware_updated        — equipment firmware updated
```

### Service Instance Events
```
service.provisioned         — service instance created and bound to resources (TMF641)
service.activated           — service instance active
service.performance_measured — SLA measurement recorded
service.sla_breached        — SLA threshold violated
service.modified            — service parameters changed
service.suspended           — service suspended
service.terminated          — service decommissioned
service.camara_capability_activated — CAMARA/Open Gateway API capability enabled
```

### Alarm & Fault Events
```
alarm.raised                — new alarm raised
alarm.acknowledged          — alarm acknowledged by operator
alarm.correlated            — alarm correlated with root cause
alarm.severity_changed      — alarm severity escalated/de-escalated
alarm.cleared               — alarm condition resolved
alarm.suppressed            — alarm suppressed (maintenance window)
alarm.subscriber_impact_assessed — affected subscriber count calculated
ticket.created              — trouble ticket opened (auto or manual)
ticket.assigned             — ticket assigned to team/individual
ticket.escalated            — ticket priority escalated
ticket.work_logged          — work note added
ticket.resolved             — fault resolved
ticket.closed               — ticket closed
ticket.reopened             — ticket reopened
```

### 5G Slice Events
```
slice.requested             — slice instantiation requested
slice.designed              — slice design approved
slice.instantiated          — slice network resources allocated and active
slice.sla_measured          — slice SLA performance recorded
slice.sla_breached          — slice SLA threshold violated
slice.scaled                — slice capacity scaled up/down
slice.billing_updated       — slice billing model or price changed
slice.subscriber_added      — subscriber attached to slice
slice.subscriber_removed    — subscriber detached from slice
slice.deactivated           — slice taken down
slice.terminated            — slice decommissioned
```

### AI Events
```
ai.churn_predicted          — churn risk score computed for customer
ai.offer_personalised       — personalised offer generated
ai.fraud_detected           — anomaly/fraud pattern detected in CDR stream
ai.root_cause_analysed      — cross-domain root cause analysis completed (OSS→BSS link)
ai.capacity_forecasted      — capacity demand forecast generated
ai.catalogue_drafted        — LLM generated product catalogue entry
ai.bill_adjustment_suggested — AI recommends billing adjustment
ai.network_optimisation     — network configuration recommendation
ai.slice_scaling_suggested  — slice scaling recommendation
ai.suggestion_accepted      — operator accepted AI suggestion
ai.suggestion_rejected      — operator rejected AI suggestion
ai.proactive_care_triggered — proactive customer care action initiated
```

---

## Read Models (CQRS Projections)

```sql
CREATE TABLE rm_subscriber_360 (
    customer_id         UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    customer_type       TEXT NOT NULL,
    status              TEXT NOT NULL,
    identity            JSONB NOT NULL DEFAULT '{}',
    -- Example: {"name": "Alice Smith", "email": "alice@example.com", "phone": "+447700900123", "credit_class": "standard"}
    subscriptions       JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"id": "sub-001", "product": "Essential 10GB", "msisdn": "+447700900123",
    --    "status": "active", "account_type": "postpaid",
    --    "allowances": {"data_mb": {"total": 10240, "used": 3072}, "voice_min": {"total": 300, "used": 45}},
    --    "activated_at": "2026-01-15", "contract_end": "2027-01-14"}
    -- ]
    balances            JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"type": "monetary", "currency": "GBP", "current_cents": 0, "credit_limit_cents": 50000}]
    billing_summary     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"current_period_cents": 2075, "outstanding_cents": 0, "last_payment_date": "2026-05-03", "dunning_level": 0}
    recent_orders       JSONB NOT NULL DEFAULT '[]',
    recent_tickets      JSONB NOT NULL DEFAULT '[]',
    service_quality     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"active_alarms_affecting": 0, "sla_status": "met", "avg_latency_ms": 12}
    engagement          JSONB NOT NULL DEFAULT '{}',
    -- Example: {"churn_risk": 0.15, "ltv_cents": 180000, "segment": "young_professional", "nps": 8}
    gdpr                JSONB NOT NULL DEFAULT '{}',
    -- Example: {"consent": {"marketing": true, "analytics": false}, "retention_until": "2028-01-14"}
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_sub360_tenant ON rm_subscriber_360 (tenant_id);
CREATE INDEX idx_rm_sub360_status ON rm_subscriber_360 (tenant_id, status);
CREATE INDEX idx_rm_sub360_engagement ON rm_subscriber_360 USING GIN (engagement);

CREATE TABLE rm_network_status (
    resource_id         UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    resource_type       TEXT NOT NULL,
    name                TEXT NOT NULL,
    status              TEXT NOT NULL,
    health_status       TEXT,
    equipment           JSONB NOT NULL DEFAULT '{}',
    -- Example: {"vendor": "Nokia", "model": "AirScale 5G BTS", "firmware": "24.3.1", "management_ip": "10.1.2.100"}
    capacity            JSONB NOT NULL DEFAULT '{}',
    -- Example: {"max_users": 1200, "current_users": 450, "utilisation_pct": 37.5}
    active_services     INTEGER NOT NULL DEFAULT 0,
    active_alarms       JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id": "alarm-uuid", "type": "high_cpu", "severity": "warning", "raised_at": "..."}]
    active_tickets      JSONB NOT NULL DEFAULT '[]',
    nfv_state           JSONB NOT NULL DEFAULT '{}',
    -- Example: {"descriptor": "vnfd-upf-001", "lcm_state": "instantiated", "replicas": {"desired": 3, "ready": 3}}
    slice_info          JSONB NOT NULL DEFAULT '{}',
    affected_subscribers INTEGER NOT NULL DEFAULT 0,
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_network_tenant ON rm_network_status (tenant_id);
CREATE INDEX idx_rm_network_type ON rm_network_status (tenant_id, resource_type);
CREATE INDEX idx_rm_network_health ON rm_network_status (tenant_id, health_status);
CREATE INDEX idx_rm_network_alarms ON rm_network_status USING GIN (active_alarms);

CREATE TABLE rm_billing_dashboard (
    tenant_id           UUID NOT NULL,
    period              DATE NOT NULL,          -- billing period start date
    summary             JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "total_subscribers": 12500,
    --   "active_subscriptions": 14200,
    --   "revenue": {
    --     "recurring_cents": 21375000, "usage_cents": 3420000,
    --     "one_off_cents": 150000, "total_cents": 24945000
    --   },
    --   "collections": {
    --     "paid_cents": 22000000, "outstanding_cents": 2945000,
    --     "overdue_cents": 500000, "write_off_cents": 45000
    --   },
    --   "by_product_type": {
    --     "mobile_bundle": {"subscribers": 10000, "revenue_cents": 18000000},
    --     "broadband": {"subscribers": 2000, "revenue_cents": 5000000},
    --     "iot_connectivity": {"subscribers": 500, "revenue_cents": 1945000}
    --   },
    --   "churn": {"terminated": 85, "ported_out": 23, "rate_pct": 0.86},
    --   "arpu_cents": 1996,
    --   "dunning": {"level_1": 150, "level_2": 45, "level_3": 12}
    -- }
    invoices_generated  INTEGER NOT NULL DEFAULT 0,
    invoices_sent       INTEGER NOT NULL DEFAULT 0,
    disputes_open       INTEGER NOT NULL DEFAULT 0,
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, period)
);

CREATE TABLE rm_fault_correlation (
    correlation_id      TEXT PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    status              TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                            'active', 'acknowledged', 'resolved'
                        )),
    root_cause          JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "resource_id": "uuid", "resource_name": "gNodeB London-East-01",
    --   "alarm_type": "equipment_failure", "severity": "critical",
    --   "probable_cause": "power_supply_failure",
    --   "ai_analysis": "PSU A failed, causing sector 1 and 2 radio shutdown. 450 subscribers affected."
    -- }
    correlated_alarms   JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"alarm_id": "uuid", "resource": "gNodeB London-East-01", "type": "equipment_failure", "severity": "critical"},
    --   {"alarm_id": "uuid", "resource": "Cell London-E-03", "type": "link_down", "severity": "major"},
    --   {"alarm_id": "uuid", "resource": "Cell London-E-04", "type": "link_down", "severity": "major"}
    -- ]
    affected_resources  INTEGER NOT NULL DEFAULT 0,
    affected_subscribers INTEGER NOT NULL DEFAULT 0,
    affected_services   INTEGER NOT NULL DEFAULT 0,
    business_impact     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"revenue_at_risk_cents": 675000, "sla_breaches": 3, "churn_risk_customers": 12}
    trouble_ticket_id   TEXT,
    raised_at           TIMESTAMPTZ NOT NULL,
    resolved_at         TIMESTAMPTZ,
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_fault_tenant ON rm_fault_correlation (tenant_id);
CREATE INDEX idx_rm_fault_status ON rm_fault_correlation (tenant_id, status);

CREATE TABLE rm_slice_dashboard (
    slice_id            TEXT PRIMARY KEY,       -- matches stream_id for slice streams
    tenant_id           UUID NOT NULL,
    s_nssai             TEXT NOT NULL,
    slice_type          TEXT NOT NULL,
    status              TEXT NOT NULL,
    sla                 JSONB NOT NULL DEFAULT '{}',
    -- Example: {"throughput_mbps": 1000, "latency_ms": 1, "reliability_pct": 99.999}
    sla_status          TEXT,
    performance         JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "current_throughput_mbps": 950, "current_latency_ms": 0.8,
    --   "current_reliability_pct": 99.9995,
    --   "sla_breaches_24h": 0, "sla_breaches_30d": 1
    -- }
    subscribers         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"max": 5000, "current": 1200, "added_24h": 5, "removed_24h": 1}
    resources           JSONB NOT NULL DEFAULT '[]',
    billing             JSONB NOT NULL DEFAULT '{}',
    -- Example: {"model": "sla_tiered", "current_tier": "premium", "period_revenue_cents": 500000}
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_slice_tenant ON rm_slice_dashboard (tenant_id);
CREATE INDEX idx_rm_slice_type ON rm_slice_dashboard (tenant_id, slice_type);
CREATE INDEX idx_rm_slice_sla ON rm_slice_dashboard (tenant_id, sla_status);
```

---

## Event-Driven Cross-Domain Correlation Examples

### Alarm → subscriber impact (stream processing)

```sql
-- When an alarm.raised event arrives, find all affected subscribers
-- by replaying service.provisioned events for the affected resource
SELECT e.event_data->>'customer_id' AS customer_id,
       e.event_data->>'subscription_id' AS subscription_id,
       e.event_data->'params'->>'msisdn' AS msisdn
FROM event_store e
WHERE e.stream_type = 'service'
  AND e.event_type = 'service.provisioned'
  AND e.event_data->>'resource_id' = $1  -- the alarmed resource
  AND e.tenant_id = $2
  AND NOT EXISTS (
      SELECT 1 FROM event_store e2
      WHERE e2.stream_id = e.stream_id
        AND e2.event_type = 'service.terminated'
        AND e2.sequence_num > e.sequence_num
  );
```

### Temporal query: subscriber plan on a specific date

```sql
-- Replay subscription events up to a target date to determine plan state
SELECT event_type, event_data, ce_time
FROM event_store
WHERE stream_id = 'subscription:sub-001'
  AND ce_time <= '2026-03-15T23:59:59Z'
ORDER BY sequence_num ASC;
-- Application replays these events to reconstruct the subscription state at that point
```

### Billing dispute resolution: full charging history

```sql
-- All charging events for a subscription in a billing period
SELECT ce_time, event_type,
       event_data->>'rated_amount_cents' AS amount,
       event_data->'usage'->>'bytes' AS data_bytes,
       event_data->'nchf'->>'rating_group' AS rating_group
FROM event_store
WHERE stream_id = 'charging:sub-001'
  AND ce_time BETWEEN '2026-05-01' AND '2026-05-31'
ORDER BY sequence_num ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | Partitioned event store + snapshots + projection checkpoints |
| Read Models | 5 | Subscriber 360°, network status, billing dashboard, fault correlation, slice dashboard |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Single unified event store across BSS and OSS** — rather than separate event stores per domain, all events (customer, charging, network, alarm, slice) share one partitioned table. This makes cross-domain correlation a single-table query and ensures consistent ordering via `ce_time`.

2. **CloudEvents envelope on every event** — `ce_source`, `ce_type`, `ce_specversion`, and `ce_time` follow the CloudEvents 1.0 specification, making events directly publishable to external systems and compatible with TMF724 Event Management API.

3. **Kafka-native with exactly-once tracking** — `kafka_topic` and `kafka_offset` on each event enable idempotent ingestion from the Kafka event backbone. Events can be replayed from Kafka if the event store needs rebuilding.

4. **14 stream types spanning full BSS-OSS domain** — tenant, customer, catalogue, order, subscription, charging, billing, network, service, alarm, ticket, slice, ai, config. Each stream type has its own event taxonomy, but all share the same table and indexing strategy.

5. **Causation chain for regulatory compliance** — `causation_id` links events that triggered other events (e.g., an `alarm.raised` event causing a `ticket.created` event causing an `ai.root_cause_analysed` event). This chain provides auditable provenance for any system action.

6. **GDPR as an event** — `customer.anonymised` is an event that triggers downstream projections to scrub PII from read models. The event store retains the anonymisation event itself (recording that erasure occurred) while the original PII events can be cryptographically erased using per-customer encryption keys.

7. **Five read models for five user personas** — `rm_subscriber_360` (care agents), `rm_network_status` (NOC operators), `rm_billing_dashboard` (finance), `rm_fault_correlation` (network engineers), `rm_slice_dashboard` (5G operations). Each is optimised for its primary use case.

8. **Fault correlation as a projected aggregate** — rather than correlating alarms at write time, the `rm_fault_correlation` read model aggregates related alarm events by `correlation_id` and enriches them with subscriber impact and revenue-at-risk calculations. This means correlation logic can be updated without migrating stored data.

9. **Nchf charging session as event stream** — the 3GPP Nchf lifecycle (Create → Update* → Release) maps naturally to an event sequence on a charging stream. Quota management becomes event replay: the current quota state is the sum of all quota grant and consumption events.

10. **ODA component provenance** — `ce_source` and `oda_component` on every event identify which ODA functional block produced the event, supporting TM Forum ODA certification and enabling per-component event analytics.

11. **Stream snapshots for performance** — high-event-count streams (charging streams for heavy-usage subscribers, alarm streams for noisy network elements) use periodic snapshots to avoid replaying thousands of events for current-state queries. The snapshot interval is configurable per stream type.

12. **Billing dashboard as a time-series projection** — `rm_billing_dashboard` is keyed by `(tenant_id, period)`, producing one row per billing period per tenant. This gives finance teams instant access to revenue, collections, churn, and ARPU metrics without aggregating millions of charging events at query time.

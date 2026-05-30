# Telecoms BSS/OSS Platform — Phased Development Plan

> Project: 375-telecoms-bss-oss-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three `data-model-suggestion-*.md` files. The database design adopts **Data Model Suggestion 1 (Entity-Centric Normalized Relational)** as the canonical schema, because the platform's core value proposition — auditable, standards-compliant, cross-domain BSS↔OSS reasoning — depends on referential integrity and direct mapping to TM Forum Open API entities. JSONB columns from that model are retained for flexible sub-structures (pricing, provisioning specs, SLA terms), giving us the hybrid benefits of Suggestion 2 within a normalized spine. The event-driven backbone from Suggestion 3 is adopted *as transport* (Kafka) rather than as the system of record.

---

## Core Requirements (Synthesis)

**What it does.** An open, AI-native, standards-aligned BSS/OSS platform that unifies subscriber management, convergent charging, order management, network inventory, service fulfilment, and fault management in one modular multi-tenant stack — targeted at MVNOs, regional carriers, and digital-first operators priced out of Tier-1 vendor offerings.

**Who uses it.** Operator engineering teams (API integration), catalogue/pricing managers (product config), billing/finance teams (invoicing, revenue assurance), NOC operators and network engineers (inventory, fault, performance), care agents (360° customer view), and enterprise B2B/consumer subscribers (self-service portals). Multi-tenant: one deployment serves many operators.

**Key differentiators.** (1) Affordable full-stack BSS *and* OSS in one open codebase; (2) a unified cross-domain intelligence layer that links network faults → service impact → subscription → billing → churn, which incumbents bolt on as separate silos; (3) standards-first (TM Forum Open APIs, 3GPP Nchf, ETSI MANO/ZSM, CAMARA) to escape data-model lock-in; (4) explainable AI surfaces for regulator-auditable decisions.

**MVP feature set** (from features.md "Must-have"): TMF620 product catalogue; convergent charging (prepaid/postpaid/hybrid, 3GPP Nchf-aligned); order management + automated provisioning (TMF622, TMF641); subscriber/CRM 360° view; network inventory physical+logical with service-to-resource linking (TMF638); fault management (alarm ingest, correlation, trouble tickets); RBAC, multi-tenancy, audit logging.

**Post-MVP** (v1.1 / backlog): Kafka event-driven backbone; AI anomaly detection on CDR/event streams; subscriber + B2B self-service portals; 5G slice lifecycle with per-slice SLA billing; NL catalogue configuration (LLM); ODA certification path; digital twin; A2A multi-agent orchestration; cross-domain root-cause → churn analysis; carbon reporting.

**Deployment model.** Cloud-native, container-first, Kubernetes-targeted; multi-tenant SaaS with self-hosted option. REST (OAS 3.1) + GraphQL northbound APIs, AsyncAPI-documented Kafka event streams, developer portal.

**Integration surface.** 3GPP 5G core (Nchf to CHF/SMF), mediation/CDR ingestion, ETSI NFV-MANO (SOL005 to NFVO), NMS/EMS southbound (alarms, performance), payment gateways, OAuth2/OIDC identity providers, LLM providers (Vercel AI Gateway / Anthropic / Bedrock), GSMA Open Gateway/CAMARA.

**Standards compliance.** TMF620/621/622/638/641/724, 3GPP TS 32.290/32.291 (Nchf), ETSI NFV-SOL005 + ZSM, CAMARA, OAuth2 (RFC 6749)/OIDC, OWASP API Security Top 10, NIST SP 800-207 (zero trust), GDPR + telecoms retention, OpenAPI 3.1, AsyncAPI 3.0, JSON Schema 2020-12, CloudEvents 1.0, ISO 27001.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | Domain is heavy on data modelling, AI/LLM integration, and rapid standards-conformant API surface. Python's ecosystem (Pydantic, FastAPI, SQLAlchemy, the Anthropic/OpenAI SDKs, scikit-learn/river for streaming anomaly detection) covers every layer. Type hints + Pydantic give us JSON-Schema-aligned validation matching TMF/CAMARA models. |
| Performance-critical charging path | **Python with optional Rust extension (PyO3) deferred** | The convergent charging rating loop is the only true hot path. We build it in pure Python first (Phase 4) behind a `RatingEngine` interface; if benchmarks demand it, a Rust/PyO3 rating kernel can be swapped in without touching callers. Decision made, not deferred architecturally. |
| API framework | **FastAPI** | Native OpenAPI 3.1 generation (required for TMF/CAMARA SDK generation), Pydantic v2 models, async I/O for Kafka/HTTP fan-out, dependency-injection for tenant/auth scoping. |
| GraphQL | **Strawberry GraphQL** (mounted on FastAPI) | Code-first, Pydantic-friendly GraphQL for the 360° aggregate queries where REST would require many round-trips. Added in Phase 9, not MVP. |
| Database | **PostgreSQL 16** | Data Model Suggestion 1 requires FK integrity, partitioning (`charging_records`, `audit_log`), JSONB+GIN, row-level multi-tenancy, and `gen_random_uuid()`. Postgres delivers all of these. SQLite is insufficient (no partitioning, weak JSONB). |
| Row-level isolation | **Postgres RLS policies keyed on `tenant_id`** | Defence-in-depth for multi-tenancy; the app sets `SET app.tenant_id` per request; RLS enforces it even if an app query forgets the WHERE clause (NIST 800-207 zero-trust). |
| ORM / query layer | **SQLAlchemy 2.0 (async) + Alembic** | Mature async ORM with explicit Core for hot paths; Alembic for the coordinated migrations the normalized model needs. |
| Schema migrations | **Alembic** | Versioned, reviewable migrations; partition management via migration scripts. |
| Task queue / async workers | **Celery + Redis** (MVP), **Kafka consumers** (v1.1) | Provisioning workflows, invoice generation, dunning, and AI scoring are long-running/async. Celery+Redis is the zero-Kafka MVP path; the Kafka event backbone (Phase 8) becomes the production transport for sub-second billing/provisioning events. |
| Event backbone | **Apache Kafka** (Redpanda for local dev) | Required by the product thesis (real-time mediation/rating/provisioning). Events documented with AsyncAPI 3.0 + CloudEvents 1.0 envelope. |
| Cache / quota reservation | **Redis** | Online-charging quota reservation (`balances.reserved_value`) and session state need low-latency atomic ops; also Celery broker. |
| Frontend (operator console) | **Next.js 16 (App Router) + React + shadcn/ui + Tailwind** | Operator console (catalogue, orders, NOC dashboards, 360° view) and self-service portals. Server Components for data-heavy dashboards; deployed on the same K8s cluster or Vercel. Added Phase 9. |
| AI / LLM access | **Vercel AI Gateway → Anthropic Claude (primary), with provider failover** | NL catalogue config, root-cause narratives, churn explanations, anomaly summarisation. Gateway gives provider failover + cost tracking; explainability text stored in `ai_suggestions.explanation` for regulatory audit. |
| Streaming anomaly detection | **river (online ML) + scikit-learn (batch)** | CDR/alarm anomaly detection over high-volume streams; `river` does incremental learning without retraining; scikit-learn for batch churn models. |
| Auth | **OAuth 2.0 / OIDC (Authlib), JWT bearer** | TMF APIs mandate OAuth2; partner/developer + subscriber portal auth. mTLS for network-function ↔ charging interfaces per 3GPP security architecture. |
| API spec artifacts | **OpenAPI 3.1 (FastAPI auto), AsyncAPI 3.0 (hand-authored + CI-validated)** | SDK generation + TMF/CAMARA conformance tooling. |
| Containerisation | **Docker + docker-compose (dev), Helm chart (prod K8s)** | Cloud-native deployment thesis; ODA components map to deployable services. |
| Testing | **pytest, pytest-asyncio, testcontainers (Postgres/Kafka/Redis), schemathesis (API fuzzing), Playwright (UI)** | testcontainers gives real-dependency integration tests; schemathesis fuzzes the OpenAPI surface against OWASP API risks. |
| Code quality | **ruff (lint+format), mypy (strict), bandit (security), pip-audit** | Single fast linter/formatter; strict typing for a large domain model; bandit + pip-audit for OWASP/supply-chain. |
| Package / deps | **uv + pyproject.toml** | Fast, reproducible dependency resolution and lockfile. |
| Observability | **OpenTelemetry → OTLP; structured JSON logs** | `correlation_id` propagation across services (audit_log + distributed tracing) is required for cross-domain RCA. |
| Key libraries | Pydantic v2, SQLAlchemy 2, Alembic, FastAPI, Authlib, confluent-kafka, redis-py, celery, river, scikit-learn, anthropic, strawberry-graphql, reportlab (invoice PDF), phonenumbers (E.164), pycountry (ISO 3166), structlog | Domain-specific support for E.164/MSISDN handling, ISO country codes, invoice rendering, charging, and AI. |

### Project Structure

```
telecoms-bss-oss/
├── pyproject.toml
├── uv.lock
├── README.md
├── Dockerfile
├── docker-compose.yml                 # postgres, redis, redpanda, api, worker, console
├── helm/                              # production K8s chart (ODA-component-aligned services)
│   └── bssoss/
├── alembic.ini
├── migrations/                        # Alembic versions (incl. partition setup)
│   └── versions/
├── openapi/                          # exported OAS 3.1 specs per ODA component
├── asyncapi/                         # AsyncAPI 3.0 channel definitions (Kafka)
├── src/
│   └── bssoss/
│       ├── __init__.py
│       ├── main.py                    # FastAPI app factory, router mounting
│       ├── config.py                  # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── base.py                # async engine, session, RLS context
│       │   ├── models/                # SQLAlchemy ORM models (one file per domain)
│       │   │   ├── tenant.py          # tenants, users
│       │   │   ├── customer.py        # customers
│       │   │   ├── catalogue.py       # products
│       │   │   ├── order.py           # orders
│       │   │   ├── subscription.py    # subscriptions, balances
│       │   │   ├── charging.py        # charging_records (partitioned)
│       │   │   ├── billing.py         # invoices
│       │   │   ├── inventory.py       # network_resources, service_instances
│       │   │   ├── fault.py           # alarms, trouble_tickets
│       │   │   ├── slicing.py         # network_slices
│       │   │   └── ai_audit.py        # ai_suggestions, audit_log (partitioned)
│       │   └── repositories/          # tenant-scoped data access
│       ├── schemas/                   # Pydantic request/response (TMF-aligned)
│       │   ├── tmf620.py … tmf724.py
│       │   └── internal.py
│       ├── api/                       # FastAPI routers (REST), one per ODA component
│       │   ├── deps.py                # auth, tenant, db-session dependencies
│       │   ├── catalogue.py           # TMF620
│       │   ├── ordering.py            # TMF622 + TMF641
│       │   ├── customers.py
│       │   ├── subscriptions.py
│       │   ├── charging.py            # Nchf-facing
│       │   ├── billing.py
│       │   ├── inventory.py           # TMF638
│       │   ├── fault.py
│       │   ├── slicing.py
│       │   ├── events.py              # TMF724
│       │   └── ai.py
│       ├── graphql/                   # Strawberry schema (Phase 9)
│       ├── domain/                    # business logic (framework-agnostic)
│       │   ├── catalogue/
│       │   ├── ordering/              # order orchestration state machine
│       │   ├── provisioning/          # fulfilment plan executor + adapters
│       │   ├── charging/              # RatingEngine, quota, balances
│       │   ├── billing/               # invoice generation, dunning, tax
│       │   ├── fault/                 # alarm correlation, RCA
│       │   ├── slicing/
│       │   └── ai/                    # anomaly, churn, RCA narrative, NL catalogue
│       ├── integrations/
│       │   ├── kafka/                 # producer/consumer, CloudEvents envelope
│       │   ├── nfv/                   # ETSI SOL005 NFVO adapter (+ mock)
│       │   ├── nms/                   # alarm/perf southbound adapters (+ mock)
│       │   ├── payments/              # gateway adapter (+ mock)
│       │   └── llm/                   # AI Gateway/Anthropic client
│       ├── auth/                      # OAuth2/OIDC, JWT, RBAC, RLS context
│       ├── audit/                     # audit-log writer, GDPR tooling
│       ├── workers/                   # Celery tasks (provisioning, billing, AI)
│       └── observability/             # OTel setup, structured logging
│       └── console/                   # (Next.js app lives in /console at repo root)
├── console/                          # Next.js 16 operator console + portals (Phase 9)
└── tests/
    ├── unit/
    ├── integration/                  # testcontainers
    ├── e2e/
    └── fixtures/                     # sample CDRs, alarms, TMF payloads, catalogues
```

The structure groups by **concern** (api / domain / db / integrations) and maps service boundaries to **ODA functional blocks** (Party Management → tenant+customer; Core Commerce → catalogue+ordering+charging+billing; Production → inventory+provisioning+fault+slicing; Intelligence & Insights → ai; Common → auth+audit+events). Each phase adds files without restructuring.

---

## Phase 1: Foundation — Project Skeleton, Config, Multi-Tenancy & Auth

### Purpose
Establish the runnable scaffold every later phase depends on: dependency management, FastAPI app factory, configuration, the Postgres connection with row-level-security tenant context, Alembic migrations for the `tenants`/`users` tables, OAuth2/JWT authentication, RBAC, and the audit-log writer. After this phase the platform boots, authenticates a request, scopes it to a tenant, and records an audit entry — the spine for all CRUD that follows.

### Tasks

#### 1.1 — Project scaffold & tooling

**What**: Create the repo skeleton, `pyproject.toml`, dev tooling, Docker Compose, and a health endpoint.

**Design**:
- `pyproject.toml` declares dependencies and tool config (ruff, mypy strict, pytest).
- `docker-compose.yml` services: `postgres:16`, `redis:7`, `redpanda` (Kafka API), `api`, `worker`.
- `config.py` using `pydantic_settings.BaseSettings`:
```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    kafka_bootstrap: str = "localhost:9092"
    jwt_issuer: str
    jwt_audience: str = "bssoss-api"
    oidc_jwks_url: str | None = None
    llm_gateway_url: str | None = None
    environment: Literal["dev", "staging", "prod"] = "dev"
    model_config = SettingsConfigDict(env_prefix="BSSOSS_", env_file=".env")
```
- `main.py` exposes `create_app() -> FastAPI` mounting routers and a `GET /healthz` returning `{"status":"ok","version":...}`.

**Testing**:
- `Unit: create_app() returns FastAPI instance with /healthz registered`
- `Integration: GET /healthz → 200, body {"status":"ok"}`
- `Unit: Settings loads from env with BSSOSS_ prefix; missing database_url → ValidationError naming the field`
- `CI: ruff, mypy --strict, bandit all pass on the skeleton`

#### 1.2 — Database core, RLS tenant context, base model

**What**: Async SQLAlchemy engine/session, a tenant-context mechanism that sets `app.tenant_id` for RLS, and a declarative base with shared columns.

**Design**:
- `db/base.py`: `async_engine`, `async_session_factory`, and an async context manager `tenant_session(tenant_id: UUID)` that runs `SET LOCAL app.tenant_id = :tid` then yields a session.
- `TimestampMixin` (created_at/updated_at) and `TenantMixin` (`tenant_id: Mapped[UUID]`).
- RLS policy template applied per operational table in migrations:
```sql
ALTER TABLE <t> ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON <t>
  USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

**Testing**:
- `Integration (testcontainers PG): two tenants insert rows; querying under tenant A's context returns only A's rows`
- `Integration: query without app.tenant_id set → 0 rows / RLS denies (fails closed)`
- `Unit: tenant_session sets and resets SET LOCAL within transaction scope`

#### 1.3 — `tenants` and `users` tables + migration

**What**: Implement the `tenants` and `users` schema from Data Model Suggestion 1 with Alembic migration #0001.

**Design**: ORM models mirroring the DDL in `data-model-suggestion-1.md` (tenant_type/status enums via CHECK; `regulatory_ids`, `billing_config`, `kafka_topics` as JSONB; `users.role` 14-value enum; `UNIQUE(tenant_id,email)`). Indexes as specified.

**Testing**:
- `Integration: insert tenant with tenant_type='mvno' → succeeds; tenant_type='invalid' → IntegrityError (CHECK)`
- `Integration: two users same email different tenants → ok; same tenant+email → UNIQUE violation`
- `Migration: alembic upgrade head then downgrade base runs clean on empty PG`

#### 1.4 — OAuth2/OIDC authentication & JWT validation

**What**: Bearer-token auth validating JWTs against an OIDC JWKS, resolving the calling user + tenant.

**Design**:
- `auth/jwt.py`: fetch+cache JWKS, verify signature/`iss`/`aud`/`exp`; extract `sub`, `tenant_id`, `roles`.
- FastAPI dependency `get_current_principal() -> Principal` where
```python
@dataclass
class Principal:
    user_id: UUID
    tenant_id: UUID
    roles: list[str]
    scopes: list[str]
```
- `get_tenant_session` dependency chains principal → `tenant_session(principal.tenant_id)`.
- mTLS path stubbed for NF↔CHF (real in Phase 4).

**Testing**:
- `Unit: valid signed JWT → Principal with correct tenant_id/roles`
- `Unit: expired token → 401; wrong audience → 401; bad signature → 401`
- `Integration: protected endpoint without Authorization header → 401`

#### 1.5 — RBAC enforcement

**What**: Role/scope-based authorization decorator/dependency keyed to the 14 `users.role` values.

**Design**: `auth/rbac.py` `require_roles(*roles)` and `require_scope(scope)` dependencies; a static role→permission matrix (e.g. `catalogue_manager` → catalogue write; `noc_operator` → fault read/ack; `read_only` → read all). Returns 403 on failure with `{"code":"FORBIDDEN","required":[...]}`.

**Testing**:
- `Unit: principal with role 'billing_manager' calling require_roles('catalogue_manager') → 403`
- `Unit: 'admin' passes any require_roles`
- `Integration: catalogue_manager POST /catalogue → allowed; care_agent POST /catalogue → 403`

#### 1.6 — Audit-log writer & `audit_log` table (partitioned)

**What**: Implement `audit_log` (range-partitioned by `created_at`) and a writer invoked on every mutating action.

**Design**: ORM + migration creating the parent partitioned table and a monthly-partition creation helper. `audit.record(actor, action, entity_type, entity_id, changes, *, tmf_api=None, oda_component=None, gdpr_relevant=False, correlation_id=...)`. Writer derives `data_residency` from the tenant. Mutating API deps auto-emit a baseline entry.

**Testing**:
- `Integration: create-customer flow writes audit_log row with action='create', entity_type='customer', actor_type='user'`
- `Unit: changes diff {status: active→suspended} captured correctly`
- `Integration: audit row lands in correct monthly partition`

---

## Phase 2: Customer Management (CRM 360° Foundation)

### Purpose
Implement the subscriber/customer domain — the anchor entity for BSS. Everything (orders, subscriptions, billing, AI) references a customer. This phase delivers B2C/B2B/wholesale customer CRUD with GDPR consent tracking and the data shape that powers the 360° view later.

### Tasks

#### 2.1 — `customers` table, model & repository

**What**: Implement the `customers` schema (Suggestion 1) with tenant-scoped repository.

**Design**: ORM model with individual (first/last/dob) and enterprise (company/registration/tax) fields, `customer_type`/`status`/`account_type`/`credit_class` CHECK enums, `billing_address`/`gdpr_consent`/`metadata` JSONB, churn/LTV fields, `data_retention_until`, `anonymised_at`. Repository methods: `create`, `get`, `list(filter, page)`, `update`, `set_status`. E.164 validation on `phone` via `phonenumbers`.

**Testing**:
- `Unit: create individual customer with valid phone → stored normalized E.164`
- `Unit: enterprise customer missing company_name → app-level ValidationError`
- `Integration: list customers respects tenant isolation + segment/tag GIN filters`

#### 2.2 — Customer REST API

**What**: CRUD endpoints `POST/GET/PATCH /customers`, `GET /customers/{id}`, `GET /customers?type=&status=&segment=`.

**Design**: Pydantic request/response models; pagination `{items, total, page, page_size}`; PATCH performs partial update + audit diff; status transitions validated (`prospect→active→suspended/terminated→churned→anonymised`). RBAC: `care_agent`/`order_manager` write, `read_only` read.

**Testing**:
- `Integration: POST valid customer → 201 with id; audit entry written`
- `Integration: GET list paginates correctly; filters apply`
- `Integration: PATCH illegal status (terminated→active) → 409`
- `API fuzz (schemathesis): customers endpoints pass OAS conformance`

#### 2.3 — GDPR tooling (consent, retention, erasure/anonymisation)

**What**: Consent management, retention-deadline computation, and right-to-erasure via anonymisation.

**Design**:
- `audit/gdpr.py`: `record_consent(customer_id, {marketing,analytics,third_party})`; `compute_retention(customer, jurisdiction) -> datetime` (resolves GDPR-minimisation vs telecoms-retention tension using tenant `data_residency` + retention rules table); `anonymise(customer_id)` nulls PII (name, dob, address, phone, email), sets `status='anonymised'`, `anonymised_at`, and writes a `gdpr_relevant` audit entry — but **preserves** charging_records/invoices needed for retention by replacing direct PII with a tombstone reference.
- Endpoint `POST /customers/{id}/erase` (role `compliance_officer`).

**Testing**:
- `Unit: anonymise() removes all PII fields, retains financial records via tombstone`
- `Unit: compute_retention for UK CDR metadata returns >= statutory minimum`
- `Integration: erase writes gdpr_relevant audit row; subsequent GET shows anonymised status`

---

## Phase 3: Product Catalogue (TMF620)

### Purpose
Deliver the product catalogue — the engine that drives consistent offers across channels and feeds every order, subscription, and charging decision. TMF620 compliance here is the first standards-conformance milestone and a procurement-scoring differentiator.

### Tasks

#### 3.1 — `products` table, model & versioning

**What**: Implement `products` (Suggestion 1) with version control and lifecycle states.

**Design**: ORM with `product_type` (16-value enum), `category`, `status` (`draft→test→active→sunset→retired`), `version` integer, `valid_from/valid_to`, `channels[]`, and JSONB `pricing_json`/`allowances_json`/`charging_config`/`provisioning_spec`/`eligibility_rules`, `dependencies[]`, `camara_apis[]`. Editing an `active` product creates a new `version` row (immutable history) rather than mutating in place.

**Testing**:
- `Unit: editing active product increments version, prior version retained`
- `Integration: status transition draft→active allowed; retired→active rejected`
- `Unit: pricing_json validated against PricingSchema (recurring/one_off/usage/overage)`

#### 3.2 — TMF620 REST API

**What**: TMF620-conformant `productCatalogManagement` endpoints: `GET/POST /catalog/productOffering`, `GET /catalog/productOffering/{id}`, `PATCH`, plus `productSpecification` projection.

**Design**: Pydantic models matching TMF620 `ProductOffering` (id, name, lifecycleStatus, validFor, productSpecification, productOfferingPrice). Internal `products` rows mapped to/from TMF620 representation in `schemas/tmf620.py`. `tmf_product_id` stored for external identity. RBAC: `catalogue_manager` write.

**Testing**:
- `Integration: POST TMF620 ProductOffering payload → persisted; GET returns TMF620-shaped JSON`
- `Fixture: TM Forum sample TMF620 payload round-trips without data loss`
- `Conformance: response validates against TMF620 OpenAPI schema`

#### 3.3 — Eligibility & dependency resolution

**What**: A pure-function service that, given a customer + product, returns eligibility and resolves prerequisite products.

**Design**: `domain/catalogue/eligibility.py`: `check_eligibility(customer, product) -> EligibilityResult(eligible: bool, reasons: list[str])` applying `eligibility_rules` (min_credit_class, required_identity, excluded_segments); `resolve_dependencies(product) -> list[Product]` topologically ordering `dependencies[]` (detect cycles).

**Testing**:
- `Unit: prepaid_only customer for product requiring 'standard' credit → ineligible with reason`
- `Unit: dependency chain A→B→C returns [C,B,A]; cyclic deps → error`

---

## Phase 4: Convergent Charging Engine (3GPP Nchf)

### Purpose
The commercial heart of the platform: a convergent rating/charging engine handling prepaid (online), postpaid (offline), and hybrid on one platform, modelling 3GPP Nchf session lifecycle and ABMF balance management. This is where the "ship the core in phases 3–5" principle applies — it must be production-grade.

### Tasks

#### 4.1 — `subscriptions` & `balances` tables

**What**: Implement `subscriptions` (Suggestion 1) and `balances` (3GPP ABMF).

**Design**: ORM per DDL — `subscriptions` with MSISDN/IMSI/ICCID/IMEI, `account_type`, `allowances_json`, `current_balance_cents`, `rating_group`, `network_slice_id`, `service_instance_ids[]`; `balances` supporting monetary/data/voice/sms/loyalty buckets with `reserved_value`, `credit_limit`, `threshold_alert_pct`, `valid_to`. Redis mirror of monetary/data balances for low-latency reservation.

**Testing**:
- `Unit: create subscription validates E.164 MSISDN, IMSI format`
- `Integration: customer can hold multiple balance buckets (monetary+data) simultaneously`

#### 4.2 — RatingEngine

**What**: A pluggable rating engine that prices a usage event against the subscription's product pricing.

**Design**:
```python
@dataclass
class UsageEvent:
    subscription_id: UUID
    record_type: str          # data_session, voice_mo, sms_mo, …
    quantity: int             # bytes/seconds/messages
    unit: str
    timestamp: datetime
    roaming: bool = False
    location_mcc_mnc: str | None = None

@dataclass
class RatingResult:
    rated_amount_cents: int
    tax_amount_cents: int
    rating_group: int
    applied_allowance: str | None   # which bucket absorbed usage
    overage_quantity: int

class RatingEngine(Protocol):
    def rate(self, event: UsageEvent, product: Product, balances: list[Balance]) -> RatingResult: ...
```
Algorithm: resolve rating_group → consume from matching allowance bucket first → price overage at `pricing_json.overage` → apply roaming multiplier if `roaming` and zone matches → compute tax via TaxResolver (Phase 6). Pure function; no I/O — testable in isolation. Interface allows a future Rust/PyO3 kernel.

**Testing**:
- `Unit: 1GB data session, 20GB allowance with 5GB used → 0 cents, allowance bucket decremented`
- `Unit: usage exceeding allowance → overage priced at overage rate`
- `Unit: roaming EU voice with 'eu_included' pack → 0 cents; without → roaming rate`
- `Property test: rated_amount monotonic in quantity beyond allowance`

#### 4.3 — Online charging: quota reservation & balance management

**What**: Nchf-style online charging with quota grant/reserve/debit against prepaid balances.

**Design**:
- `domain/charging/ocs.py`: `reserve_quota(subscription, requested_units) -> QuotaGrant` (atomically increments `balances.reserved_value` in Redis, fails if insufficient); `commit_usage(grant, actual_units)` debits `current_value`, releases remainder; `release(grant)` on session abort.
- Maps to Nchf operations: `initial` → reserve, `update` → commit+reserve next, `terminate` → final commit + release.

**Testing**:
- `Unit: reserve exceeding available balance → QuotaExhausted, no reservation`
- `Integration (Redis): concurrent reservations don't oversell balance (atomic)`
- `Unit: terminate commits actual < granted, releases difference`

#### 4.4 — `charging_records` (partitioned) & Nchf API

**What**: Implement the partitioned `charging_records` table and the Nchf-facing charging API.

**Design**: ORM + migration creating range-partitioned-by-`event_timestamp` table with monthly partitions and a partition-maintenance task. REST endpoints modelling 3GPP TS 32.291 Nchf:
- `POST /nchf-convergedcharging/v1/chargingdata` (create → `initial`)
- `POST /nchf-convergedcharging/v1/chargingdata/{ref}/update`
- `POST /nchf-convergedcharging/v1/chargingdata/{ref}/release`
Each persists a `charging_records` row (`nchf_session_id`, `nchf_operation`, `rating_group`, quota fields, `balance_before/after`). mTLS required (NF↔CHF). `source_system='ocs'|'mediation'`.

**Testing**:
- `Integration: create→update→release sequence persists three linked charging_records`
- `Integration: record lands in event_timestamp's monthly partition`
- `Conformance: Nchf request/response shape matches TS 32.291 schema fixtures`
- `Integration: request without mTLS client cert → 401`

#### 4.5 — Offline charging & CDR mediation ingest

**What**: Batch CDR file ingestion for postpaid/offline charging.

**Design**: `domain/charging/mediation.py`: parse CDR files (CSV + ASN.1-stub; pluggable `CdrParser`), normalise to `UsageEvent`, rate via RatingEngine, persist `charging_records` with `source_system='mediation'`, `source_file` ref. Celery task `ingest_cdr_file(path)`; idempotent on `(source_file, record_hash)`.

**Testing**:
- `Fixture: sample CDR CSV (100 records) → 100 charging_records, correct totals`
- `Unit: duplicate file re-ingest is idempotent (no double-charging)`
- `Unit: malformed CDR line → quarantined, batch continues`

---

## Phase 5: Order Management & Provisioning (TMF622 / TMF641)

### Purpose
Orchestrate the order lifecycle from sales channel through network fulfilment, creating subscriptions and triggering automated (zero-touch) provisioning. This connects catalogue → customer → subscription → service and is the operational backbone for service introduction.

### Tasks

#### 5.1 — `orders` table & order state machine

**What**: Implement `orders` (Suggestion 1) with an explicit lifecycle state machine.

**Design**: ORM per DDL (`order_type` 15-value, `status` 11-value, `items_json`, `provisioning_plan`, `fulfilment_json`, `rollback_order_id`). State machine:
```
acknowledged → validated → in_progress → pending_provisioning →
provisioning → pending_activation → activated → completed
                       ↓ (any) → failed → rolled_back
                       ↓ → cancelled
```
`domain/ordering/state_machine.py` enforces legal transitions; illegal transition raises `InvalidTransition`.

**Testing**:
- `Unit: validated→completed (skipping provisioning) rejected`
- `Unit: any non-terminal state → failed allowed`
- `Integration: order persists with provisioning_plan steps`

#### 5.2 — Order orchestration & TMF622 API

**What**: TMF622 `productOrderingManagement` endpoints + orchestration that validates eligibility, reserves resources, and creates subscriptions.

**Design**: `POST /productOrderingManagement/v4/productOrder`, `GET`, `PATCH` (cancel). On submit: validate items against catalogue + eligibility (Phase 3.3) → build `provisioning_plan` from product `provisioning_spec` → enqueue provisioning (Celery) → on activation create `subscriptions` + `service_instances`. TMF622-shaped payloads in `schemas/tmf622.py`.

**Testing**:
- `Integration: new_subscription order → validated, provisioning enqueued, subscription created on activation`
- `Integration: order for ineligible customer → status=failed with reason`
- `Conformance: TMF622 ProductOrder payload round-trips`

#### 5.3 — Provisioning engine & adapters (TMF641, zero-touch)

**What**: Execute the provisioning plan step-by-step against pluggable network adapters, with rollback.

**Design**:
```python
class ProvisioningStep(BaseModel):
    step: int
    action: str          # allocate_msisdn, provision_hlr, activate_sim, create_apn …
    status: Literal["pending","in_progress","completed","failed"]
    params: dict
    result: dict | None

class NetworkAdapter(Protocol):
    async def execute(self, action: str, params: dict) -> dict: ...
    async def rollback(self, action: str, params: dict) -> None: ...
```
`domain/provisioning/executor.py` runs steps sequentially, updating `orders.provisioning_plan`; on failure, rolls back completed steps in reverse and sets order `failed`. Ships with `MockNetworkAdapter` (deterministic) and a `SOL005Adapter` stub (ETSI NFV-MANO, completed in Phase 7). Emits service-to-resource links into `service_instances.resource_ids`.

**Testing**:
- `Unit: 3-step plan all succeed → order activated, service_instance created`
- `Unit: step 2 fails → step 1 rolled back, order=failed`
- `Integration (mock adapter): zero-touch new_subscription provisions MSISDN+APN end to end`

---

## Phase 6: Billing & Revenue Management

### Purpose
Turn charging records into invoices. Convergent billing aggregates recurring, one-off, and usage charges per cycle, applies tax, supports prepaid top-ups and postpaid invoicing, and runs dunning. Revenue assurance reconciles charging vs billed amounts.

### Tasks

#### 6.1 — `invoices` table & tax resolver

**What**: Implement `invoices` (Suggestion 1) and a jurisdiction-aware tax resolver.

**Design**: ORM per DDL (`status` 9-value, `invoice_type`, `line_items_json`, `tax_breakdown_json`, `dunning_level`, `credit_note_for`). `domain/billing/tax.py`: `TaxResolver.resolve(amount_cents, jurisdiction, product_type) -> TaxResult(rate, tax_cents, jurisdiction)` from a tenant-configurable rate table (default UK VAT 20%).

**Testing**:
- `Unit: GB jurisdiction 3459 cents → 692 tax (20%)`
- `Unit: tax-exempt product → 0 tax`
- `Integration: invoice_number unique per tenant enforced`

#### 6.2 — Invoice generation (billing run)

**What**: A billing-cycle job that aggregates uninvoiced `charging_records` into invoices.

**Design**: Celery task `run_billing_cycle(tenant_id, period_start, period_end)`: for each active postpaid/hybrid subscription, gather `charging_records WHERE invoiced=false AND event_timestamp in period`, add recurring charges from product pricing, build `line_items_json`, apply tax per line, compute totals, create invoice, mark records `invoiced=true` + set `invoice_id` (single transaction per customer). Render PDF via reportlab → `pdf_url`.

**Testing**:
- `Integration: subscription with recurring + 2.3GB overage → invoice with correct line items and total`
- `Unit: billing run is idempotent for a period (no double invoicing)`
- `Integration: charging_records marked invoiced after run`

#### 6.3 — Top-ups, payments & dunning

**What**: Prepaid balance top-up, payment recording, and overdue dunning escalation.

**Design**: `POST /balances/{id}/topup` (records `balance_topup` charging_record, increments balance). `POST /invoices/{id}/payment` (updates paid/outstanding, status). Celery beat `run_dunning()`: invoices past `due_date` and unpaid → increment `dunning_level`, emit notification, and at threshold trigger suspend order. Payment gateway behind `PaymentAdapter` (+ mock).

**Testing**:
- `Integration: topup 1000 cents → balance +1000, charging_record type=balance_topup`
- `Unit: partial payment → status=partially_paid, outstanding recalculated`
- `Integration: invoice 35 days overdue → dunning_level escalates, suspend triggered at threshold`

#### 6.4 — Revenue assurance reconciliation

**What**: Report comparing rated charging totals against invoiced totals to flag leakage.

**Design**: `domain/billing/revenue_assurance.py`: `reconcile(tenant_id, period) -> ReconciliationReport(rated_total, invoiced_total, variance, uninvoiced_records)`. Endpoint `GET /revenue-assurance?period=`.

**Testing**:
- `Unit: rated 100000, invoiced 99500 → variance 500 flagged`
- `Integration: uninvoiced records correctly counted`

---

## Phase 7: Network Inventory, Fault Management & NFV Integration (TMF638)

### Purpose
Deliver the OSS core: physical/logical network inventory with service-to-resource linking, alarm ingestion + correlation + trouble tickets, and ETSI NFV-MANO orchestration. This completes the BSS↔OSS span and enables the cross-domain queries that are the platform's differentiator.

### Tasks

#### 7.1 — `network_resources` & `service_instances` (TMF638/TMF641)

**What**: Implement both inventory tables (Suggestion 1) with self-referencing hierarchy and service-resource binding.

**Design**: ORM per DDL — `network_resources` self-referencing `parent_id`, 40+ `resource_type` values (physical/logical/5G NF), `capacity_json`, NFV descriptor fields, `health_status`; `service_instances` with `resource_ids[]` (GIN), `resource_mapping`, `service_params`, `sla_json`, `sla_status`. Repository helpers: `resources_for_service(id)`, `services_for_resource(id)` (the inverse query enabling impact analysis).

**Testing**:
- `Integration: site→rack→router hierarchy via parent_id; tree traversal returns descendants`
- `Integration: services_for_resource returns all service_instances bound to a router`
- `Conformance: TMF638 Resource payload round-trips`

#### 7.2 — TMF638 inventory API

**What**: `resourceInventoryManagement` endpoints: `GET/POST /resourceInventoryManagement/v4/resource`, hierarchy + service-link queries.

**Design**: TMF638-shaped Pydantic models; filters by type/status/health; `GET /resource/{id}/services` (impact view). RBAC: `network_engineer` write, `noc_operator` read.

**Testing**:
- `Integration: POST resource → 201; GET filters by health_status='critical'`
- `Integration: GET /resource/{id}/services lists affected services`

#### 7.3 — `alarms` table, ingestion & correlation

**What**: Implement `alarms` + alarm ingestion adapter + correlation engine.

**Design**: ORM per DDL (18 `alarm_type`, 5 `severity`, correlation fields, `affected_services[]`, `affected_subscribers`, `business_impact`). `integrations/nms/` `AlarmAdapter` (+ mock) ingests raw alarms → normalises → persists. `domain/fault/correlation.py`: rule-based correlation grouping alarms by `resource_id`/time-window/topology (a parent resource failure suppresses child alarms; sets `root_cause_alarm_id`, `is_root_cause`, `correlated_alarm_ids`). Computes `affected_services` via `services_for_resource` and `affected_subscribers` count.

**Testing**:
- `Unit: router-down alarm + 5 downstream link alarms → router alarm flagged root cause, children correlated/suppressed`
- `Integration: alarm on resource with 3 bound services → affected_services populated, affected_subscribers counted`
- `Unit: alarm clear auto-clears correlated children`

#### 7.4 — `trouble_tickets` table & auto-generation

**What**: Implement `trouble_tickets` and auto-create tickets from correlated root-cause alarms.

**Design**: ORM per DDL (status/priority/category enums, SLA interval tracking, `alarm_ids[]`, `work_log_json`). `domain/fault/ticketing.py`: root-cause critical/major alarm → create ticket (`source='alarm_auto'`, priority mapped from severity, linked `alarm_ids`, SLA targets from policy). Endpoints for manual create/assign/resolve. SLA breach detection job sets `sla_response_met`/`sla_resolve_met`.

**Testing**:
- `Integration: root-cause critical alarm → P1 ticket auto-created, alarm linked`
- `Unit: severity→priority mapping (critical→p1, minor→p3)`
- `Integration: ticket unresolved past sla_target_resolve → sla_resolve_met=false`

#### 7.5 — ETSI NFV-MANO (SOL005) adapter

**What**: Complete the `SOL005Adapter` for VNF/CNF lifecycle orchestration via the Os-Ma-nfvo interface.

**Design**: `integrations/nfv/sol005.py` implementing instantiate/scale/terminate against an NFVO REST API (NFV-SOL005); maps to `network_resources` of type `vnf`/`cnf` with `nfv_descriptor_id`/`nfv_instance_id`/`orchestrator`. Used by the provisioning executor (Phase 5.3) for virtualised services. Ships with a mock NFVO for tests.

**Testing**:
- `Integration (mock NFVO): instantiate VNF → network_resource created with nfv_instance_id, status in_service`
- `Unit: terminate VNF → resource status decommissioning then retired`

---

## Phase 8: Event-Driven Backbone (Kafka, AsyncAPI, CloudEvents) & TMF724

### Purpose
Introduce the real-time event backbone the product thesis demands: sub-second propagation of charging, provisioning, and fault events across components, replacing batch coupling. This unlocks AI streaming (Phase 10) and is documented as AsyncAPI/CloudEvents for interoperability (TMF724 Event Management).

### Tasks

#### 8.1 — Kafka producer/consumer + CloudEvents envelope

**What**: Event publishing/consuming infrastructure with a CloudEvents 1.0 envelope.

**Design**: `integrations/kafka/`: `EventPublisher.publish(topic, event: DomainEvent)` wrapping payload in CloudEvents (`ce_id`, `ce_source` = ODA component, `ce_type` e.g. `charging.session.created`, `ce_time`, `ce_specversion=1.0`); consumer base class with at-least-once + idempotency key. Per-tenant topics from `tenants.kafka_topics`.

**Testing**:
- `Integration (redpanda testcontainer): publish→consume round-trips CloudEvents envelope intact`
- `Unit: consumer deduplicates by ce_id (idempotent)`

#### 8.2 — Domain event emission

**What**: Emit events from charging, ordering, provisioning, billing, and fault flows.

**Design**: Define `DomainEvent` types: `charging.session.created/updated/released`, `order.status_changed`, `provisioning.completed/failed`, `invoice.generated`, `alarm.raised/correlated/cleared`, `balance.threshold_breached`. Each domain service publishes after committing its DB transaction (transactional-outbox pattern via an `outbox` table to avoid dual-write loss).

**Testing**:
- `Integration: completing an order publishes order.status_changed=completed`
- `Unit: outbox row written in same tx as state change; publisher drains outbox`
- `Integration: charging release publishes charging.session.released with amounts`

#### 8.3 — TMF724 Event Management API

**What**: TMF724-conformant event subscription/notification (hub + listener).

**Design**: `POST /eventManagement/v4/hub` (register listener callback + event filter), `DELETE /hub/{id}`; platform POSTs matching events to subscriber callbacks. AsyncAPI 3.0 channel docs generated in `asyncapi/`.

**Testing**:
- `Integration: register hub for alarm.raised → receive callback when alarm raised`
- `Conformance: TMF724 hub payload + notification shape validated`
- `CI: AsyncAPI document validates against spec`

---

## Phase 9: Northbound GraphQL, Operator Console & Self-Service Portals

### Purpose
Provide human-facing surfaces: a GraphQL API for efficient aggregate reads (360° view), a Next.js operator console (catalogue, orders, NOC, billing dashboards), and self-service portals for consumer and enterprise B2B subscribers — closing the gap features.md flags as underserved.

### Tasks

#### 9.1 — GraphQL 360° aggregate API

**What**: Strawberry GraphQL schema exposing the cross-domain 360° customer view in one query.

**Design**: `graphql/schema.py` with `Customer` type resolving nested `subscriptions`, `balances`, `invoices`, `serviceInstances`, `openTickets`, `churnRiskScore`. DataLoader batching to avoid N+1. Mounted at `/graphql`, OAuth2-guarded, tenant-scoped.

**Testing**:
- `Integration: single GraphQL query returns customer + subscriptions + balances + invoices`
- `Unit: DataLoader batches subscription lookups (1 query for N customers)`

#### 9.2 — Operator console (Next.js)

**What**: Operator web console for catalogue, orders, customers, NOC (alarms/tickets), and billing.

**Design**: Next.js 16 App Router + shadcn/ui; Server Components fetch via REST/GraphQL with OAuth2 (Authlib/NextAuth). Pages: Catalogue manager (create/version products), Order board (lifecycle kanban), Customer 360°, NOC dashboard (alarm list + correlation tree + ticket queue), Billing (invoice list, billing-run trigger), AI Hub (suggestions review). Role-gated navigation.

**Testing**:
- `E2E (Playwright): catalogue_manager creates a product → appears in list`
- `E2E: NOC operator acknowledges an alarm → status updates`
- `E2E: care_agent opens Customer 360° → sees subscriptions+balances+tickets`

#### 9.3 — Self-service portals (consumer + B2B)

**What**: Subscriber-facing portal (view usage/balance/bills, top-up, change plan) and a B2B enterprise portal (configure/order/monitor services).

**Design**: Separate Next.js route groups with subscriber-scoped OAuth2 (OIDC). Consumer: usage meter, balance top-up, invoice download, plan change (creates an order). B2B: catalogue browse (enterprise category), bulk ordering, service SLA dashboard, ticket raising.

**Testing**:
- `E2E: subscriber tops up balance → balance reflects increase`
- `E2E: subscriber changes plan → order created, visible to operator`
- `E2E: B2B user views service SLA dashboard with sla_status`

---

## Phase 10: AI-Native Intelligence Layer

### Purpose
Deliver the differentiating cross-domain intelligence: streaming anomaly/fraud detection on CDRs and network events, predictive churn combining network quality + billing signals, automated root-cause analysis linking OSS faults to BSS impact, and NL product-catalogue authoring — all with explainable, auditable outputs stored in `ai_suggestions`.

### Tasks

#### 10.1 — `ai_suggestions` table & suggestion lifecycle

**What**: Implement `ai_suggestions` (Suggestion 1) and the human-in-the-loop review workflow.

**Design**: ORM per DDL (12 `suggestion_type`, `confidence`, `target_entity_type/id`, `detail_json`, `explanation`, `model_id/version`, review fields, `status` incl. `auto_applied`). Service `record_suggestion(...)` + `review(id, decision, notes)` (accept/reject/auto-apply). Progressive-autonomy config: per suggestion_type threshold above which `auto_applied`, else pending review.

**Testing**:
- `Unit: suggestion with confidence below threshold → status=pending`
- `Integration: reviewer accepts churn suggestion → status=accepted, audit logged`

#### 10.2 — Streaming anomaly & fraud detection

**What**: Online anomaly detection over the CDR/charging Kafka stream.

**Design**: Kafka consumer feeding `river` incremental models (e.g. Half-Space Trees) per subscription/rating-group; flags outliers (sudden roaming spend spike, velocity fraud) → `ai_suggestions(type='fraud_detection'/'anomaly_alert')` with `detail_json` factors. Baselines updated online.

**Testing**:
- `Unit: injected spend spike 10x baseline → anomaly suggestion with factors`
- `Integration: stream of normal CDRs produces no false positives above threshold`

#### 10.3 — Cross-domain root-cause analysis & churn prediction

**What**: Link OSS fault events to BSS impact (RCA) and predict churn from combined network+billing signals.

**Design**:
- RCA: on `alarm.correlated` (root cause), assemble affected services → subscriptions → customers (the cross-domain JOIN that Suggestion 1 makes natural), compute revenue-at-risk, generate an LLM narrative `explanation`, emit `ai_suggestions(type='root_cause_analysis')`.
- Churn: scikit-learn model over features (recent service_degradation alarm count, bill_increase_pct, contract_end proximity, balance trend) → `customers.churn_risk_score` + `ai_suggestions(type='churn_prediction')` with `recommended_action` and `estimated_ltv_save_cents`.

**Testing**:
- `Integration: critical alarm on resource serving 50 subs → RCA suggestion lists revenue-at-risk + customers`
- `Unit: churn features → score in [0,1]; high-risk profile scores high`
- `Unit: LLM RCA narrative produced and stored as explanation (mocked LLM)`

#### 10.4 — Natural-language catalogue configuration (LLM)

**What**: Business user describes an offer in plain English; LLM generates a TMF620 product draft.

**Design**: `domain/ai/nl_catalogue.py`: prompt template instructing the model to emit a structured `ProductDraft` (name, product_type, pricing_json, allowances_json, charging_config, provisioning_spec) as JSON validated against the catalogue schema; user reviews/edits in the console before saving as a `draft` product. Uses AI Gateway (Anthropic primary). System prompt enforces schema + telecoms constraints.

System prompt skeleton:
```
You are a telecoms product catalogue assistant. Given a plain-English offer
description, output ONLY a JSON object matching the ProductDraft schema:
{name, product_type (one of <enum>), category, pricing_json{recurring,usage,overage},
 allowances_json, charging_config{charging_type,rating_group}, provisioning_spec}.
Currency is the tenant default. Do not invent fields outside the schema.
```

**Testing**:
- `Unit (mocked LLM): "20GB data, £29.99/mo, £2/GB overage" → valid ProductDraft with pricing_json populated`
- `Unit: LLM output failing schema validation → rejected with error, not persisted`

---

## Phase 11: 5G Network Slicing with Per-Slice SLA Billing

### Purpose
Add 5G network slice lifecycle management (S-NSSAI/SST/SD) with per-slice SLA monitoring and slice-aware billing — a capability features.md identifies as theoretically supported but not production-proven across vendors, making it a strong differentiator.

### Tasks

#### 11.1 — `network_slices` table & lifecycle

**What**: Implement `network_slices` (Suggestion 1) with ETSI ZSM-aligned lifecycle.

**Design**: ORM per DDL (`s_nssai`, `sst` 1/2/3, `sd`, `slice_type` 9-value, `sla_json`, `billing_model`, `resource_ids[]`, `max/current_subscribers`, `etsi_zsm_ref`). State machine `designed→instantiating→active→modifying→deactivating→terminated`. Provisioning via SOL005 adapter allocating `network_resources` (amf/smf/upf/nssf) to the slice.

**Testing**:
- `Unit: slice lifecycle transitions enforced`
- `Integration: instantiate slice allocates NF resources, status=active`

#### 11.2 — Slice SLA monitoring & slice-aware billing

**What**: Monitor slice SLA from performance data and bill per the slice `billing_model`.

**Design**: Performance ingestion sets `sla_status` (met/at_risk/breached) by comparing measured KPIs to `sla_json` (throughput/latency/reliability). Slice usage emits `charging_records(record_type='slice_usage', network_slice_id)`; billing run prices per `billing_model` (flat_rate/usage_based/sla_tiered with breach credits). SLA breach → `ai_suggestions(type='slice_scaling')`.

**Testing**:
- `Unit: measured latency above sla → sla_status=breached`
- `Integration: usage_based slice → slice_usage charging_records → invoice line`
- `Unit: sla_tiered breach applies credit to invoice`

---

## Phase 12: Hardening, Conformance, Deployment & Docs

### Purpose
Make the platform production-deployable and procurement-ready: security hardening to OWASP/NIST, TM Forum ODA conformance path, multi-region data residency, Helm-based K8s deployment, observability, and developer-portal documentation.

### Tasks

#### 12.1 — Security hardening (OWASP API Top 10, NIST 800-207)

**What**: Systematic security pass and automated security testing.

**Design**: Object-level authorization checks on every `/{id}` route (BOLA); response field allow-listing (no excessive exposure); rate limiting; input validation; secrets via env/secret store; mTLS on NF interfaces; RLS verified fail-closed. Add `schemathesis` + `bandit` + `pip-audit` to CI gate.

**Testing**:
- `Integration: tenant A token requesting tenant B's customer/{id} → 404 (no leak)`
- `Security scan: schemathesis finds no auth bypass; bandit/pip-audit clean`

#### 12.2 — ODA conformance & OpenAPI/AsyncAPI publication

**What**: Run TMF conformance checks and publish all API specs.

**Design**: Export OAS 3.1 per ODA component to `openapi/`; validate TMF620/622/638/641/724 responses against TM Forum CTK schemas; tag each component with its ODA functional block; generate a developer-portal site (Redoc/Scalar) from the specs.

**Testing**:
- `CI: all TMF endpoints validate against TM Forum schema fixtures`
- `CI: OpenAPI 3.1 + AsyncAPI 3.0 documents are valid`

#### 12.3 — Multi-region data residency

**What**: Enforce subscriber-data residency per tenant `data_residency`.

**Design**: Region-aware connection routing; data-residency tag propagated to `audit_log.data_residency`; deployment guidance for per-region Postgres + Kafka. Block cross-region reads of subscriber PII.

**Testing**:
- `Integration: tenant with data_residency='eu' stores/reads PII only from eu region binding`
- `Unit: cross-region PII read attempt → denied`

#### 12.4 — Helm chart, K8s deployment & observability

**What**: Production Helm chart with ODA-component-aligned services and full observability.

**Design**: `helm/bssoss/` deploying api, workers, console, and per-component services with HPA; OpenTelemetry traces (correlation_id end-to-end), Prometheus metrics, structured logs. `docker build` for all images succeeds; smoke-test job hits `/healthz`.

**Testing**:
- `CI: helm lint + template renders; docker images build`
- `Integration: deployed stack (kind cluster) passes /healthz + a create-order smoke test`
- `Observability: a cross-domain RCA flow produces a single linked trace by correlation_id`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (skeleton, RLS, auth, RBAC, audit)   ─── required by everything
    │
Phase 2: Customer Management                              ─── requires P1
    │
Phase 3: Product Catalogue (TMF620)                       ─── requires P1
    │
    ├── Phase 4: Convergent Charging (Nchf)               ─── requires P2, P3
    │       │
    │   Phase 5: Order Mgmt & Provisioning (TMF622/641)   ─── requires P2, P3 (uses P4 for subs)
    │       │
    │   Phase 6: Billing & Revenue Mgmt                   ─── requires P4
    │
    └── Phase 7: Inventory, Fault, NFV (TMF638)           ─── requires P1 (links to P5 services)
            │
Phase 8: Event Backbone (Kafka/CloudEvents/TMF724)        ─── requires P4, P5, P7
    │
    ├── Phase 9: GraphQL, Console, Portals                ─── requires P2–P7 (P9 can parallel P10/P11)
    ├── Phase 10: AI Intelligence Layer                   ─── requires P8 (stream) + P6/P7 data
    └── Phase 11: 5G Slicing + SLA Billing                ─── requires P6, P7
            │
Phase 12: Hardening, Conformance, Deployment, Docs        ─── requires all
```

**Parallelism opportunities:**
- After Phase 1: Phase 2 (Customer) and Phase 3 (Catalogue) can be built concurrently.
- After Phases 2+3: Phase 7 (OSS inventory/fault) can be built concurrently with the BSS chain (Phase 4 → 5 → 6).
- After Phase 8: Phases 9 (UI), 10 (AI), and 11 (Slicing) can be developed concurrently by separate streams.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pytest`), including testcontainers-backed integration tests.
3. Linting and formatting pass (`ruff check`, `ruff format --check`).
4. Type checking passes (`mypy --strict`).
5. Security checks pass (`bandit`, `pip-audit`) with no new high findings.
6. Docker build succeeds for all affected images; `docker-compose up` brings the stack to healthy.
7. The phase's headline capability works end-to-end (demonstrated by an integration or E2E test).
8. New configuration options documented in `config.py` and the README env table.
9. New REST endpoints appear in the auto-generated OpenAPI 3.1 spec; new events appear in the AsyncAPI 3.0 document.
10. Database migrations created via Alembic; `upgrade head` then `downgrade base` runs clean on an empty database; partitioned tables have a partition-maintenance path.
11. RLS policy added for every new operational (tenant-scoped) table and verified to fail closed.
12. Mutating actions emit an `audit_log` entry with the correct ODA component and TMF API tags where applicable.
13. TMF/3GPP/ETSI-facing endpoints validate against the relevant standard's schema fixtures (TMF620/622/638/641/724, Nchf TS 32.291, SOL005).
```

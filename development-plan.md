# Franchise Management Platform — Phased Development Plan

> Project: `460-franchise-management-platform` · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesizes `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The database design adopts **Data Model Suggestion 1 — Normalized Relational (PostgreSQL + RLS + PostGIS)** as the foundation, because it delivers ACID guarantees for financial/royalty data, engine-level multi-tenant isolation via Row-Level Security (RLS), and native territory geospatial support with zero additional infrastructure — the right trade-off for an open-source, self-hostable MVP. JSONB columns (borrowed from Suggestion 3) absorb the variability of audit checklists, KPI definitions, and franchisor settings without schema churn.

---

## Product Summary

**What it does.** A unified, AI-native platform that gives franchisors system-wide visibility into operations, compliance, training, and unit performance, while giving franchisees self-service access to standards, training, and benchmarks. It replaces the email/PDF/spreadsheet status quo for franchise networks of 20 to several thousand locations.

**Primary personas.**
- *Franchisor admin / ops director* — configures standards, audits, KPIs; monitors the whole network.
- *Field rep* — runs on-site audits and visits from a tablet/phone.
- *Franchisee owner* — runs one or more locations; reports revenue, completes training, responds to corrective actions.
- *Franchisee staff* — completes training, reads SOPs and announcements.

**Key differentiators.** Transparent open-source self-hostable platform (no incumbent offers this); AI-native (anomaly detection across the network, franchisee risk scoring, lead scoring, NLP feedback insights, audit-prep assistant) rather than bolt-on analytics; integrated compliance + training + operations + finance in one system.

**Deployment model.** Self-hosted / cloud / hybrid via Docker Compose (single-node MVP) and a Helm-ready container set later. Multi-tenant SaaS-capable from day one via RLS, but a single franchisor can self-host for data sovereignty.

**MVP scope (features.md "Must-have").** Centralized CRM (prospects + franchisees), multi-unit performance dashboard, franchisor↔franchisee communication portal, document/asset management, real-time reporting, mobile-responsive UI, QuickBooks accounting integration.

**Post-MVP (v1.1 / backlog).** Workflow/approvals, SCORM LMS, helpdesk ticketing, onboarding workflows, segmentation, advanced analytics/forecasting, multi-channel comms (email/SMS), then the AI layer: lead scoring, predictive risk, anomaly detection, NLP feedback, territory optimization, compliance automation.

**Standards the build must honor.** OAuth 2.0 (RFC 6749/6750) + OpenID Connect/SAML 2.0 for SSO; TLS 1.2/1.3 for all transport; OpenAPI 3.1 for the public API; JSON Schema (2020-12) for audit/agreement/FDD validation; SCORM 1.2 and 2004 (3rd/4th ed.) CMI data model for the LMS; FTC Franchise Rule 16 CFR 436 (FDD 23-item structure) for onboarding documents; ASC 606/952 + IFRS 15 revenue recognition semantics for royalties; GDPR/CCPA data-subject rights and consent; SOC 2 / ISO 27001 controls (audit log, access control); MCP for the AI insight/recommendation surface.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | **Python 3.12** | The differentiating value is AI-native (anomaly detection, risk/lead scoring, NLP feedback). Python gives first-class access to scikit-learn, statsmodels, the OpenAI/Anthropic SDKs, and MCP tooling, while still being excellent for REST APIs. |
| API framework | **FastAPI** | Async, Pydantic-validated, and emits **OpenAPI 3.1** automatically (standards.md requirement). Dependency-injection model maps cleanly to per-request RLS tenant context. |
| Validation / schemas | **Pydantic v2** | One source of truth for request/response models and JSON Schema export for FDD/audit/agreement validation (standards.md). |
| ORM / DB toolkit | **SQLAlchemy 2.0 + Alembic** | Mature, supports raw SQL escape hatches for PostGIS and window-function-heavy reporting; Alembic gives versioned, CI-tested migrations (Suggestion 1 calls for Flyway/Liquibase-style migrations). |
| Database | **PostgreSQL 16 + PostGIS 3.4** | Adopts Data Model Suggestion 1: ACID for money, RLS for tenant isolation, PostGIS for territories, `tsvector` full-text for SOP/asset search, JSONB for variable structures. No second datastore needed for MVP. |
| Cache / queue broker | **Redis 7** | Session/dashboard cache + Celery broker for async work (webhooks, QuickBooks sync, SCORM rollups, AI scoring). |
| Task queue | **Celery + Redis** | Royalty runs, accounting sync, anomaly scans, email/SMS sends, and LLM calls are long-running and bursty; they must not block API requests. |
| Object storage | **S3-compatible (MinIO self-hosted / AWS S3 cloud)** | Audit photos/videos, SCORM packages, marketing assets, signed documents. MinIO keeps self-host deployments single-stack. |
| AuthN / SSO | **OAuth 2.0 + OIDC (Authlib), SAML 2.0 (python3-saml)** | standards.md mandates OAuth 2.0 and enterprise SSO. JWT access tokens; refresh-token rotation. |
| Frontend | **Next.js 15 (App Router) + React + TypeScript + Tailwind + shadcn/ui** | Mobile-responsive is mandatory (field teams on tablets/phones). SSR for fast dashboards; PWA for offline audit capture. shadcn/ui for accessible components. |
| Charts | **Recharts** | Dashboard KPI trend/benchmark visualizations. |
| Maps / geo UI | **MapLibre GL JS** | Render territory polygons and location points (open-source, no API-key lock-in). |
| AI / LLM | **Pluggable provider via a `LLMClient` abstraction (Anthropic + OpenAI), exposed over MCP** | standards.md recommends MCP for AI insights. Classical ML (scikit-learn) for risk/anomaly scoring; LLMs for NLP feedback summarization and the audit-prep/chatbot assistant. |
| Accounting integration | **QuickBooks Online API (intuit-oauth + REST)** | Baseline integration per README/features. Connector interface abstracts other accounting systems later. |
| Payments | **Stripe** | Royalty/fee collection (standards.md lists Stripe). |
| Email / SMS | **SMTP/SendGrid + Twilio** behind a `NotificationChannel` interface | Multi-channel comms (v1.1). |
| Containerisation | **Docker + Docker Compose (MVP), Helm chart (later)** | Self-hosted/cloud/hybrid deployment requirement. |
| Testing | **pytest + pytest-asyncio + httpx + testcontainers (Postgres) ; Playwright (e2e UI) ; Vitest (frontend unit)** | Real Postgres in CI via testcontainers so RLS and PostGIS are exercised, not mocked. |
| Lint / format / types | **ruff + black + mypy (backend) ; eslint + prettier + tsc (frontend)** | Enforced in CI; gate for Definition of Done. |
| Package managers | **uv (Python) ; pnpm (frontend)** | Fast, lockfile-based, reproducible. |
| API client SDK | **Generated from OpenAPI 3.1 (openapi-typescript)** | Frontend stays in sync with backend contract automatically. |

### Project Structure

```
franchise-management-platform/
├── docker-compose.yml              # postgres+postgis, redis, minio, api, worker, web
├── docker-compose.prod.yml
├── .env.example
├── Makefile                        # make up / test / lint / migrate / seed
├── deploy/
│   └── helm/                       # Helm chart (later phase)
├── backend/
│   ├── pyproject.toml              # uv-managed
│   ├── alembic.ini
│   ├── Dockerfile
│   ├── alembic/
│   │   └── versions/               # one migration per phase increment
│   ├── app/
│   │   ├── main.py                 # FastAPI app factory, router registration
│   │   ├── config.py               # Pydantic Settings (env-driven)
│   │   ├── db/
│   │   │   ├── session.py          # async engine, session, RLS tenant-context dep
│   │   │   ├── base.py             # declarative base, mixins (TenantMixin, TimestampMixin)
│   │   │   └── rls.py              # SET app.current_franchisor_id helpers
│   │   ├── core/
│   │   │   ├── security.py         # JWT, password hashing, OAuth/OIDC, SAML
│   │   │   ├── rbac.py             # permission codes, require_permission() dep
│   │   │   ├── audit_log.py        # write-through audit logging
│   │   │   └── errors.py           # typed exceptions -> RFC 7807 problem+json
│   │   ├── models/                 # SQLAlchemy ORM (one module per bounded context)
│   │   │   ├── tenancy.py  franchisee.py  ops_compliance.py
│   │   │   ├── training.py  finance.py  comms.py  analytics.py
│   │   ├── schemas/                # Pydantic request/response models
│   │   ├── services/               # business logic (no FastAPI imports)
│   │   │   ├── royalty.py  audit.py  kpi.py  onboarding.py  scorm.py ...
│   │   ├── api/
│   │   │   └── v1/                  # routers grouped by context
│   │   ├── integrations/
│   │   │   ├── quickbooks.py  stripe_pay.py  notifications/ (email,sms)
│   │   ├── ai/
│   │   │   ├── llm_client.py        # provider-agnostic LLM wrapper
│   │   │   ├── mcp_server.py        # MCP tool surface for insights
│   │   │   ├── risk_model.py  anomaly.py  lead_scoring.py  feedback_nlp.py
│   │   └── workers/
│   │       ├── celery_app.py
│   │       └── tasks/               # royalty_run, qbo_sync, anomaly_scan, sends
│   └── tests/
│       ├── conftest.py             # testcontainers Postgres, RLS fixtures
│       ├── unit/  integration/  e2e/  fixtures/
├── frontend/
│   ├── package.json                # pnpm
│   ├── Dockerfile
│   ├── app/                        # Next.js App Router
│   │   ├── (auth)/  (franchisor)/  (franchisee)/
│   │   └── api/                    # BFF proxy where needed
│   ├── components/                 # shadcn/ui-based
│   ├── lib/api/                    # generated OpenAPI client
│   └── tests/                      # Vitest + Playwright
└── docs/
    ├── architecture.md  api.md  deployment.md
```

The structure groups by **bounded context** (tenancy, franchisee, ops/compliance, training, finance, comms, analytics) so each phase adds files within existing modules rather than restructuring.

---

## Phase 1: Foundation — Tenancy, Auth, RBAC, and Platform Scaffolding

### Purpose
Stand up the runnable skeleton every later phase depends on: containerized Postgres+PostGIS/Redis/MinIO, the FastAPI app factory, migrations, multi-tenant data isolation via RLS, authentication (password + JWT, OIDC/SAML stubs), role-based access control, and the platform-wide audit log. After this phase a franchisor and its users exist, can log in, and all queries are tenant-isolated at the database engine.

### Tasks

#### 1.1 — Repo, containers, and config
**What**: Bootstrap the monorepo, Docker Compose stack, and environment-driven configuration.

**Design**:
- `docker-compose.yml` services: `postgres` (`postgis/postgis:16-3.4`), `redis:7`, `minio`, `api`, `worker`, `web`. Health checks on each; `api`/`worker` wait for `postgres` healthy.
- `app/config.py` using `pydantic_settings.BaseSettings`:
```python
class Settings(BaseSettings):
    database_url: str            # postgresql+asyncpg://...
    redis_url: str
    jwt_secret: str
    jwt_access_ttl_seconds: int = 900
    jwt_refresh_ttl_seconds: int = 1209600
    s3_endpoint: str; s3_bucket: str; s3_access_key: str; s3_secret_key: str
    environment: Literal["dev","test","staging","prod"] = "dev"
    cors_origins: list[str] = []
    model_config = SettingsConfigDict(env_file=".env")
```
- `Makefile` targets: `up`, `down`, `migrate`, `seed`, `test`, `lint`, `typecheck`.
- `app/main.py` app factory registers routers, exception handlers (RFC 7807 problem+json), CORS, and the OpenAPI 3.1 metadata block.

**Testing**:
- `Unit: Settings loads from env → all required fields present, defaults applied`.
- `Unit: missing DATABASE_URL → ValidationError naming the field`.
- `Integration: docker compose up → GET /healthz returns {status:"ok", db:"ok", redis:"ok"} 200`.

#### 1.2 — Database session, RLS tenant context, base mixins
**What**: Async SQLAlchemy engine plus a per-request mechanism that sets the Postgres RLS tenant variable.

**Design**:
- `db/session.py`: `async_engine`, `async_sessionmaker`. FastAPI dependency `get_session()` opens a session and, after resolving the authenticated user, runs `SET LOCAL app.current_franchisor_id = :id` (Suggestion 1, lines 29-39).
- `db/base.py` mixins:
```python
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())

class TenantMixin:
    franchisor_id: Mapped[UUID] = mapped_column(ForeignKey("franchisors.id"), nullable=False, index=True)
```
- Every tenant-scoped table gets RLS enabled + a `tenant_isolation` policy (`USING (franchisor_id = current_setting('app.current_franchisor_id')::uuid)`), created in the migration.

**Testing**:
- `Integration (real Postgres): set tenant A, insert franchisee; switch to tenant B, SELECT → 0 rows` (RLS proven at engine level).
- `Integration: query with no tenant var set → policy returns 0 rows (fail-closed)`.

#### 1.3 — Tenancy & user migration + models (Suggestion 1 §1)
**What**: Implement `franchisors`, `users`, `roles`, `permissions`, `role_permissions`, `user_roles` exactly per Suggestion 1 lines 47-122.

**Design**:
- ORM models in `models/tenancy.py`. `users` UNIQUE(`franchisor_id`,`email`); `role` enum-constrained at the schema layer via Pydantic, not a DB enum (flexibility).
- Alembic migration `0001_tenancy` creates tables, RLS policies, and indexes.
- Seed script inserts system permissions (`audit.create`, `training.manage`, `royalty.view`, ...) and the six system roles from README/Suggestion 1 (`franchisor_admin`, `ops_director`, `field_rep`, `franchisee_owner`, `franchisee_manager`, `franchisee_staff`).

**Testing**:
- `Unit: seed → 6 system roles + all permission codes present`.
- `Integration: create franchisor + admin user → row persisted with hashed password`.
- `Migration test: alembic upgrade head from empty DB succeeds; downgrade base succeeds`.

#### 1.4 — Authentication (password + JWT) and SSO stubs
**What**: Login, token issuance/refresh, password hashing; OIDC and SAML behind a provider interface.

**Design**:
- `core/security.py`: argon2 password hashing; `create_access_token(user_id, franchisor_id, roles)` → JWT (HS256, `jwt_secret`); refresh-token rotation stored in Redis with `jti`.
- Endpoints: `POST /v1/auth/login` (email+password→tokens), `POST /v1/auth/refresh`, `POST /v1/auth/logout`.
- SSO: `AuthProvider` protocol with `oidc` (Authlib) and `saml` (python3-saml) implementations; routes `GET /v1/auth/sso/{provider}/start` and `/callback` wired but provider config optional in MVP.
- All transport TLS-terminated (documented; enforced via reverse proxy in deploy).

**Testing**:
- `Unit: hash+verify password round-trip`.
- `Unit: expired token → 401 with problem+json type "token_expired"`.
- `Integration: login valid → 200 + access+refresh; refresh → new access; reused old refresh → 401 (rotation enforced)`.
- `Integration: login wrong password → 401, audit_log records failed_login`.

#### 1.5 — RBAC dependency + audit log
**What**: `require_permission("code")` FastAPI dependency and write-through audit logging.

**Design**:
- `core/rbac.py`: resolves the user's effective permission set (union of role permissions, optionally franchisee-scoped via `user_roles.franchisee_id`); raises 403 problem+json on miss.
- `core/audit_log.py`: `record(action, entity_type, entity_id, changes, request)` writes to `audit_log` (Suggestion 1 lines 953-967) capturing IP and user-agent. A SQLAlchemy event hook auto-diffs `update`/`delete` into the `changes` JSONB (SOC 2 / ISO 27001 control).

**Testing**:
- `Unit: user with role lacking permission → require_permission raises 403`.
- `Integration: update franchisee → audit_log row with changes {field:{old,new}}`.
- `Integration: franchisee_owner scoped to franchisee X cannot read franchisee Y → 403`.

---

## Phase 2: Franchisee Lifecycle Core (CRM, Brands, Franchisees, Locations)

### Purpose
Deliver the heart of the MVP CRM: brands, prospects with a sales pipeline, franchisees, and their locations. This is the entity backbone every other module references (audits attach to locations, royalties to agreements, training to users at franchisees). Implements Suggestion 1 §2 (lines 132-263, 314-349).

### Tasks

#### 2.1 — Brands, prospects, prospect activities
**What**: CRUD + pipeline management for `franchise_brands`, `franchise_prospects`, `prospect_activities`.

**Design**:
- Models/migration per Suggestion 1 lines 132-176.
- Pipeline stages enum (schema layer): `inquiry → qualified → application → discovery_day → fdd_review → approved → declined → withdrawn`.
- Endpoints: `POST/GET/PATCH /v1/brands`, `/v1/prospects` (filter by stage, assigned_to, source), `POST /v1/prospects/{id}/activities`, `POST /v1/prospects/{id}/advance` (stage transition validated against an allowed-transitions map; illegal transition → 409).
- `lead_score` column present but left null until Phase 9 (AI).

**Testing**:
- `Unit: advance inquiry→qualified allowed; advance approved→inquiry → 409`.
- `Integration: create prospect, log call activity, GET → activity in timeline`.
- `Integration: filter prospects by stage=qualified → only qualified returned, tenant-scoped`.

#### 2.2 — Franchisees & agreements
**What**: `franchisees`, `franchise_agreements`, `franchise_agreement_tiers`.

**Design**:
- Models per Suggestion 1 lines 178-234. Franchisee status lifecycle: `onboarding → pre_opening → open → {good_standing|probation|at_risk} → {suspended|terminated|transferred}`.
- Agreement carries royalty config (`royalty_type` ∈ `percentage_gross|percentage_net|fixed_monthly|tiered`, rate, ad-fund rate, minimum, tiers). This is the source of truth Phase 6 royalty engine reads.
- Endpoints: `POST/GET/PATCH /v1/franchisees`, `POST /v1/franchisees/{id}/agreements`, agreement tiers nested create.
- Convert flow: `POST /v1/prospects/{id}/convert` creates a franchisee linked via `prospect_id` (only from stage `approved`).

**Testing**:
- `Unit: tiered agreement requires ≥1 tier → 422 if none`.
- `Integration: convert approved prospect → franchisee created, prospect.stage unchanged-but-linked`.
- `Integration: convert non-approved prospect → 409`.

#### 2.3 — Locations with PostGIS geometry
**What**: `locations` table with point geometry and spatial index.

**Design**:
- Model per Suggestion 1 lines 236-265, including `geom GEOMETRY(Point,4326)` + GIST index. On create/update, derive `geom` from lat/long via `ST_SetSRID(ST_MakePoint(lng,lat),4326)`.
- `operating_hours` JSONB validated by Pydantic schema (`{mon:{open,close},...}`).
- Endpoints: `POST/GET/PATCH /v1/locations`, `GET /v1/locations/nearby?lat&lng&radius_km` (PostGIS `ST_DWithin`).

**Testing**:
- `Integration: create location with lat/lng → geom populated; nearby query within radius returns it, outside excludes it`.
- `Unit: invalid operating_hours shape → 422`.

#### 2.4 — Onboarding checklists
**What**: `onboarding_checklists`, `onboarding_checklist_items`, `franchisee_onboarding_progress` (Suggestion 1 lines 314-349).

**Design**:
- Checklist templates per brand with versioned items (`due_days_offset`, `responsible_role`, `required`).
- On franchisee creation in `onboarding` status, materialize progress rows from the active checklist.
- Endpoints: `GET /v1/franchisees/{id}/onboarding`, `PATCH /v1/onboarding-progress/{id}` (mark completed/waived with evidence_url).
- FDD linkage note: an onboarding item type can reference an FTC FDD document (Phase 3 docs) to enforce the 14-day pre-signing delivery rule (16 CFR 436) — store `fdd_delivered_at` on the item and flag if agreement signed <14 days later.

**Testing**:
- `Integration: new franchisee → progress rows materialized for all active checklist items`.
- `Unit: completing a required item with no evidence when required → 422`.
- `Unit: agreement signed 10 days after FDD delivery → compliance warning flag set`.

---

## Phase 3: Operations Standards, Documents & Asset Management

### Purpose
Provide the versioned operations library (SOPs, brand guidelines, FDD documents) and marketing-asset management with full-text search and acknowledgement tracking — an MVP "must-have" and the substrate compliance audits reference. Implements Suggestion 1 §3 doc tables (lines 359-409) and marketing assets (lines 864-882).

### Tasks

#### 3.1 — Object storage abstraction
**What**: Upload/download service over S3-compatible storage.

**Design**:
- `integrations/storage.py`: `put_object`, `presigned_get_url`, `presigned_put_url` (direct browser upload). Keys namespaced `{franchisor_id}/{entity}/{uuid}/{filename}`.
- Virus-scan hook stub; content-type allowlist per upload context.

**Testing**:
- `Integration (MinIO): presigned put then get round-trips a file`.
- `Unit: disallowed content-type → 415`.

#### 3.2 — Operations documents with versioning, FTS, acknowledgements
**What**: `document_categories`, `operations_documents`, `document_versions`, `document_acknowledgements`.

**Design**:
- Models per Suggestion 1 lines 359-409. `search_vector TSVECTOR` GIN-indexed; trigger or service updates it from `title + content_body` on write.
- Versioning: editing a published doc creates a new `document_versions` row and increments `version`; old acknowledgements remain pinned to their version.
- `document_type` includes `fdd` for FTC Franchise Rule documents (23-item structure validated by a JSON Schema stored alongside).
- Endpoints: `POST/GET/PATCH /v1/documents`, `GET /v1/documents/search?q=`, `POST /v1/documents/{id}/acknowledge`, `GET /v1/documents/{id}/versions`.

**Testing**:
- `Integration: publish doc, full-text search matching token → returned ranked`.
- `Integration: edit published doc → version increments, prior version retrievable`.
- `Integration: acknowledge v1, doc bumped to v2 → user shows acknowledged v1, pending v2`.

#### 3.3 — Marketing assets with branding-compliance approval
**What**: `marketing_assets` with approval gate and download counting.

**Design**:
- Model per Suggestion 1 lines 864-882. Assets require `approved=true` (set by user with `marketing.approve`) before franchisees can download. Thumbnail generation queued to a worker for image/video.
- Endpoints: `POST/GET /v1/assets`, `POST /v1/assets/{id}/approve`, `GET /v1/assets/{id}/download` (increments `download_count`, returns presigned URL; 403 if unapproved and caller is franchisee).

**Testing**:
- `Integration: franchisee downloads unapproved asset → 403; after approve → 200 + download_count++`.

---

## Phase 4: Communication Hub (MVP must-have)

### Purpose
Enable franchisor↔franchisee communication: announcements with read/acknowledge tracking, and direct/threaded messaging. Completes the MVP communication-portal requirement. Implements Suggestion 1 §6 announcement/message tables (lines 786-832).

### Tasks

#### 4.1 — Announcements with targeting & acknowledgements
**What**: `announcements`, `announcement_recipients`.

**Design**:
- Models per Suggestion 1 lines 786-813. Targeting by audience (`all|owners|managers|staff|custom`), `target_brands`, `target_regions`. On publish, a worker fans out recipient rows for the resolved audience and (when configured) dispatches in-app + email/SMS via the notification interface (channels stubbed until Phase 7).
- Endpoints: `POST /v1/announcements`, `POST /v1/announcements/{id}/publish`, `GET /v1/announcements` (recipient-scoped feed), `POST /v1/announcements/{id}/read`, `/acknowledge`.

**Testing**:
- `Integration: publish to audience=owners → recipient rows only for owner users in scope`.
- `Integration: requires_acknowledgement → ack endpoint sets acknowledged_at; read-only ack on non-ack announcement → 409`.

#### 4.2 — Direct & threaded messaging
**What**: `messages`, `message_recipients`.

**Design**:
- Models per Suggestion 1 lines 815-832. `thread_id` self-references the first message; replies inherit it. Unread counts via index on `message_recipients(recipient_id) WHERE read_at IS NULL`.
- Endpoints: `POST /v1/messages`, `GET /v1/threads`, `GET /v1/threads/{id}`, `POST /v1/messages/{id}/read`.

**Testing**:
- `Integration: reply to message → same thread_id; recipient unread count increments then clears on read`.

---

## Phase 5: Revenue Reporting & Performance Dashboard (MVP must-have)

### Purpose
Capture unit revenue, compute KPIs, and render the multi-unit performance dashboard with rankings, trends, and network benchmarks — the centerpiece MVP deliverable. Implements Suggestion 1 revenue_reports (lines 677-700) and §7 KPI tables (lines 892-927).

### Tasks

#### 5.1 — Revenue reports ingestion
**What**: `revenue_reports` with manual entry and CSV bulk upload.

**Design**:
- Model per Suggestion 1 lines 677-700; UNIQUE(`location_id`,`reporting_period`,`period_type`) prevents double-reporting. `source` ∈ `manual|pos_import|accounting_sync`.
- Endpoints: `POST /v1/revenue-reports`, `POST /v1/revenue-reports/import` (CSV → rows, validation report), `POST /v1/revenue-reports/{id}/verify`.

**Testing**:
- `Unit: duplicate period for location → 409`.
- `Integration: CSV with one bad row → valid rows inserted, error report lists bad row index+reason`.

#### 5.2 — KPI definitions & snapshot computation
**What**: `kpi_definitions`, `kpi_snapshots`, and a computation service.

**Design**:
- Models per Suggestion 1 lines 892-925. Seed standard KPIs from research.md: `same_store_sales_growth`, `auv`, `labor_cost_ratio`, `inventory_turnover`, `royalty_collection_rate`, `nps`. `calculation_formula` stored as a safe expression evaluated against a revenue-report feature dict.
- A Celery task `compute_kpis(period)` aggregates revenue_reports → kpi_snapshots, fills `previous_value`, `change_pct`, `benchmark_network_avg`, `benchmark_top_quartile` (per-brand window functions), and sets `status` against warning/critical thresholds.

**Testing**:
- `Unit: labor_cost_ratio = labor_cost/gross_revenue computed correctly from a report`.
- `Integration: two periods → same_store_sales_growth change_pct correct; status flips to "critical" below threshold`.
- `Integration: network_avg/top_quartile match hand-computed values across 5 locations`.

#### 5.3 — Dashboard & benchmarking API + frontend
**What**: Aggregated dashboard endpoints and the franchisor/franchisee dashboard UI.

**Design**:
- Endpoints: `GET /v1/dashboard/overview` (network rollup), `GET /v1/franchisees/{id}/scorecard`, `GET /v1/rankings?kpi=&period=` (ranked locations with percentile), `GET /v1/kpis/{code}/trend?location_id=`.
- Frontend: franchisor overview (network KPI tiles, ranking table, MapLibre location map colored by status), franchisee scorecard (own units vs network benchmark via Recharts). Mobile-responsive; PWA shell.
- Heavy rollups served from a materialized view refreshed by the KPI task (Suggestion 1 lines 1033, 1023).

**Testing**:
- `Integration: rankings endpoint orders locations by KPI desc with correct percentiles`.
- `E2E (Playwright): login as franchisor → overview renders KPI tiles + ranking; login as franchisee → sees only own units`.
- `E2E mobile viewport: dashboard layout reflows, no horizontal scroll`.

---

## Phase 6: Royalties, Invoicing & Payments

### Purpose
Turn reported revenue into royalty calculations, invoices, and tracked payments with GAAP/IFRS-aligned revenue semantics and Stripe collection — closing the franchisor financial loop. Implements Suggestion 1 §5 (lines 677-776).

### Tasks

#### 6.1 — Royalty calculation engine
**What**: `royalty_calculations` plus a deterministic engine driven by agreement terms.

**Design**:
- Service `services/royalty.py: calculate(agreement, revenue_report) -> RoyaltyResult`. Supports all `royalty_type`s incl. tiered (walks `franchise_agreement_tiers` by revenue band) and `minimum_royalty` flooring (`minimum_applied=true`). Computes ad-fund and technology fees. All money in `NUMERIC`/`Decimal` — never float.
- Aligns to ASC 606/952 + IFRS 15: royalties recognized in the period of the underlying reported revenue (`billing_period == revenue_report.reporting_period`).
- Celery `royalty_run(franchisor_id, period)` batches all open franchisees; idempotent (re-run replaces same-period calc rows in a transaction).

**Testing**:
- `Unit: percentage_gross 6.5% on $100k → $6,500 exact Decimal`.
- `Unit: tiered (0-50k@7%, 50k+@5%) on $80k → 50k*7% + 30k*5% = $5,000`.
- `Unit: computed below minimum → minimum_applied true, total_due == minimum`.
- `Integration: royalty_run twice for same period → no duplicate rows (idempotent)`.

#### 6.2 — Invoicing
**What**: `invoices`, `invoice_line_items` generated from royalty calculations.

**Design**:
- Model per Suggestion 1 lines 724-758. `invoice_number` unique per franchisor; line types `royalty|ad_fund|technology_fee|late_fee|credit|other`. Status lifecycle `draft→sent→viewed→partially_paid→paid|overdue|written_off`. Overdue transition by a daily worker comparing `due_date` and `balance_due`.
- Endpoints: `POST /v1/invoices/generate` (from a royalty run), `POST /v1/invoices/{id}/send`, `GET /v1/invoices`.

**Testing**:
- `Integration: generate invoice from royalty calc → line items sum to subtotal; total_amount = subtotal+tax+late_fee-credits`.
- `Integration: overdue worker flips past-due unpaid invoice to "overdue"`.

#### 6.3 — Payments & Stripe
**What**: `payments` with Stripe collection and reconciliation.

**Design**:
- Model per Suggestion 1 lines 760-776. `POST /v1/invoices/{id}/pay` creates a Stripe PaymentIntent; webhook `POST /v1/webhooks/stripe` (signature-verified) updates payment + invoice `balance_due`/status atomically. ACH/wire/check recorded manually.

**Testing**:
- `Integration (mocked Stripe): valid webhook signature → payment completed, invoice balance reduced, status paid`.
- `Integration: invalid webhook signature → 401, no state change`.
- `Unit: partial payment → invoice partially_paid, balance_due correct`.

#### 6.4 — QuickBooks Online sync
**What**: Two-way-ish accounting integration (invoices/payments → QBO).

**Design**:
- `integrations/quickbooks.py`: OAuth 2.0 connect flow (`/v1/integrations/qbo/connect`/`callback`, tokens encrypted at rest); `sync_invoice`, `sync_payment` push to QBO; `pull_revenue` optional import into revenue_reports (`source=accounting_sync`). Runs via worker; failures retried with backoff and surfaced in an integration-status endpoint.

**Testing**:
- `Integration (mocked QBO API): connect stores tokens; sync_invoice maps fields to QBO Invoice payload`.
- `Integration: token refresh on 401 from QBO → retried with new token`.

---

## Phase 7: Multi-Channel Notifications, Helpdesk & Async Workflows

### Purpose
Add the v1.1 engagement and support layer: real email/SMS/in-app delivery behind one interface, a helpdesk ticketing system, and the corrective-action workflow that operations depends on. Implements Suggestion 1 support tables (lines 834-862) and corrective_actions (lines 495-517).

### Tasks

#### 7.1 — Notification channels
**What**: `NotificationChannel` interface with in-app, email (SMTP/SendGrid), SMS (Twilio).

**Design**:
- `integrations/notifications/`: `send(channel, recipient, template, context)`. Per-user channel preferences; delivery records update `announcement_recipients.delivery_channel/delivered_at` and message delivery. Templated, localized (`users.locale`).

**Testing**:
- `Integration (mocked Twilio/SendGrid): announcement publish dispatches per-recipient preferred channel; delivery timestamps recorded`.
- `Unit: user opted out of SMS → falls back to email/in-app`.

#### 7.2 — Helpdesk ticketing
**What**: `support_tickets`, `ticket_comments`.

**Design**:
- Models per Suggestion 1 lines 834-862. Ticket lifecycle `open→assigned→in_progress→waiting_on_requestor→resolved→closed`; internal vs requestor-visible comments; CSAT 1-5 on close.
- Endpoints: `POST/GET/PATCH /v1/tickets`, `POST /v1/tickets/{id}/comments`, `POST /v1/tickets/{id}/assign`.

**Testing**:
- `Integration: requestor cannot see is_internal=true comments`.
- `Unit: close without resolution → 409; reopen resolved ticket allowed`.

#### 7.3 — Corrective-action workflow with escalation
**What**: `corrective_actions` (will be referenced by Phase 8 audits).

**Design**:
- Model per Suggestion 1 lines 495-517. Status `open→acknowledged→in_progress→resolved→verified`, with `escalated`/`overdue`. A worker escalates overdue actions: bumps `escalation_level`, sets `escalated_to` (up the responsible-role chain), and notifies via Phase 7.1.

**Testing**:
- `Integration: action past due_date and unresolved → worker marks overdue, escalation_level++, escalated_to set, notification sent`.
- `Unit: verify before resolve → 409`.

---

## Phase 8: Compliance Audits (field-grade, offline-capable)

### Purpose
Deliver configurable audit templates, scored audits with photo/video evidence, and automatic corrective-action generation — the defensible-compliance capability and a major differentiator. Implements Suggestion 1 audit tables (lines 411-493) and field_visits (lines 295-312).

### Tasks

#### 8.1 — Audit templates (configurable checklists)
**What**: `audit_templates`, `audit_template_sections`, `audit_template_items`.

**Design**:
- Models per Suggestion 1 lines 411-447. Item `question_type` ∈ `yes_no|score_1_5|score_1_10|text|photo|multi_select`; `is_critical` (auto-fail), `requires_photo`, section `weight`, template `passing_score`. Versioned; templates exportable/importable as JSON validated by a JSON Schema (standards.md JSON Schema 2020-12).
- Endpoints: `POST/GET /v1/audit-templates` (nested sections/items), `POST /v1/audit-templates/{id}/version`.

**Testing**:
- `Unit: template import with invalid item type → 422 citing JSON Schema path`.
- `Unit: weighted scoring config validated (weights sum sanity)`.

#### 8.2 — Conduct audits with evidence + scoring
**What**: `audits`, `audit_responses`, `audit_evidence`, and the scoring service.

**Design**:
- Models per Suggestion 1 lines 449-493. Scoring service computes weighted `total_score`, `score_percentage`, and `passed` (false if any `is_critical` item failed regardless of score). Evidence uploaded via presigned URLs (Phase 3.1) with geotag (`latitude`/`longitude`) and capture timestamp for defensibility.
- On completion, non-compliant responses auto-generate `corrective_actions` (Phase 7.3) with severity mapped from criticality.
- Offline-first PWA: audits cached locally and synced; idempotent submit keyed by client-generated audit UUID.
- Endpoints: `POST /v1/audits` (schedule), `PATCH /v1/audits/{id}/responses` (bulk), `POST /v1/audits/{id}/complete`, `GET /v1/audits/{id}`.

**Testing**:
- `Unit: critical item failed → passed=false even if percentage ≥ passing_score`.
- `Unit: weighted score across sections matches hand calc`.
- `Integration: complete audit with non-compliant responses → corrective_actions created with mapped severity`.
- `Integration: offline submit with same client UUID twice → single audit (idempotent)`.
- `E2E (Playwright, mobile): field rep completes audit, captures photo, submits → score shown`.

#### 8.3 — Field visits & regulatory tracking
**What**: `field_visits` scheduling and `regulatory_requirements` calendar.

**Design**:
- Models per Suggestion 1 lines 295-312 and 519-533. Field visits link audits; recurring regulatory requirements (`recurrence`) generate upcoming compliance tasks per applicable brand/jurisdiction (supports GDPR/PDPA/food-safety multi-country tracking from README).
- Endpoints: `POST/GET /v1/field-visits`, `GET /v1/compliance/calendar?from&to`.

**Testing**:
- `Integration: annual requirement → next due date computed; appears in calendar window`.

---

## Phase 9: Training & LMS (SCORM-compliant)

### Purpose
Provide the integrated LMS with SCORM runtime, enrollments, completion dashboards, and certifications — a key differentiator versus audit-only/training-only incumbents. Implements Suggestion 1 §4 (lines 543-667). FERPA-aware handling of training records (standards.md).

### Tasks

#### 9.1 — Programs, courses, modules + SCORM package handling
**What**: `training_programs`, `courses`, `course_modules` with SCORM package ingestion.

**Design**:
- Models per Suggestion 1 lines 543-588. SCORM `.zip` uploaded to object storage, `imsmanifest.xml` parsed to register SCOs and detect `scorm_version` (`1.2`, `2004_3rd`, `2004_4th`). Course types: `scorm|video|document|quiz|live_session|on_the_job`.
- Endpoints: `POST/GET /v1/training/programs`, `/courses`, `POST /v1/courses/{id}/scorm-package`.

**Testing**:
- `Unit: parse sample imsmanifest.xml → SCOs + version detected`.
- `Integration: upload SCORM 1.2 zip → modules created with scorm_sco_id`.

#### 9.2 — Enrollments + SCORM runtime (CMI data model)
**What**: `enrollments`, `scorm_tracking`, `scorm_interactions` and the SCORM API endpoint.

**Design**:
- Models per Suggestion 1 lines 590-644. Implement the SCORM RTE: a JS adapter (`API`/`API_1484_11`) in the frontend calls `POST /v1/scorm/{enrollment}/commit` persisting CMI fields (`cmi.completion_status`, `cmi.success_status`, `cmi.score.*`, `cmi.location`, `cmi.suspend_data`, `cmi.progress_measure`) and interactions. Completion/score roll up to `enrollments` (status/score) and trigger certification issuance.
- Endpoints: `POST /v1/enrollments`, `GET/POST /v1/scorm/{enrollment}/cmi`, `POST /v1/scorm/{enrollment}/commit`.

**Testing**:
- `Integration: commit completion_status=completed,score_raw=90 → enrollment completed, score=90`.
- `Integration: resume → cmi.location and suspend_data returned verbatim`.
- `Unit: max_attempts exceeded → enrollment failed, new attempt blocked`.

#### 9.3 — Certifications & completion dashboards
**What**: `certifications`, `user_certifications`, location-based completion reporting.

**Design**:
- Models per Suggestion 1 lines 646-667. Auto-issue cert on required-program completion with `validity_months` expiry; a worker expires lapsed certs. Dashboard endpoint aggregates completion by location/role (research.md "location-based reporting").
- Endpoints: `GET /v1/training/completion?location_id=&program_id=`, `GET /v1/users/{id}/certifications`.

**Testing**:
- `Integration: complete required program → user_certification issued with expiry = now + validity_months`.
- `Integration: completion dashboard → percentages per location correct`.

---

## Phase 10: AI-Native Layer (risk, anomalies, lead scoring, NLP, assistant)

### Purpose
Deliver the AI-native advantage that distinguishes this platform: early at-risk franchisee detection, network anomaly detection, lead scoring, NLP feedback insights, and an MCP-exposed assistant. Implements Suggestion 1 anomaly_detections (lines 929-947) and populates `franchisees.risk_score`, `franchise_prospects.lead_score`. Uses MCP per standards.md.

### Tasks

#### 10.1 — LLM client + MCP server
**What**: Provider-agnostic LLM wrapper and an MCP server exposing read-only insight tools.

**Design**:
- `ai/llm_client.py`: `LLMClient` protocol with Anthropic + OpenAI backends, prompt-caching, token budgeting, retries. System prompt template enforces "cite the data rows used; never fabricate metrics."
- `ai/mcp_server.py`: MCP tools `get_franchisee_scorecard`, `list_anomalies`, `query_kpis`, `summarize_feedback` — all tenant-scoped and RBAC-checked so AI cannot bypass RLS.

**Testing**:
- `Integration (mocked LLM): MCP tool call respects tenant scope (cannot read other franchisor's data)`.
- `Unit: provider failover Anthropic→OpenAI on error`.

#### 10.2 — Franchisee risk scoring & lead scoring
**What**: scikit-learn models populating `risk_score` and `lead_score`.

**Design**:
- `ai/risk_model.py`: gradient-boosted classifier over features (KPI trends, audit scores, corrective-action counts, payment delinquency, training completion). Outputs `risk_score` 0-100 + `risk_factors[]`. Nightly worker scores all open franchisees.
- `ai/lead_scoring.py`: scores prospects (investment_capacity, source, engagement activity count) → `lead_score`.
- Models versioned; explainability via SHAP-style top factors stored in `risk_factors`.

**Testing**:
- `Unit: feature extraction produces stable vector for a franchisee fixture`.
- `Integration: nightly scoring updates risk_score + risk_factors; high-delinquency fixture scores high risk`.

#### 10.3 — Anomaly detection & recommendations
**What**: Populate `anomaly_detections` from KPI snapshots.

**Design**:
- `ai/anomaly.py`: per-KPI seasonal/statistical detection (rolling z-score / STL residual) across the network; emits `anomaly_type` (`spike|drop|trend_reversal|outlier|pattern_break`), severity, expected vs actual, deviation. LLM generates `ai_recommendation` grounded in the detected rows. Triggered by the Phase 5 `compute_kpis` task.
- Endpoint: `GET /v1/anomalies` (filter severity/location), `POST /v1/anomalies/{id}/acknowledge`.

**Testing**:
- `Unit: injected 40% revenue drop → "drop" anomaly, severity critical, deviation_pct ≈ -40`.
- `Integration (mocked LLM): anomaly gets ai_recommendation text; acknowledge sets acknowledged_by/at`.

#### 10.4 — NLP feedback insights & assistant chatbot
**What**: Summarize franchisee feedback at scale; in-app assistant.

**Design**:
- `ai/feedback_nlp.py`: clusters and summarizes free-text feedback (tickets, NPS comments, audit notes) into themes with sentiment; surfaces on the dashboard.
- Assistant endpoint `POST /v1/assistant/chat` backed by the MCP tools — answers "how is location X trending?", "what's overdue for franchisee Y?" with grounded, RBAC-scoped data.

**Testing**:
- `Integration (mocked LLM): feedback batch → themes with counts + sentiment`.
- `Integration: assistant query for franchisee Y as a franchisee-owner of Z → refuses cross-tenant/out-of-scope data`.

---

## Phase 11: Privacy/Compliance Controls, Hardening & Deployment

### Purpose
Make the platform production-deployable and audit-ready: GDPR/CCPA data-subject workflows, consent, security hardening, observability, and packaged deployment (Docker + Helm). Satisfies standards.md privacy (GDPR/CCPA), SOC 2 / ISO 27001 control expectations, and the self-hosted/cloud/hybrid deployment goal.

### Tasks

#### 11.1 — Data-subject rights & consent
**What**: GDPR/CCPA export, erasure, and consent tracking.

**Design**:
- Endpoints: `POST /v1/privacy/export/{user_id}` (machine-readable JSON bundle of the subject's data), `POST /v1/privacy/erase/{user_id}` (soft-delete + crypto-shred PII, preserving financial records required by ASC 606 with pseudonymization). `consents` table records purpose/timestamp/version.
- Data residency note: per-region Postgres clusters with app-level routing by franchisor jurisdiction (Suggestion 1 lines 1056-1059).

**Testing**:
- `Integration: export returns all subject rows across modules`.
- `Integration: erase pseudonymizes PII but retains invoice totals (financial-record retention)`.

#### 11.2 — Security hardening & observability
**What**: Rate limiting, secrets management, structured logging, metrics, tracing.

**Design**:
- Rate limiting (Redis token bucket) on auth and write endpoints; security headers; dependency scanning in CI. OpenTelemetry traces + Prometheus metrics; the Phase-1 `audit_log` provides the SOC 2 audit trail. RFC 7807 errors everywhere.

**Testing**:
- `Integration: exceed login rate limit → 429`.
- `Integration: /metrics exposes request + task counters`.

#### 11.3 — Deployment packaging
**What**: Production Compose + Helm chart + CI/CD.

**Design**:
- `docker-compose.prod.yml` (TLS via reverse proxy, separate worker scaling); `deploy/helm/` chart (api, worker, web, postgres/postgis, redis, minio, ingress with TLS). CI runs lint/type/test (testcontainers Postgres), builds images, runs full Alembic migrate-from-scratch, publishes OpenAPI 3.1 spec artifact.

**Testing**:
- `CI: helm template renders valid manifests; docker build for api/worker/web succeeds`.
- `CI: alembic upgrade head from empty DB on every run`.
- `E2E: full stack via compose → smoke test login→dashboard→audit→invoice`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (tenancy, auth, RBAC, RLS, audit log)   ── required by everything
    │
Phase 2: Franchisee Lifecycle (CRM, brands, locations)      ── requires 1
    │
    ├── Phase 3: Documents & Assets        ── requires 2 ─┐
    ├── Phase 4: Communication Hub         ── requires 2  │ (3,4,5 can parallelize)
    └── Phase 5: Revenue & Dashboard       ── requires 2 ─┘
            │
            ├── Phase 6: Royalties/Invoicing/Payments  ── requires 2,5
            └── Phase 8: Compliance Audits             ── requires 3,7
    Phase 7: Notifications/Helpdesk/Workflow ── requires 4 (enables 8 escalations)
            │
    Phase 9: Training & LMS                  ── requires 1,2 (parallel with 6/8)
            │
    Phase 10: AI Layer                       ── requires 5 (KPIs), 8 (audits), 2 (CRM)
            │
    Phase 11: Privacy/Hardening/Deployment   ── requires all; cross-cutting, finalized last
```

**Parallelism opportunities**
- After Phase 2: Phases **3, 4, 5** can be built concurrently by separate workstreams.
- **Phase 7** can proceed alongside 3/4/5 (depends only on 4 for announcement delivery).
- **Phase 9 (LMS)** is largely independent and can run parallel to Phases 6/8.
- **Phase 8** requires Phase 3 (evidence storage/docs) and Phase 7 (corrective-action escalation).
- **Phase 10 (AI)** must follow 5, 8, and 2 since it consumes KPIs, audits, and CRM data.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase implemented.
2. All unit and integration tests pass (integration tests run against real PostgreSQL+PostGIS via testcontainers; RLS isolation explicitly tested).
3. `ruff`, `black --check`, and `mypy` pass for backend; `eslint`, `prettier --check`, `tsc --noEmit` pass for frontend.
4. New/changed endpoints appear in the auto-generated **OpenAPI 3.1** spec, and the frontend client is regenerated.
5. Alembic migration(s) created; `alembic upgrade head` from an empty database and `downgrade base` both succeed in CI.
6. RLS policies present and enabled on every new tenant-scoped table.
7. `docker compose up` brings the stack to a healthy state; the phase's feature works end-to-end (Playwright e2e where a UI exists).
8. New config options documented in `.env.example` and `docs/`.
9. Audit-log coverage for all create/update/delete on new entities (SOC 2 / ISO 27001).
10. Money handled as `Decimal`/`NUMERIC` only; no floating-point currency anywhere in new code.
```

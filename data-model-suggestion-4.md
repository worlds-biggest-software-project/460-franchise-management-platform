# Data Model Suggestion 4: Multi-Model Polyglot Architecture (PostgreSQL + Neo4j + TimescaleDB)

## Overview

This model uses a polyglot persistence strategy that matches each subdomain of the franchise management platform to the storage engine best suited for its access patterns. Rather than forcing every data shape into a single paradigm, it assigns:

- **PostgreSQL** for transactional core data (franchisees, agreements, invoices, payments)
- **Neo4j** (graph database) for relationship-rich data (franchise network topology, territory relationships, organizational hierarchies, influence mapping)
- **TimescaleDB** (time-series on PostgreSQL) for temporal performance data (KPIs, revenue time-series, audit score trends, anomaly detection)

The franchise management domain has three distinct data access patterns that align poorly with a single database model. First, franchise networks are inherently graph-shaped: franchisors own brands, brands have territories, territories contain locations, locations are operated by franchisees, franchisees employ staff, staff complete training, and these relationships create rich traversal patterns for impact analysis, territory optimization, and organizational visibility. Second, performance benchmarking and anomaly detection are fundamentally time-series problems: tracking thousands of KPI values across hundreds of locations over years, computing moving averages, detecting trend reversals, and comparing period-over-period changes. Third, the transactional core (agreements, invoices, payments) demands ACID guarantees and relational integrity.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Transactional Core | PostgreSQL 16+ with PostGIS | ACID for financials, RLS for multi-tenancy, PostGIS for geo |
| Graph Layer | Neo4j 5.x (or Memgraph) | Network traversal, territory analysis, organizational hierarchy |
| Time-Series Layer | TimescaleDB 2.x (PostgreSQL extension) | KPI tracking, anomaly detection, performance trending |
| Event Bus | Apache Kafka | Sync data across stores, event-driven projections |
| Search | OpenSearch / Elasticsearch | Full-text search across documents, training content |
| Cache | Redis | Session management, dashboard caching, real-time alerts |
| Object Storage | S3/MinIO | SCORM packages, audit photos/videos, document files |
| API Gateway | Kong or AWS API Gateway | Route queries to appropriate backend store |

---

## Architecture Diagram

```
                           ┌───────────────────────┐
                           │    API Gateway         │
                           │  (Route by domain)     │
                           └────┬──────┬──────┬─────┘
                                │      │      │
                    ┌───────────┘      │      └───────────┐
                    ▼                  ▼                  ▼
          ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
          │ Transaction  │   │   Graph       │   │  Time-Series │
          │  Service     │   │   Service     │   │  Service     │
          │ (Core CRUD)  │   │ (Network)     │   │ (Analytics)  │
          └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
                 │                  │                  │
                 ▼                  ▼                  ▼
          ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
          │ PostgreSQL   │   │   Neo4j      │   │ TimescaleDB  │
          │ + PostGIS    │   │  (Graph DB)  │   │ (Time-Series)│
          └──────────────┘   └──────────────┘   └──────────────┘
                 │                  │                  │
                 └────────────┬─────┘──────────────────┘
                              ▼
                     ┌──────────────┐
                     │ Apache Kafka │
                     │ (Event Bus)  │
                     └──────────────┘
```

---

## 1. PostgreSQL Core Schema (Transactional)

The PostgreSQL schema handles identity, lifecycle, financial transactions, and serves as the system of record for compliance-critical data.

```sql
-- ============================================================
-- POSTGRESQL: TRANSACTIONAL CORE
-- ============================================================

-- Organization
CREATE TABLE franchisors (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    legal_name          VARCHAR(500),
    industry_sector     VARCHAR(100) NOT NULL,
    headquarters_country VARCHAR(3) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    subscription_tier   VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    email               VARCHAR(320) NOT NULL,
    password_hash       VARCHAR(255),
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    role                VARCHAR(50) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    profile             JSONB NOT NULL DEFAULT '{}',
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (franchisor_id, email)
);

CREATE TABLE franchise_brands (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    name                VARCHAR(255) NOT NULL,
    config              JSONB NOT NULL DEFAULT '{}',
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE franchisees (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    brand_id            UUID NOT NULL REFERENCES franchise_brands(id),
    entity_name         VARCHAR(500) NOT NULL,
    primary_owner_id    UUID REFERENCES users(id),
    status              VARCHAR(30) NOT NULL DEFAULT 'onboarding',
    risk_score          NUMERIC(5,2),
    entity_details      JSONB NOT NULL DEFAULT '{}',
    opening_date        DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_franchisees_status ON franchisees (franchisor_id, status);

CREATE TABLE locations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    name                VARCHAR(255) NOT NULL,
    store_number        VARCHAR(50),
    status              VARCHAR(20) NOT NULL DEFAULT 'planned',
    address_line1       VARCHAR(255) NOT NULL,
    city                VARCHAR(100) NOT NULL,
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20),
    country             VARCHAR(3) NOT NULL,
    geom                GEOMETRY(Point, 4326),
    attributes          JSONB NOT NULL DEFAULT '{}',
    opened_date         DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_locations_geom ON locations USING GIST (geom);

CREATE TABLE territories (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    brand_id            UUID NOT NULL REFERENCES franchise_brands(id),
    name                VARCHAR(255) NOT NULL,
    territory_type      VARCHAR(50) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'available',
    assigned_franchisee_id UUID REFERENCES franchisees(id),
    boundary            GEOMETRY(MultiPolygon, 4326),
    demographics        JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_territories_boundary ON territories USING GIST (boundary);

-- ── Financial Tables (ACID-critical) ─────────────────────────

CREATE TABLE franchise_agreements (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    brand_id            UUID NOT NULL REFERENCES franchise_brands(id),
    agreement_number    VARCHAR(100) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    start_date          DATE NOT NULL,
    end_date            DATE NOT NULL,
    financial_terms     JSONB NOT NULL,
    document_url        TEXT,
    signed_at           TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE revenue_reports (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    location_id         UUID NOT NULL REFERENCES locations(id),
    reporting_period    DATE NOT NULL,
    period_type         VARCHAR(20) NOT NULL DEFAULT 'monthly',
    gross_revenue       NUMERIC(14,2) NOT NULL,
    net_revenue         NUMERIC(14,2),
    revenue_breakdown   JSONB NOT NULL DEFAULT '{}',
    source              VARCHAR(30) NOT NULL DEFAULT 'manual',
    status              VARCHAR(20) NOT NULL DEFAULT 'submitted',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (location_id, reporting_period, period_type)
);

CREATE TABLE royalty_calculations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    agreement_id        UUID NOT NULL REFERENCES franchise_agreements(id),
    revenue_report_id   UUID NOT NULL REFERENCES revenue_reports(id),
    billing_period      DATE NOT NULL,
    gross_revenue       NUMERIC(14,2) NOT NULL,
    applicable_revenue  NUMERIC(14,2) NOT NULL,
    royalty_amount      NUMERIC(12,2) NOT NULL,
    ad_fund_amount      NUMERIC(12,2),
    technology_fee      NUMERIC(12,2),
    total_due           NUMERIC(12,2) NOT NULL,
    calculation_details JSONB NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE invoices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    invoice_number      VARCHAR(50) NOT NULL UNIQUE,
    billing_period      DATE NOT NULL,
    issue_date          DATE NOT NULL,
    due_date            DATE NOT NULL,
    total_amount        NUMERIC(12,2) NOT NULL,
    balance_due         NUMERIC(12,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD',
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    line_items          JSONB NOT NULL DEFAULT '[]',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE payments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    invoice_id          UUID REFERENCES invoices(id),
    amount              NUMERIC(12,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD',
    payment_method      VARCHAR(30) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    payment_details     JSONB NOT NULL DEFAULT '{}',
    processed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ── Compliance (Audits) ─────────────────────────────────────

CREATE TABLE audit_templates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    name                VARCHAR(255) NOT NULL,
    audit_type          VARCHAR(50) NOT NULL,
    version             INTEGER NOT NULL DEFAULT 1,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    template_definition JSONB NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE audits (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    template_id         UUID NOT NULL REFERENCES audit_templates(id),
    location_id         UUID NOT NULL REFERENCES locations(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    auditor_id          UUID NOT NULL REFERENCES users(id),
    status              VARCHAR(30) NOT NULL DEFAULT 'scheduled',
    scheduled_date      DATE NOT NULL,
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    total_score         NUMERIC(7,2),
    score_percentage    NUMERIC(5,2),
    passed              BOOLEAN,
    responses           JSONB NOT NULL DEFAULT '[]',
    section_scores      JSONB NOT NULL DEFAULT '[]',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE corrective_actions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    audit_id            UUID REFERENCES audits(id),
    location_id         UUID NOT NULL REFERENCES locations(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    title               VARCHAR(500) NOT NULL,
    severity            VARCHAR(20) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'open',
    assigned_to         UUID REFERENCES users(id),
    due_date            DATE NOT NULL,
    details             JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ── Training & LMS ──────────────────────────────────────────

CREATE TABLE training_programs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    brand_id            UUID REFERENCES franchise_brands(id),
    name                VARCHAR(255) NOT NULL,
    program_type        VARCHAR(50) NOT NULL,
    config              JSONB NOT NULL DEFAULT '{}',
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE courses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    training_program_id UUID REFERENCES training_programs(id),
    title               VARCHAR(500) NOT NULL,
    course_type         VARCHAR(30) NOT NULL,
    content             JSONB NOT NULL DEFAULT '{}',
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE enrollments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    course_id           UUID NOT NULL REFERENCES courses(id),
    franchisee_id       UUID REFERENCES franchisees(id),
    location_id         UUID REFERENCES locations(id),
    status              VARCHAR(30) NOT NULL DEFAULT 'enrolled',
    enrolled_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    due_date            DATE,
    completed_at        TIMESTAMPTZ,
    score               NUMERIC(5,2),
    tracking_data       JSONB NOT NULL DEFAULT '{}',
    UNIQUE (user_id, course_id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ── Communication ───────────────────────────────────────────

CREATE TABLE support_tickets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    ticket_number       VARCHAR(50) NOT NULL,
    created_by          UUID NOT NULL REFERENCES users(id),
    franchisee_id       UUID REFERENCES franchisees(id),
    category            VARCHAR(100) NOT NULL,
    subject             VARCHAR(500) NOT NULL,
    priority            VARCHAR(20) NOT NULL DEFAULT 'medium',
    status              VARCHAR(30) NOT NULL DEFAULT 'open',
    assigned_to         UUID REFERENCES users(id),
    thread              JSONB NOT NULL DEFAULT '[]',
    metrics             JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (franchisor_id, ticket_number)
);

-- Platform audit log
CREATE TABLE audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    franchisor_id       UUID NOT NULL,
    user_id             UUID,
    action              VARCHAR(50) NOT NULL,
    entity_type         VARCHAR(100) NOT NULL,
    entity_id           UUID,
    change_data         JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 2. Neo4j Graph Schema (Network Intelligence)

The graph database models the franchise network as a rich, traversable graph. This is where the platform's AI capabilities (territory optimization, influence mapping, risk propagation, organizational analysis) draw their power.

### Node Types

```cypher
// ── Core Network Nodes ──────────────────────────────────────

// Franchisor (root of a network)
CREATE CONSTRAINT franchisor_id IF NOT EXISTS
    FOR (f:Franchisor) REQUIRE f.id IS UNIQUE;

// Brand within a franchisor's portfolio
CREATE CONSTRAINT brand_id IF NOT EXISTS
    FOR (b:Brand) REQUIRE b.id IS UNIQUE;

// Franchisee entity
CREATE CONSTRAINT franchisee_id IF NOT EXISTS
    FOR (fe:Franchisee) REQUIRE fe.id IS UNIQUE;

// Physical location
CREATE CONSTRAINT location_id IF NOT EXISTS
    FOR (l:Location) REQUIRE l.id IS UNIQUE;

// Territory
CREATE CONSTRAINT territory_id IF NOT EXISTS
    FOR (t:Territory) REQUIRE t.id IS UNIQUE;

// Person (user in any role)
CREATE CONSTRAINT person_id IF NOT EXISTS
    FOR (p:Person) REQUIRE p.id IS UNIQUE;

// Training Program
CREATE CONSTRAINT program_id IF NOT EXISTS
    FOR (tp:TrainingProgram) REQUIRE tp.id IS UNIQUE;

// Course
CREATE CONSTRAINT course_id IF NOT EXISTS
    FOR (c:Course) REQUIRE c.id IS UNIQUE;

// Certification
CREATE CONSTRAINT cert_id IF NOT EXISTS
    FOR (cert:Certification) REQUIRE cert.id IS UNIQUE;

// Audit
CREATE CONSTRAINT audit_id IF NOT EXISTS
    FOR (a:Audit) REQUIRE a.id IS UNIQUE;

// Region (geographic grouping)
CREATE CONSTRAINT region_name IF NOT EXISTS
    FOR (r:Region) REQUIRE (r.franchisor_id, r.name) IS UNIQUE;

// Market (metro area or market area)
CREATE CONSTRAINT market_id IF NOT EXISTS
    FOR (m:Market) REQUIRE m.id IS UNIQUE;
```

### Relationship Types

```cypher
// ── Organizational Hierarchy ────────────────────────────────

// Franchisor -[OWNS_BRAND]-> Brand
CREATE (fr:Franchisor {id: $franchisorId, name: "FranchiseCo"})
CREATE (b:Brand {id: $brandId, name: "BurgerWorld"})
CREATE (fr)-[:OWNS_BRAND {since: date("2010-01-01")}]->(b)

// Brand -[HAS_TERRITORY]-> Territory
CREATE (b)-[:HAS_TERRITORY]->(t:Territory {id: $territoryId, name: "Chicago North", type: "exclusive"})

// Territory -[ASSIGNED_TO]-> Franchisee
CREATE (t)-[:ASSIGNED_TO {since: date("2020-06-15"), agreement_id: $agreementId}]->(fe:Franchisee)

// Franchisee -[OPERATES]-> Location
CREATE (fe)-[:OPERATES {since: date("2020-09-01")}]->(l:Location {id: $locationId, name: "BurgerWorld #142"})

// Location -[LOCATED_IN]-> Territory
CREATE (l)-[:LOCATED_IN]->(t)

// Location -[IN_MARKET]-> Market
CREATE (l)-[:IN_MARKET]->(m:Market {id: $marketId, name: "Chicago Metro"})

// Market -[IN_REGION]-> Region
CREATE (m)-[:IN_REGION]->(r:Region {name: "Midwest"})

// ── People Relationships ────────────────────────────────────

// Person -[OWNS]-> Franchisee (ownership with percentage)
CREATE (p:Person {id: $ownerId})-[:OWNS {
    ownership_pct: 60,
    is_managing_member: true,
    since: date("2020-06-15")
}]->(fe)

// Person -[WORKS_AT]-> Location (employment)
CREATE (staff:Person {id: $staffId})-[:WORKS_AT {
    role: "shift_manager",
    since: date("2022-01-15"),
    is_active: true
}]->(l)

// Person -[MANAGES]-> Person (management hierarchy)
CREATE (manager:Person)-[:MANAGES {since: date("2022-01-15")}]->(staff)

// Person -[FIELD_REP_FOR]-> Region (field representative coverage)
CREATE (rep:Person {id: $repId})-[:FIELD_REP_FOR {
    since: date("2021-03-01")
}]->(r)

// ── Training & Certification Relationships ──────────────────

// Person -[ENROLLED_IN]-> Course
CREATE (p)-[:ENROLLED_IN {
    enrolled_at: datetime(),
    status: "in_progress",
    score: 85.0,
    completed_at: null
}]->(c:Course {id: $courseId})

// Person -[HOLDS]-> Certification
CREATE (p)-[:HOLDS {
    issued_at: datetime("2025-01-15T00:00:00Z"),
    expires_at: datetime("2027-01-15T00:00:00Z"),
    certificate_number: "CERT-2025-0042"
}]->(cert:Certification {id: $certId, name: "Food Safety Manager"})

// Course -[REQUIRES]-> Course (prerequisites)
CREATE (advanced:Course)-[:REQUIRES]->(basic:Course)

// TrainingProgram -[INCLUDES]-> Course
CREATE (tp:TrainingProgram)-[:INCLUDES {sort_order: 1}]->(c)

// Certification -[REQUIRES_PROGRAM]-> TrainingProgram
CREATE (cert)-[:REQUIRES_PROGRAM]->(tp)

// ── Compliance Relationships ────────────────────────────────

// Audit -[AUDITED]-> Location
CREATE (a:Audit {id: $auditId, score: 87.5, passed: true, date: date("2025-03-15")})
CREATE (a)-[:AUDITED]->(l)

// Audit -[CONDUCTED_BY]-> Person
CREATE (a)-[:CONDUCTED_BY]->(auditor:Person)

// Audit -[FOUND_ISSUE]-> CorrectiveAction
CREATE (a)-[:FOUND_ISSUE]->(ca:CorrectiveAction {id: $caId, severity: "major", status: "open"})

// ── Territory Adjacency ─────────────────────────────────────

// Territory -[ADJACENT_TO]-> Territory (computed from PostGIS, stored in graph)
CREATE (t1:Territory)-[:ADJACENT_TO {
    shared_border_length_km: 12.5,
    overlap_area_sqkm: 0  // should be 0 for properly defined territories
}]->(t2:Territory)

// Territory -[OVERLAPS]-> Territory (indicates a problem)
CREATE (t3:Territory)-[:OVERLAPS {
    overlap_area_sqkm: 2.3,
    detected_at: datetime(),
    resolution_status: "pending"
}]->(t4:Territory)

// ── Influence & Risk Propagation ────────────────────────────

// Franchisee -[REFERRED]-> Franchisee (referral network)
CREATE (fe1:Franchisee)-[:REFERRED {
    date: date("2023-05-20"),
    referral_bonus_paid: true
}]->(fe2:Franchisee)

// Location -[COMPETES_WITH]-> Location (cannibalization risk)
CREATE (l1:Location)-[:COMPETES_WITH {
    distance_km: 3.2,
    overlap_population: 15000,
    cannibalization_risk: "medium"
}]->(l2:Location)
```

### Key Graph Queries

```cypher
// ── Territory Optimization: Find best location for new franchise ──
// Find markets with high potential and no existing locations
MATCH (r:Region {name: "Midwest"})<-[:IN_REGION]-(m:Market)
WHERE NOT EXISTS {
    MATCH (m)<-[:IN_MARKET]-(l:Location {status: "open"})
}
RETURN m.name AS market, m.population AS population, m.median_income AS income
ORDER BY m.population DESC LIMIT 10;

// ── Risk Propagation: If franchisee X fails, who is affected? ──
// Find all people, locations, and downstream relationships
MATCH path = (fe:Franchisee {id: $franchiseeId})-[*1..3]-(connected)
RETURN path;

// ── Training Compliance Gap Analysis ──
// Find staff at a location who lack required certifications
MATCH (l:Location {id: $locationId})<-[:WORKS_AT {is_active: true}]-(p:Person)
MATCH (cert:Certification {required: true})
WHERE NOT EXISTS {
    MATCH (p)-[:HOLDS {status: "active"}]->(cert)
    WHERE (p)-[:HOLDS]->(cert) AND
          (p)-[h:HOLDS]->(cert) AND h.expires_at > datetime()
}
RETURN p.name AS staff_member, cert.name AS missing_certification;

// ── Franchise Network Expansion Analysis ──
// Find territories adjacent to high-performing locations
MATCH (l:Location)-[:LOCATED_IN]->(t1:Territory)
WHERE l.status = 'open'
MATCH (t1)-[:ADJACENT_TO]->(t2:Territory {status: "available"})
RETURN t2.name AS available_territory,
       t2.population AS population,
       l.name AS adjacent_high_performer,
       l.latest_revenue AS revenue
ORDER BY t2.population DESC;

// ── Multi-Unit Operator Portfolio View ──
// Show all entities owned by a person across the network
MATCH (p:Person {id: $personId})-[:OWNS]->(fe:Franchisee)-[:OPERATES]->(l:Location)
MATCH (fe)-[:ASSIGNED_TO]-(t:Territory)
RETURN fe.entity_name, collect(l.name) AS locations,
       collect(t.name) AS territories,
       sum(l.latest_revenue) AS total_revenue;

// ── Audit Score Trend by Field Rep's Region ──
MATCH (rep:Person {id: $repId})-[:FIELD_REP_FOR]->(r:Region)
MATCH (r)<-[:IN_REGION]-(m:Market)<-[:IN_MARKET]-(l:Location)
MATCH (a:Audit)-[:AUDITED]->(l)
WHERE a.date >= date("2024-01-01")
RETURN l.name, a.date, a.score
ORDER BY l.name, a.date;

// ── Cannibalization Impact Assessment ──
// Before approving a new location, check competitive impact
MATCH (proposed:Location {id: $proposedId})-[:COMPETES_WITH]->(existing:Location)
MATCH (existing)<-[:OPERATES]-(fe:Franchisee)
RETURN existing.name, fe.entity_name,
       proposed.distance_km AS distance,
       proposed.overlap_population AS shared_population,
       proposed.cannibalization_risk AS risk
ORDER BY proposed.distance_km;
```

---

## 3. TimescaleDB Schema (Time-Series Analytics)

TimescaleDB extends PostgreSQL with automatic partitioning (hypertables), continuous aggregates, and compression for time-series data. This layer handles all temporal performance data.

```sql
-- ============================================================
-- TIMESCALEDB: TIME-SERIES ANALYTICS
-- ============================================================

-- Install TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- ── Raw KPI Measurements ────────────────────────────────────
-- High-frequency data points: one row per KPI per location per period

CREATE TABLE kpi_measurements (
    time                TIMESTAMPTZ NOT NULL,
    franchisor_id       UUID NOT NULL,
    location_id         UUID NOT NULL,
    franchisee_id       UUID NOT NULL,
    kpi_code            VARCHAR(100) NOT NULL,
    value               DOUBLE PRECISION NOT NULL,
    metadata            JSONB DEFAULT '{}'
);

-- Convert to hypertable (auto-partitioned by time)
SELECT create_hypertable('kpi_measurements', 'time',
    chunk_time_interval => INTERVAL '1 month');

-- Create indexes for common query patterns
CREATE INDEX idx_kpi_location ON kpi_measurements (franchisor_id, location_id, kpi_code, time DESC);
CREATE INDEX idx_kpi_franchisee ON kpi_measurements (franchisor_id, franchisee_id, kpi_code, time DESC);

-- ── Revenue Time-Series ─────────────────────────────────────
-- Daily/weekly/monthly revenue data points for trend analysis

CREATE TABLE revenue_timeseries (
    time                TIMESTAMPTZ NOT NULL,
    franchisor_id       UUID NOT NULL,
    location_id         UUID NOT NULL,
    franchisee_id       UUID NOT NULL,
    gross_revenue       DOUBLE PRECISION NOT NULL,
    net_revenue         DOUBLE PRECISION,
    transaction_count   INTEGER,
    average_ticket      DOUBLE PRECISION,
    labor_cost          DOUBLE PRECISION,
    cogs                DOUBLE PRECISION,
    labor_pct           DOUBLE PRECISION,
    cogs_pct            DOUBLE PRECISION,
    source              VARCHAR(30) DEFAULT 'manual'
);

SELECT create_hypertable('revenue_timeseries', 'time',
    chunk_time_interval => INTERVAL '3 months');

CREATE INDEX idx_revenue_ts_location ON revenue_timeseries (franchisor_id, location_id, time DESC);

-- ── Audit Score Time-Series ─────────────────────────────────

CREATE TABLE audit_score_timeseries (
    time                TIMESTAMPTZ NOT NULL,
    franchisor_id       UUID NOT NULL,
    location_id         UUID NOT NULL,
    franchisee_id       UUID NOT NULL,
    audit_id            UUID NOT NULL,
    audit_type          VARCHAR(50) NOT NULL,
    score_percentage    DOUBLE PRECISION NOT NULL,
    passed              BOOLEAN NOT NULL,
    critical_failures   INTEGER DEFAULT 0,
    non_compliant_items INTEGER DEFAULT 0,
    section_scores      JSONB DEFAULT '[]'
);

SELECT create_hypertable('audit_score_timeseries', 'time',
    chunk_time_interval => INTERVAL '6 months');

-- ── Training Activity Time-Series ───────────────────────────

CREATE TABLE training_activity_timeseries (
    time                TIMESTAMPTZ NOT NULL,
    franchisor_id       UUID NOT NULL,
    location_id         UUID,
    franchisee_id       UUID NOT NULL,
    user_id             UUID NOT NULL,
    course_id           UUID NOT NULL,
    event_type          VARCHAR(30) NOT NULL,  -- 'started', 'progress', 'completed', 'failed'
    score               DOUBLE PRECISION,
    time_spent_seconds  INTEGER,
    progress_pct        DOUBLE PRECISION
);

SELECT create_hypertable('training_activity_timeseries', 'time',
    chunk_time_interval => INTERVAL '3 months');

-- ── Anomaly Detection Results ───────────────────────────────

CREATE TABLE anomaly_detections (
    time                TIMESTAMPTZ NOT NULL,
    franchisor_id       UUID NOT NULL,
    location_id         UUID NOT NULL,
    franchisee_id       UUID NOT NULL,
    kpi_code            VARCHAR(100) NOT NULL,
    anomaly_type        VARCHAR(50) NOT NULL,
    severity            VARCHAR(20) NOT NULL,
    expected_value      DOUBLE PRECISION,
    actual_value        DOUBLE PRECISION,
    deviation_pct       DOUBLE PRECISION,
    z_score             DOUBLE PRECISION,
    model_version       VARCHAR(50),
    description         TEXT,
    ai_recommendation   TEXT,
    acknowledged        BOOLEAN DEFAULT FALSE
);

SELECT create_hypertable('anomaly_detections', 'time',
    chunk_time_interval => INTERVAL '3 months');

-- ============================================================
-- CONTINUOUS AGGREGATES (Materialized Views with auto-refresh)
-- ============================================================

-- ── Monthly Revenue Summary per Location ────────────────────
CREATE MATERIALIZED VIEW monthly_revenue_summary
    WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 month', time) AS month,
    franchisor_id,
    location_id,
    franchisee_id,
    SUM(gross_revenue) AS total_revenue,
    SUM(net_revenue) AS total_net_revenue,
    SUM(transaction_count) AS total_transactions,
    AVG(average_ticket) AS avg_ticket,
    AVG(labor_pct) AS avg_labor_pct,
    AVG(cogs_pct) AS avg_cogs_pct,
    COUNT(*) AS data_points
FROM revenue_timeseries
GROUP BY month, franchisor_id, location_id, franchisee_id;

-- Auto-refresh policy: update daily, looking back 3 days for late data
SELECT add_continuous_aggregate_policy('monthly_revenue_summary',
    start_offset => INTERVAL '3 months',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');

-- ── Weekly KPI Averages ─────────────────────────────────────
CREATE MATERIALIZED VIEW weekly_kpi_averages
    WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 week', time) AS week,
    franchisor_id,
    location_id,
    kpi_code,
    AVG(value) AS avg_value,
    MIN(value) AS min_value,
    MAX(value) AS max_value,
    COUNT(*) AS sample_count
FROM kpi_measurements
GROUP BY week, franchisor_id, location_id, kpi_code;

SELECT add_continuous_aggregate_policy('weekly_kpi_averages',
    start_offset => INTERVAL '1 month',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');

-- ── Quarterly Network Benchmarks ────────────────────────────
CREATE MATERIALIZED VIEW quarterly_network_benchmarks
    WITH (timescaledb.continuous) AS
SELECT
    time_bucket('3 months', time) AS quarter,
    franchisor_id,
    kpi_code,
    AVG(value) AS network_avg,
    percentile_cont(0.5) WITHIN GROUP (ORDER BY value) AS network_median,
    percentile_cont(0.75) WITHIN GROUP (ORDER BY value) AS top_quartile,
    percentile_cont(0.25) WITHIN GROUP (ORDER BY value) AS bottom_quartile,
    MIN(value) AS network_min,
    MAX(value) AS network_max,
    stddev(value) AS std_deviation,
    COUNT(DISTINCT location_id) AS location_count
FROM kpi_measurements
GROUP BY quarter, franchisor_id, kpi_code;

SELECT add_continuous_aggregate_policy('quarterly_network_benchmarks',
    start_offset => INTERVAL '6 months',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');

-- ============================================================
-- TIME-SERIES ANALYSIS QUERIES
-- ============================================================

-- Same-store sales growth (year over year)
SELECT
    curr.month,
    curr.location_id,
    curr.total_revenue AS current_revenue,
    prev.total_revenue AS prior_year_revenue,
    ((curr.total_revenue - prev.total_revenue) / NULLIF(prev.total_revenue, 0) * 100)
        AS yoy_growth_pct
FROM monthly_revenue_summary curr
LEFT JOIN monthly_revenue_summary prev
    ON curr.location_id = prev.location_id
    AND curr.month = prev.month + INTERVAL '1 year'
WHERE curr.franchisor_id = $1
    AND curr.month >= '2025-01-01'
ORDER BY yoy_growth_pct DESC;

-- Rolling 12-week average for a KPI with trend detection
SELECT
    week,
    location_id,
    avg_value,
    AVG(avg_value) OVER (
        PARTITION BY location_id
        ORDER BY week
        ROWS BETWEEN 11 PRECEDING AND CURRENT ROW
    ) AS rolling_12w_avg,
    avg_value - LAG(avg_value, 4) OVER (
        PARTITION BY location_id ORDER BY week
    ) AS change_vs_4w_ago
FROM weekly_kpi_averages
WHERE franchisor_id = $1
    AND kpi_code = 'customer_satisfaction'
    AND week >= NOW() - INTERVAL '6 months'
ORDER BY location_id, week;

-- Anomaly detection: locations with KPIs more than 2 standard deviations from network mean
SELECT
    k.location_id,
    k.kpi_code,
    k.value AS actual_value,
    b.network_avg,
    b.std_deviation,
    (k.value - b.network_avg) / NULLIF(b.std_deviation, 0) AS z_score
FROM kpi_measurements k
JOIN quarterly_network_benchmarks b
    ON k.franchisor_id = b.franchisor_id
    AND k.kpi_code = b.kpi_code
    AND time_bucket('3 months', k.time) = b.quarter
WHERE k.franchisor_id = $1
    AND k.time >= NOW() - INTERVAL '1 month'
    AND ABS((k.value - b.network_avg) / NULLIF(b.std_deviation, 0)) > 2.0
ORDER BY z_score DESC;

-- ── Compression & Retention Policies ────────────────────────

-- Compress old data for storage efficiency (90%+ compression typical)
SELECT add_compression_policy('revenue_timeseries', INTERVAL '6 months');
SELECT add_compression_policy('kpi_measurements', INTERVAL '3 months');
SELECT add_compression_policy('audit_score_timeseries', INTERVAL '1 year');
SELECT add_compression_policy('training_activity_timeseries', INTERVAL '6 months');

-- Retention: drop raw data older than 5 years (continuous aggregates retained)
SELECT add_retention_policy('kpi_measurements', INTERVAL '5 years');
SELECT add_retention_policy('revenue_timeseries', INTERVAL '5 years');
SELECT add_retention_policy('training_activity_timeseries', INTERVAL '3 years');
```

---

## Data Synchronization Architecture

Data flows between the three stores through an event bus (Kafka). PostgreSQL is the system of record; Neo4j and TimescaleDB are derived views.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Write Path                                   │
│                                                                     │
│  Application  ──► PostgreSQL  ──► Kafka (CDC via Debezium)         │
│                   (System of Record)                                │
│                                                                     │
│  Kafka  ──► Neo4j Sync Service  ──► Neo4j                          │
│         ──► TimescaleDB Sync Service  ──► TimescaleDB              │
│         ──► Search Indexer  ──► OpenSearch                         │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                        Read Path                                    │
│                                                                     │
│  Dashboard queries  ──► PostgreSQL (entity data, financials)        │
│  Network analysis   ──► Neo4j (traversals, path finding)            │
│  KPI/Trends         ──► TimescaleDB (time-series aggregation)       │
│  Search             ──► OpenSearch (full-text)                      │
│                                                                     │
│  API Gateway routes each query to the appropriate store             │
└─────────────────────────────────────────────────────────────────────┘
```

### CDC (Change Data Capture) Configuration

```yaml
# Debezium connector configuration for PostgreSQL -> Kafka
connector.class: io.debezium.connector.postgresql.PostgresConnector
database.hostname: pg-primary
database.dbname: franchise_platform
table.include.list: >
  public.franchisors,
  public.franchisees,
  public.locations,
  public.territories,
  public.users,
  public.audits,
  public.corrective_actions,
  public.enrollments,
  public.revenue_reports
plugin.name: pgoutput
slot.name: franchise_cdc
publication.name: franchise_pub
transforms: route
transforms.route.type: io.debezium.transforms.ByLogicalTableRouter
transforms.route.topic.regex: (.*)
transforms.route.topic.replacement: franchise.cdc.$1
```

### Neo4j Sync Service (Example)

```typescript
// When a new franchisee is created in PostgreSQL, sync to Neo4j
async function handleFranchiseeCreated(event: CDCEvent) {
    const { id, franchisor_id, brand_id, entity_name, status, primary_owner_id } = event.after;

    await neo4jSession.run(`
        MERGE (fr:Franchisor {id: $franchisorId})
        MERGE (b:Brand {id: $brandId})
        MERGE (fe:Franchisee {id: $franchiseeId})
        SET fe.entity_name = $entityName,
            fe.status = $status,
            fe.updated_at = datetime()
        MERGE (fr)-[:OWNS_BRAND]->(b)
        MERGE (b)-[:HAS_FRANCHISEE]->(fe)
    `, { franchisorId: franchisor_id, brandId: brand_id, franchiseeId: id, entityName: entity_name, status });

    if (primary_owner_id) {
        await neo4jSession.run(`
            MATCH (fe:Franchisee {id: $franchiseeId})
            MERGE (p:Person {id: $ownerId})
            MERGE (p)-[:OWNS {is_primary: true}]->(fe)
        `, { franchiseeId: id, ownerId: primary_owner_id });
    }
}

// When a revenue report is submitted, sync to TimescaleDB
async function handleRevenueReportCreated(event: CDCEvent) {
    const report = event.after;
    const breakdown = JSON.parse(report.revenue_breakdown || '{}');

    await timescalePool.query(`
        INSERT INTO revenue_timeseries (time, franchisor_id, location_id, franchisee_id,
            gross_revenue, net_revenue, transaction_count, average_ticket,
            labor_cost, cogs, labor_pct, cogs_pct, source)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13)
    `, [
        report.reporting_period,
        report.franchisor_id,
        report.location_id,
        report.franchisee_id,
        report.gross_revenue,
        report.net_revenue,
        breakdown.transaction_count,
        breakdown.average_ticket,
        breakdown.labor_cost,
        breakdown.cost_of_goods,
        breakdown.labor_cost ? (breakdown.labor_cost / report.gross_revenue * 100) : null,
        breakdown.cost_of_goods ? (breakdown.cost_of_goods / report.gross_revenue * 100) : null,
        report.source
    ]);
}
```

---

## Pros and Cons

### Pros

1. **Each store is optimized for its access pattern.** Graph queries in Neo4j (network traversal, path finding, influence analysis) are orders of magnitude faster than SQL JOIN chains across 5+ tables. Time-series queries in TimescaleDB (rolling averages, continuous aggregates, compression) are purpose-built for the analytics use case. PostgreSQL handles the transactional core with ACID guarantees. Each technology does what it does best.

2. **AI/ML capabilities are dramatically enhanced.** Neo4j's graph algorithms (PageRank for franchisee influence, community detection for regional clustering, shortest path for supply chain optimization) power the platform's AI-native features without custom graph implementations. TimescaleDB's window functions and continuous aggregates enable real-time anomaly detection without ETL pipelines.

3. **Territory optimization is a graph problem.** Finding the optimal location for a new franchise requires traversing territory adjacencies, computing cannibalization risk with existing locations, analyzing referral networks, and considering field representative coverage. Neo4j answers these questions in milliseconds; PostgreSQL would need complex recursive CTEs.

4. **Time-series compression reduces storage costs.** TimescaleDB achieves 90%+ compression on historical KPI data. A franchise network with 1,000 locations tracking 20 KPIs daily generates 7.3M data points per year. Without compression, this requires ~500MB/year. With TimescaleDB compression, it shrinks to ~50MB/year. Continuous aggregates pre-compute monthly/quarterly summaries automatically.

5. **Continuous aggregates eliminate expensive dashboard queries.** Network-wide benchmarking queries (average revenue across 1,000 locations, quartile rankings, standard deviations) are pre-computed as continuous aggregates that update incrementally. Dashboard rendering hits pre-computed materialized views rather than scanning millions of raw data points.

6. **PostgreSQL remains the system of record.** All writes go through PostgreSQL with full ACID guarantees. Neo4j and TimescaleDB are derived views built from CDC events. If either derived store is corrupted or lost, it can be rebuilt from PostgreSQL. This eliminates the distributed transaction problem.

7. **Independent scaling per workload.** Analytics-heavy periods (end of month, quarterly reviews) scale TimescaleDB independently. Network expansion planning scales Neo4j independently. Transaction processing scales PostgreSQL independently. No single bottleneck limits the entire platform.

### Cons

1. **Operational complexity is the highest of all four approaches.** Three databases, a CDC pipeline (Debezium), a message broker (Kafka), sync services, and monitoring for all of them. The ops team needs expertise in PostgreSQL, Neo4j, TimescaleDB, and Kafka. Deployment, backup, failover, and version upgrades are multiplicatively complex.

2. **Data consistency across stores is eventual.** After a franchisee is created in PostgreSQL, there is a window (typically milliseconds to seconds, but potentially longer during backpressure) before the franchisee node appears in Neo4j or training activity metrics appear in TimescaleDB. UI patterns must accommodate this lag.

3. **Cross-store queries require application-level joins.** "Show me the top 10 at-risk franchisees (Neo4j graph analysis) with their financial summary (PostgreSQL) and KPI trends (TimescaleDB)" requires three separate queries stitched together in application code. No single query language spans all three stores.

4. **Development velocity is slower initially.** Every new feature must consider which store(s) it touches and how data flows between them. A developer adding a new audit type must update PostgreSQL schema, Neo4j sync logic, TimescaleDB write path, and potentially OpenSearch indexing. This coordination overhead is real.

5. **Cost is significantly higher.** Three database clusters (with replicas for HA), a Kafka cluster, Debezium connectors, and sync services. For an early-stage startup, this infrastructure cost may be prohibitive. Neo4j Enterprise licensing adds further cost, though the free Community Edition works for smaller deployments.

6. **Testing requires multi-store fixtures.** Integration tests must set up state in PostgreSQL, verify it syncs to Neo4j, insert time-series data into TimescaleDB, and assert correct results across all stores. Test setup and teardown become significantly more complex.

7. **Neo4j learning curve is steep.** Cypher query language, graph data modeling patterns, and Neo4j operational practices are unfamiliar to most developers. The team needs at least one graph database specialist, which is a harder hire than a PostgreSQL generalist.

---

## Migration and Scaling Considerations

### Phased Rollout Strategy

The polyglot architecture does not need to be deployed all at once. A phased approach reduces risk:

**Phase 1 (MVP): PostgreSQL only**
- Deploy the full PostgreSQL schema (with JSONB for flexibility)
- This is equivalent to Data Model Suggestion 3
- Serves 1-100 franchisors

**Phase 2 (Analytics): Add TimescaleDB**
- TimescaleDB is a PostgreSQL extension, so it can be added to the existing PostgreSQL instance or a separate one
- Migrate KPI and revenue time-series data into hypertables
- Enable continuous aggregates for dashboards
- Triggered when analytics query performance degrades or data volume exceeds 100M rows

**Phase 3 (Network Intelligence): Add Neo4j**
- Deploy Neo4j alongside PostgreSQL
- Implement CDC pipeline with Debezium/Kafka
- Build graph sync services
- Enable territory optimization and network analysis features
- Triggered when franchise networks exceed 500 locations or territory optimization becomes a key feature

### Neo4j Scaling
- **Fabric**: Neo4j Fabric for sharding across multiple database instances
- **Read replicas**: for distributing analytical graph queries
- **Causal clustering**: for HA with automatic failover
- Typical sizing: 1 core server + 2 read replicas handles networks with 100,000+ nodes

### TimescaleDB Scaling
- **Distributed hypertables**: scale writes across multiple nodes
- **Compression**: enable on chunks older than 3-6 months
- **Retention policies**: automatically drop raw data older than retention period
- **Tiered storage**: move compressed chunks to cheaper storage (S3 via timescaledb-cloud)

### Disaster Recovery
- PostgreSQL: streaming replication + daily base backups + WAL archiving to S3
- Neo4j: online backup to S3 + causal cluster for HA
- TimescaleDB: same as PostgreSQL (it is PostgreSQL)
- Kafka: multi-broker replication + topic-level backup to S3
- Recovery priority: PostgreSQL first (source of truth), then rebuild Neo4j and TimescaleDB from CDC replay

### Data Sovereignty
- Deploy PostgreSQL, Neo4j, and TimescaleDB clusters in each required region
- Kafka topics partitioned by franchisor to enable regional routing
- CDC events for a given franchisor route only to that franchisor's regional cluster
- Cross-region queries (e.g., global network benchmarks) require explicit data sharing agreements and aggregation services that pull from regional stores

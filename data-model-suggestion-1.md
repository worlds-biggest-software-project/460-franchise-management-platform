# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

## Overview

This model uses a fully normalized relational schema in PostgreSQL, leveraging its mature ecosystem for ACID compliance, row-level security for multi-tenant isolation, PostGIS for territory management, and strong foreign key enforcement across all franchise management domains. Every entity is stored in dedicated tables with explicit relationships, ensuring referential integrity and straightforward query patterns.

The schema is organized into seven bounded contexts that mirror the platform's core capabilities: Organization & Tenancy, Franchisee Lifecycle, Operations & Compliance, Training & LMS, Financial & Royalties, Communication, and Analytics/KPIs.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Primary Database | PostgreSQL 16+ | Mature, extensible, PostGIS support, RLS for multi-tenancy |
| Geospatial | PostGIS 3.4+ | Territory polygon storage, overlap detection, proximity queries |
| Connection Pooling | PgBouncer | Handle high-concurrency from many franchise locations |
| Migrations | Flyway or Liquibase | Version-controlled schema evolution |
| ORM | Prisma, Drizzle, or SQLAlchemy | Type-safe query building with migration support |
| Full-Text Search | PostgreSQL tsvector + GIN indexes | Search across SOPs, training content, communications |
| Caching | Redis | Session management, frequently-accessed KPI dashboards |

---

## Multi-Tenancy Strategy

The platform uses a **shared database, shared schema with row-level security (RLS)** approach. Every tenant-scoped table includes a `franchisor_id` column, and PostgreSQL RLS policies automatically filter rows based on the authenticated session context.

```sql
-- Enable RLS on a table
ALTER TABLE franchisees ENABLE ROW LEVEL SECURITY;

-- Create policy for tenant isolation
CREATE POLICY tenant_isolation ON franchisees
    USING (franchisor_id = current_setting('app.current_franchisor_id')::uuid);

-- Set tenant context per request
SET app.current_franchisor_id = 'a1b2c3d4-e5f6-7890-abcd-ef1234567890';
```

---

## Schema Definition

### 1. Organization & Tenancy Context

```sql
-- ============================================================
-- ORGANIZATION & TENANCY
-- ============================================================

CREATE TABLE franchisors (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    legal_name          VARCHAR(500),
    tax_id              VARCHAR(100),
    industry_sector     VARCHAR(100) NOT NULL,  -- 'food_service', 'retail', 'hospitality', 'services'
    headquarters_country VARCHAR(3) NOT NULL,    -- ISO 3166-1 alpha-3
    headquarters_address TEXT,
    logo_url            TEXT,
    website_url         TEXT,
    subscription_tier   VARCHAR(50) NOT NULL DEFAULT 'standard',  -- 'starter', 'standard', 'enterprise'
    subscription_status VARCHAR(20) NOT NULL DEFAULT 'active',
    max_locations       INTEGER,
    settings            JSONB NOT NULL DEFAULT '{}',  -- global franchisor preferences
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
    phone               VARCHAR(50),
    avatar_url          TEXT,
    role                VARCHAR(50) NOT NULL,  -- 'franchisor_admin', 'ops_director', 'field_rep', 'franchisee_owner', 'franchisee_manager', 'franchisee_staff'
    status              VARCHAR(20) NOT NULL DEFAULT 'active',  -- 'active', 'suspended', 'invited', 'deactivated'
    timezone            VARCHAR(50) DEFAULT 'UTC',
    locale              VARCHAR(10) DEFAULT 'en-US',
    last_login_at       TIMESTAMPTZ,
    sso_provider        VARCHAR(50),
    sso_external_id     VARCHAR(255),
    mfa_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (franchisor_id, email)
);

CREATE TABLE roles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    name                VARCHAR(100) NOT NULL,
    description         TEXT,
    is_system_role      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (franchisor_id, name)
);

CREATE TABLE permissions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code                VARCHAR(100) NOT NULL UNIQUE,  -- 'audit.create', 'training.manage', 'royalty.view'
    description         TEXT,
    module              VARCHAR(50) NOT NULL  -- 'franchise', 'compliance', 'training', 'finance', 'comms'
);

CREATE TABLE role_permissions (
    role_id             UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id       UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id             UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    franchisee_id       UUID,  -- scoped to a specific franchisee if applicable
    granted_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    granted_by          UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id)
);
```

### 2. Franchisee Lifecycle Context

```sql
-- ============================================================
-- FRANCHISEE LIFECYCLE MANAGEMENT
-- ============================================================

CREATE TABLE franchise_brands (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    brand_guidelines_url TEXT,
    initial_franchise_fee NUMERIC(12,2),
    ongoing_royalty_pct  NUMERIC(5,3),  -- e.g. 6.500 = 6.5%
    ad_fund_pct         NUMERIC(5,3),
    typical_investment_min NUMERIC(14,2),
    typical_investment_max NUMERIC(14,2),
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE franchise_prospects (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    brand_id            UUID REFERENCES franchise_brands(id),
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    email               VARCHAR(320) NOT NULL,
    phone               VARCHAR(50),
    company_name        VARCHAR(255),
    source              VARCHAR(100),  -- 'website', 'referral', 'trade_show', 'broker'
    lead_score          NUMERIC(5,2),  -- AI-generated score 0-100
    pipeline_stage      VARCHAR(50) NOT NULL DEFAULT 'inquiry',
    -- 'inquiry', 'qualified', 'application', 'discovery_day', 'fdd_review', 'approved', 'declined', 'withdrawn'
    desired_territory   TEXT,
    investment_capacity NUMERIC(14,2),
    notes               TEXT,
    assigned_to         UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE prospect_activities (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prospect_id         UUID NOT NULL REFERENCES franchise_prospects(id) ON DELETE CASCADE,
    activity_type       VARCHAR(50) NOT NULL,  -- 'call', 'email', 'meeting', 'document_sent', 'stage_change'
    description         TEXT,
    performed_by        UUID REFERENCES users(id),
    performed_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE franchisees (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    brand_id            UUID NOT NULL REFERENCES franchise_brands(id),
    prospect_id         UUID REFERENCES franchise_prospects(id),
    entity_name         VARCHAR(500) NOT NULL,
    entity_type         VARCHAR(50),  -- 'llc', 'corporation', 'partnership', 'sole_proprietor'
    tax_id              VARCHAR(100),
    primary_owner_id    UUID REFERENCES users(id),
    status              VARCHAR(30) NOT NULL DEFAULT 'onboarding',
    -- 'onboarding', 'pre_opening', 'open', 'probation', 'good_standing', 'at_risk', 'suspended', 'terminated', 'transferred'
    risk_score          NUMERIC(5,2),  -- AI-generated, 0-100
    risk_factors        TEXT[],
    onboarding_started  DATE,
    opening_date        DATE,
    termination_date    DATE,
    renewal_date        DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE franchise_agreements (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    brand_id            UUID NOT NULL REFERENCES franchise_brands(id),
    agreement_number    VARCHAR(100) NOT NULL,
    agreement_type      VARCHAR(50) NOT NULL,  -- 'initial', 'renewal', 'transfer', 'amendment'
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    -- 'draft', 'pending_review', 'executed', 'expired', 'terminated'
    initial_fee         NUMERIC(12,2),
    royalty_rate         NUMERIC(5,3) NOT NULL,
    royalty_type         VARCHAR(30) NOT NULL DEFAULT 'percentage_gross',
    -- 'percentage_gross', 'percentage_net', 'fixed_monthly', 'tiered'
    ad_fund_rate        NUMERIC(5,3),
    minimum_royalty     NUMERIC(12,2),
    technology_fee      NUMERIC(12,2),
    term_years          INTEGER NOT NULL,
    start_date          DATE NOT NULL,
    end_date            DATE NOT NULL,
    renewal_options     INTEGER DEFAULT 0,
    document_url        TEXT,
    signed_at           TIMESTAMPTZ,
    signed_by_franchisor UUID REFERENCES users(id),
    signed_by_franchisee UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE franchise_agreement_tiers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agreement_id        UUID NOT NULL REFERENCES franchise_agreements(id) ON DELETE CASCADE,
    tier_order          INTEGER NOT NULL,
    revenue_floor       NUMERIC(14,2) NOT NULL,
    revenue_ceiling     NUMERIC(14,2),  -- NULL means uncapped
    royalty_rate         NUMERIC(5,3) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE locations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    name                VARCHAR(255) NOT NULL,
    store_number        VARCHAR(50),
    address_line1       VARCHAR(255) NOT NULL,
    address_line2       VARCHAR(255),
    city                VARCHAR(100) NOT NULL,
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20),
    country             VARCHAR(3) NOT NULL,  -- ISO 3166-1 alpha-3
    latitude            NUMERIC(10,7),
    longitude           NUMERIC(10,7),
    geom                GEOMETRY(Point, 4326),  -- PostGIS point
    phone               VARCHAR(50),
    email               VARCHAR(320),
    status              VARCHAR(20) NOT NULL DEFAULT 'planned',
    -- 'planned', 'under_construction', 'open', 'temporarily_closed', 'permanently_closed'
    opened_date         DATE,
    closed_date         DATE,
    square_footage      INTEGER,
    seating_capacity    INTEGER,
    timezone            VARCHAR(50),
    operating_hours     JSONB,  -- {"mon": {"open": "08:00", "close": "22:00"}, ...}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_locations_geom ON locations USING GIST (geom);

CREATE TABLE territories (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    brand_id            UUID NOT NULL REFERENCES franchise_brands(id),
    name                VARCHAR(255) NOT NULL,
    territory_type      VARCHAR(50) NOT NULL,  -- 'exclusive', 'protected', 'non_exclusive'
    definition_method   VARCHAR(50) NOT NULL,  -- 'polygon', 'zip_codes', 'radius', 'county', 'drive_time'
    boundary            GEOMETRY(MultiPolygon, 4326),  -- PostGIS polygon
    zip_codes           TEXT[],
    radius_miles        NUMERIC(8,2),
    center_point        GEOMETRY(Point, 4326),
    population_count    INTEGER,
    household_count     INTEGER,
    market_potential     NUMERIC(14,2),
    status              VARCHAR(20) NOT NULL DEFAULT 'available',
    -- 'available', 'reserved', 'assigned', 'split', 'retired'
    assigned_franchisee_id UUID REFERENCES franchisees(id),
    assigned_date       DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_territories_boundary ON territories USING GIST (boundary);

-- Territory overlap detection query example:
-- SELECT a.id, b.id, ST_Area(ST_Intersection(a.boundary, b.boundary))
-- FROM territories a JOIN territories b ON ST_Overlaps(a.boundary, b.boundary) AND a.id < b.id;

CREATE TABLE field_visits (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    location_id         UUID NOT NULL REFERENCES locations(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    field_rep_id        UUID NOT NULL REFERENCES users(id),
    visit_type          VARCHAR(50) NOT NULL,  -- 'routine', 'follow_up', 'opening_support', 'investigation'
    status              VARCHAR(30) NOT NULL DEFAULT 'scheduled',
    -- 'scheduled', 'confirmed', 'in_progress', 'completed', 'cancelled', 'no_show'
    scheduled_date      DATE NOT NULL,
    scheduled_time      TIME,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    summary             TEXT,
    next_visit_recommended DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE onboarding_checklists (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    brand_id            UUID NOT NULL REFERENCES franchise_brands(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    version             INTEGER NOT NULL DEFAULT 1,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE onboarding_checklist_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    checklist_id        UUID NOT NULL REFERENCES onboarding_checklists(id) ON DELETE CASCADE,
    sort_order          INTEGER NOT NULL,
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    required            BOOLEAN NOT NULL DEFAULT TRUE,
    due_days_offset     INTEGER,  -- days after onboarding start
    responsible_role    VARCHAR(50),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE franchisee_onboarding_progress (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    checklist_item_id   UUID NOT NULL REFERENCES onboarding_checklist_items(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',  -- 'pending', 'in_progress', 'completed', 'waived'
    completed_at        TIMESTAMPTZ,
    completed_by        UUID REFERENCES users(id),
    notes               TEXT,
    evidence_url        TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (franchisee_id, checklist_item_id)
);
```

### 3. Operations & Compliance Context

```sql
-- ============================================================
-- OPERATIONS STANDARDS & COMPLIANCE
-- ============================================================

CREATE TABLE document_categories (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    parent_id           UUID REFERENCES document_categories(id),
    name                VARCHAR(255) NOT NULL,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE operations_documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    category_id         UUID REFERENCES document_categories(id),
    title               VARCHAR(500) NOT NULL,
    document_type       VARCHAR(50) NOT NULL,  -- 'sop', 'brand_guideline', 'policy', 'form', 'checklist_template'
    version             INTEGER NOT NULL DEFAULT 1,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',  -- 'draft', 'review', 'published', 'archived'
    content_body        TEXT,
    file_url            TEXT,
    file_type           VARCHAR(20),  -- 'pdf', 'docx', 'html'
    file_size_bytes     BIGINT,
    effective_date      DATE,
    review_date         DATE,
    search_vector       TSVECTOR,
    published_by        UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ops_docs_search ON operations_documents USING GIN (search_vector);

CREATE TABLE document_versions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id         UUID NOT NULL REFERENCES operations_documents(id) ON DELETE CASCADE,
    version             INTEGER NOT NULL,
    content_body        TEXT,
    file_url            TEXT,
    change_summary      TEXT,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE document_acknowledgements (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id         UUID NOT NULL REFERENCES operations_documents(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    franchisee_id       UUID REFERENCES franchisees(id),
    acknowledged_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    version_acknowledged INTEGER NOT NULL,
    UNIQUE (document_id, user_id, version_acknowledged)
);

CREATE TABLE audit_templates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    audit_type          VARCHAR(50) NOT NULL,  -- 'operational', 'financial', 'food_safety', 'brand_standards', 'health_safety'
    version             INTEGER NOT NULL DEFAULT 1,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    passing_score       NUMERIC(5,2),  -- minimum score to pass, e.g., 80.00
    max_score           NUMERIC(7,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE audit_template_sections (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id         UUID NOT NULL REFERENCES audit_templates(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    sort_order          INTEGER NOT NULL,
    weight              NUMERIC(5,2) NOT NULL DEFAULT 1.00,  -- section weight for scoring
    description         TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE audit_template_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id          UUID NOT NULL REFERENCES audit_template_sections(id) ON DELETE CASCADE,
    sort_order          INTEGER NOT NULL,
    question_text       TEXT NOT NULL,
    question_type       VARCHAR(30) NOT NULL,  -- 'yes_no', 'score_1_5', 'score_1_10', 'text', 'photo', 'multi_select'
    is_critical         BOOLEAN NOT NULL DEFAULT FALSE,  -- auto-fail if not met
    max_points          NUMERIC(5,2) NOT NULL DEFAULT 1.00,
    guidance_text       TEXT,
    requires_photo      BOOLEAN NOT NULL DEFAULT FALSE,
    options             TEXT[],  -- for multi_select type
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE audits (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    template_id         UUID NOT NULL REFERENCES audit_templates(id),
    location_id         UUID NOT NULL REFERENCES locations(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    auditor_id          UUID NOT NULL REFERENCES users(id),
    status              VARCHAR(30) NOT NULL DEFAULT 'scheduled',
    -- 'scheduled', 'in_progress', 'pending_review', 'completed', 'cancelled'
    scheduled_date      DATE NOT NULL,
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    total_score         NUMERIC(7,2),
    max_possible_score  NUMERIC(7,2),
    score_percentage    NUMERIC(5,2),
    passed              BOOLEAN,
    overall_notes       TEXT,
    follow_up_required  BOOLEAN NOT NULL DEFAULT FALSE,
    follow_up_date      DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE audit_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audit_id            UUID NOT NULL REFERENCES audits(id) ON DELETE CASCADE,
    template_item_id    UUID NOT NULL REFERENCES audit_template_items(id),
    response_value      VARCHAR(500),  -- the raw answer
    score               NUMERIC(5,2),
    notes               TEXT,
    is_non_compliant    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE audit_evidence (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    audit_response_id   UUID NOT NULL REFERENCES audit_responses(id) ON DELETE CASCADE,
    evidence_type       VARCHAR(20) NOT NULL,  -- 'photo', 'video', 'document'
    file_url            TEXT NOT NULL,
    file_size_bytes     BIGINT,
    caption             TEXT,
    captured_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    latitude            NUMERIC(10,7),
    longitude           NUMERIC(10,7)
);

CREATE TABLE corrective_actions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    audit_id            UUID REFERENCES audits(id),
    audit_response_id   UUID REFERENCES audit_responses(id),
    location_id         UUID NOT NULL REFERENCES locations(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    title               VARCHAR(500) NOT NULL,
    description         TEXT NOT NULL,
    severity            VARCHAR(20) NOT NULL,  -- 'critical', 'major', 'minor', 'observation'
    status              VARCHAR(30) NOT NULL DEFAULT 'open',
    -- 'open', 'acknowledged', 'in_progress', 'resolved', 'verified', 'escalated', 'overdue'
    assigned_to         UUID REFERENCES users(id),
    due_date            DATE NOT NULL,
    resolved_at         TIMESTAMPTZ,
    verified_at         TIMESTAMPTZ,
    verified_by         UUID REFERENCES users(id),
    resolution_notes    TEXT,
    escalation_level    INTEGER NOT NULL DEFAULT 0,
    escalated_to        UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE regulatory_requirements (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    name                VARCHAR(255) NOT NULL,
    regulation_body     VARCHAR(255),  -- 'OSHA', 'FDA', 'local_health_dept'
    regulation_code     VARCHAR(100),
    jurisdiction_country VARCHAR(3) NOT NULL,
    jurisdiction_state   VARCHAR(100),
    description         TEXT,
    compliance_deadline DATE,
    recurrence          VARCHAR(30),  -- 'annual', 'quarterly', 'monthly', 'one_time'
    applies_to_brands   UUID[],  -- array of brand_ids
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 4. Training & LMS Context

```sql
-- ============================================================
-- TRAINING & LEARNING MANAGEMENT (SCORM-COMPLIANT)
-- ============================================================

CREATE TABLE training_programs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    brand_id            UUID REFERENCES franchise_brands(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    program_type        VARCHAR(50) NOT NULL,  -- 'initial_training', 'ongoing', 'certification', 'remedial', 'compliance'
    target_roles        TEXT[],  -- roles this program applies to
    estimated_hours     NUMERIC(6,1),
    is_mandatory        BOOLEAN NOT NULL DEFAULT FALSE,
    prerequisite_program_id UUID REFERENCES training_programs(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE courses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    training_program_id UUID REFERENCES training_programs(id),
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    course_type         VARCHAR(30) NOT NULL,  -- 'scorm', 'video', 'document', 'quiz', 'live_session', 'on_the_job'
    scorm_package_url   TEXT,
    scorm_version       VARCHAR(20),  -- 'scorm_1.2', 'scorm_2004_3rd', 'scorm_2004_4th'
    content_url         TEXT,
    duration_minutes    INTEGER,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    passing_score       NUMERIC(5,2),
    max_attempts        INTEGER,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE course_modules (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id           UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    title               VARCHAR(500) NOT NULL,
    sort_order          INTEGER NOT NULL,
    module_type         VARCHAR(30) NOT NULL,  -- 'lesson', 'quiz', 'assignment', 'resource'
    content_url         TEXT,
    scorm_sco_id        VARCHAR(255),  -- SCORM Shareable Content Object identifier
    duration_minutes    INTEGER,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE enrollments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    course_id           UUID NOT NULL REFERENCES courses(id),
    franchisee_id       UUID REFERENCES franchisees(id),
    location_id         UUID REFERENCES locations(id),
    enrolled_by         UUID REFERENCES users(id),
    enrolled_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    due_date            DATE,
    status              VARCHAR(30) NOT NULL DEFAULT 'enrolled',
    -- 'enrolled', 'in_progress', 'completed', 'failed', 'expired', 'waived'
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    score               NUMERIC(5,2),
    attempts            INTEGER NOT NULL DEFAULT 0,
    time_spent_seconds  INTEGER NOT NULL DEFAULT 0,
    certificate_url     TEXT,
    UNIQUE (user_id, course_id)
);

CREATE TABLE scorm_tracking (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID NOT NULL REFERENCES enrollments(id) ON DELETE CASCADE,
    sco_id              VARCHAR(255) NOT NULL,
    attempt_number      INTEGER NOT NULL DEFAULT 1,
    -- SCORM CMI data model fields
    cmi_completion_status VARCHAR(20),  -- 'completed', 'incomplete', 'not_attempted', 'unknown'
    cmi_success_status  VARCHAR(20),    -- 'passed', 'failed', 'unknown'
    cmi_score_raw       NUMERIC(7,2),
    cmi_score_min       NUMERIC(7,2),
    cmi_score_max       NUMERIC(7,2),
    cmi_score_scaled    NUMERIC(5,4),   -- -1.0 to 1.0
    cmi_total_time      INTERVAL,
    cmi_session_time    INTERVAL,
    cmi_location        TEXT,           -- bookmark / resume position
    cmi_suspend_data    TEXT,           -- opaque learner state
    cmi_progress_measure NUMERIC(5,4), -- 0.0 to 1.0
    last_interaction_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE scorm_interactions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tracking_id         UUID NOT NULL REFERENCES scorm_tracking(id) ON DELETE CASCADE,
    interaction_id      VARCHAR(255) NOT NULL,
    interaction_type    VARCHAR(30),  -- 'true-false', 'choice', 'fill-in', 'matching', 'performance', 'sequencing', 'likert', 'numeric', 'other'
    learner_response    TEXT,
    correct_response    TEXT,
    result              VARCHAR(30),  -- 'correct', 'incorrect', 'unanticipated', 'neutral'
    weighting           NUMERIC(5,2),
    latency             INTERVAL,
    description         TEXT,
    timestamp_utc       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE certifications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    required_program_id UUID REFERENCES training_programs(id),
    validity_months     INTEGER,  -- NULL means never expires
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE user_certifications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    certification_id    UUID NOT NULL REFERENCES certifications(id),
    franchisee_id       UUID REFERENCES franchisees(id),
    issued_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at          TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',  -- 'active', 'expired', 'revoked'
    certificate_number  VARCHAR(100),
    certificate_url     TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 5. Financial & Royalties Context

```sql
-- ============================================================
-- FINANCIAL & ROYALTIES
-- ============================================================

CREATE TABLE revenue_reports (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    location_id         UUID NOT NULL REFERENCES locations(id),
    reporting_period    DATE NOT NULL,  -- first day of period
    period_type         VARCHAR(20) NOT NULL DEFAULT 'monthly',  -- 'weekly', 'monthly'
    gross_revenue       NUMERIC(14,2) NOT NULL,
    net_revenue         NUMERIC(14,2),
    cost_of_goods       NUMERIC(14,2),
    labor_cost          NUMERIC(14,2),
    occupancy_cost      NUMERIC(14,2),
    other_expenses      NUMERIC(14,2),
    transaction_count   INTEGER,
    average_ticket      NUMERIC(10,2),
    source              VARCHAR(30) NOT NULL DEFAULT 'manual',  -- 'manual', 'pos_import', 'accounting_sync'
    status              VARCHAR(20) NOT NULL DEFAULT 'submitted',  -- 'submitted', 'verified', 'disputed', 'adjusted'
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    verified_at         TIMESTAMPTZ,
    verified_by         UUID REFERENCES users(id),
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
    applicable_revenue  NUMERIC(14,2) NOT NULL,  -- after exclusions
    royalty_rate_applied NUMERIC(5,3) NOT NULL,
    royalty_amount      NUMERIC(12,2) NOT NULL,
    ad_fund_rate        NUMERIC(5,3),
    ad_fund_amount      NUMERIC(12,2),
    technology_fee      NUMERIC(12,2),
    other_fees          NUMERIC(12,2),
    total_due           NUMERIC(12,2) NOT NULL,
    minimum_applied     BOOLEAN NOT NULL DEFAULT FALSE,
    tier_applied        VARCHAR(50),
    calculation_notes   TEXT,
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
    subtotal            NUMERIC(12,2) NOT NULL,
    tax_amount          NUMERIC(12,2) NOT NULL DEFAULT 0,
    late_fee            NUMERIC(12,2) NOT NULL DEFAULT 0,
    credits_applied     NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_amount        NUMERIC(12,2) NOT NULL,
    balance_due         NUMERIC(12,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD',
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    -- 'draft', 'sent', 'viewed', 'partially_paid', 'paid', 'overdue', 'written_off'
    sent_at             TIMESTAMPTZ,
    paid_at             TIMESTAMPTZ,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE invoice_line_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id          UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    royalty_calc_id     UUID REFERENCES royalty_calculations(id),
    description         VARCHAR(500) NOT NULL,
    line_type           VARCHAR(30) NOT NULL,  -- 'royalty', 'ad_fund', 'technology_fee', 'late_fee', 'credit', 'other'
    quantity            NUMERIC(10,2) NOT NULL DEFAULT 1,
    unit_amount         NUMERIC(12,2) NOT NULL,
    total_amount        NUMERIC(12,2) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE payments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    invoice_id          UUID REFERENCES invoices(id),
    payment_method      VARCHAR(30) NOT NULL,  -- 'ach', 'wire', 'check', 'credit_card', 'auto_debit'
    payment_reference   VARCHAR(255),
    amount              NUMERIC(12,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD',
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- 'pending', 'processing', 'completed', 'failed', 'refunded'
    processed_at        TIMESTAMPTZ,
    external_txn_id     VARCHAR(255),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 6. Communication Context

```sql
-- ============================================================
-- COMMUNICATION & COLLABORATION
-- ============================================================

CREATE TABLE announcements (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    author_id           UUID NOT NULL REFERENCES users(id),
    title               VARCHAR(500) NOT NULL,
    body                TEXT NOT NULL,
    priority            VARCHAR(20) NOT NULL DEFAULT 'normal',  -- 'low', 'normal', 'high', 'urgent'
    target_audience     VARCHAR(50) NOT NULL DEFAULT 'all',  -- 'all', 'owners', 'managers', 'staff', 'custom'
    target_brands       UUID[],
    target_regions      TEXT[],
    requires_acknowledgement BOOLEAN NOT NULL DEFAULT FALSE,
    published_at        TIMESTAMPTZ,
    expires_at          TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE announcement_recipients (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    announcement_id     UUID NOT NULL REFERENCES announcements(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id),
    read_at             TIMESTAMPTZ,
    acknowledged_at     TIMESTAMPTZ,
    delivery_channel    VARCHAR(20),  -- 'in_app', 'email', 'sms'
    delivered_at        TIMESTAMPTZ,
    UNIQUE (announcement_id, user_id)
);

CREATE TABLE messages (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    thread_id           UUID,  -- self-referencing for threads
    sender_id           UUID NOT NULL REFERENCES users(id),
    subject             VARCHAR(500),
    body                TEXT NOT NULL,
    channel             VARCHAR(20) NOT NULL DEFAULT 'in_app',  -- 'in_app', 'email', 'sms'
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE message_recipients (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id          UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    recipient_id        UUID NOT NULL REFERENCES users(id),
    read_at             TIMESTAMPTZ,
    UNIQUE (message_id, recipient_id)
);

CREATE TABLE support_tickets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    ticket_number       VARCHAR(50) NOT NULL,
    created_by          UUID NOT NULL REFERENCES users(id),
    franchisee_id       UUID REFERENCES franchisees(id),
    location_id         UUID REFERENCES locations(id),
    category            VARCHAR(100) NOT NULL,  -- 'operations', 'technology', 'marketing', 'finance', 'training', 'general'
    subject             VARCHAR(500) NOT NULL,
    description         TEXT NOT NULL,
    priority            VARCHAR(20) NOT NULL DEFAULT 'medium',
    status              VARCHAR(30) NOT NULL DEFAULT 'open',
    -- 'open', 'assigned', 'in_progress', 'waiting_on_requestor', 'resolved', 'closed'
    assigned_to         UUID REFERENCES users(id),
    resolved_at         TIMESTAMPTZ,
    satisfaction_rating INTEGER,  -- 1-5
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (franchisor_id, ticket_number)
);

CREATE TABLE ticket_comments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id           UUID NOT NULL REFERENCES support_tickets(id) ON DELETE CASCADE,
    author_id           UUID NOT NULL REFERENCES users(id),
    body                TEXT NOT NULL,
    is_internal         BOOLEAN NOT NULL DEFAULT FALSE,  -- internal notes not visible to requestor
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE marketing_assets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    brand_id            UUID REFERENCES franchise_brands(id),
    name                VARCHAR(255) NOT NULL,
    asset_type          VARCHAR(50) NOT NULL,  -- 'logo', 'banner', 'flyer', 'menu', 'social_post', 'video', 'template'
    category            VARCHAR(100),
    file_url            TEXT NOT NULL,
    thumbnail_url       TEXT,
    file_format         VARCHAR(20),
    file_size_bytes     BIGINT,
    usage_guidelines    TEXT,
    approved            BOOLEAN NOT NULL DEFAULT FALSE,
    approved_by         UUID REFERENCES users(id),
    expires_at          DATE,
    download_count      INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 7. Analytics & KPI Context

```sql
-- ============================================================
-- ANALYTICS & KPI TRACKING
-- ============================================================

CREATE TABLE kpi_definitions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    name                VARCHAR(255) NOT NULL,
    code                VARCHAR(100) NOT NULL,  -- 'same_store_sales_growth', 'labor_cost_ratio', 'auv', 'inventory_turnover'
    description         TEXT,
    unit                VARCHAR(30) NOT NULL,  -- 'percentage', 'currency', 'ratio', 'count', 'days'
    calculation_formula TEXT,
    target_value        NUMERIC(14,4),
    warning_threshold   NUMERIC(14,4),
    critical_threshold  NUMERIC(14,4),
    higher_is_better    BOOLEAN NOT NULL DEFAULT TRUE,
    category            VARCHAR(50),  -- 'financial', 'operational', 'customer', 'people'
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (franchisor_id, code)
);

CREATE TABLE kpi_snapshots (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    kpi_definition_id   UUID NOT NULL REFERENCES kpi_definitions(id),
    location_id         UUID NOT NULL REFERENCES locations(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    value               NUMERIC(14,4) NOT NULL,
    previous_value      NUMERIC(14,4),
    change_pct          NUMERIC(7,2),
    benchmark_network_avg NUMERIC(14,4),
    benchmark_top_quartile NUMERIC(14,4),
    status              VARCHAR(20),  -- 'on_track', 'warning', 'critical'
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (kpi_definition_id, location_id, period_start)
);

CREATE INDEX idx_kpi_snapshots_period ON kpi_snapshots (franchisor_id, period_start, kpi_definition_id);

CREATE TABLE anomaly_detections (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    kpi_snapshot_id     UUID REFERENCES kpi_snapshots(id),
    location_id         UUID NOT NULL REFERENCES locations(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    anomaly_type        VARCHAR(50) NOT NULL,  -- 'spike', 'drop', 'trend_reversal', 'outlier', 'pattern_break'
    severity            VARCHAR(20) NOT NULL,  -- 'info', 'warning', 'critical'
    metric_name         VARCHAR(255) NOT NULL,
    expected_value      NUMERIC(14,4),
    actual_value        NUMERIC(14,4),
    deviation_pct       NUMERIC(7,2),
    description         TEXT,
    ai_recommendation   TEXT,
    acknowledged        BOOLEAN NOT NULL DEFAULT FALSE,
    acknowledged_by     UUID REFERENCES users(id),
    acknowledged_at     TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- AUDIT LOG (platform-wide)
-- ============================================================

CREATE TABLE audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    franchisor_id       UUID NOT NULL,
    user_id             UUID,
    action              VARCHAR(50) NOT NULL,  -- 'create', 'update', 'delete', 'login', 'export', 'approve'
    entity_type         VARCHAR(100) NOT NULL,  -- 'franchisee', 'audit', 'invoice', 'course', etc.
    entity_id           UUID,
    changes             JSONB,  -- {"field": {"old": "x", "new": "y"}}
    ip_address          INET,
    user_agent          TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_log_entity ON audit_log (franchisor_id, entity_type, entity_id);
CREATE INDEX idx_audit_log_time ON audit_log (franchisor_id, created_at DESC);
```

---

## Key Indexes and Performance Considerations

```sql
-- Composite indexes for common query patterns
CREATE INDEX idx_franchisees_status ON franchisees (franchisor_id, status);
CREATE INDEX idx_locations_franchisee ON locations (franchisee_id, status);
CREATE INDEX idx_audits_location_date ON audits (franchisor_id, location_id, scheduled_date DESC);
CREATE INDEX idx_corrective_actions_status ON corrective_actions (franchisor_id, status, due_date);
CREATE INDEX idx_enrollments_user ON enrollments (user_id, status);
CREATE INDEX idx_invoices_status ON invoices (franchisor_id, status, due_date);
CREATE INDEX idx_revenue_reports_period ON revenue_reports (franchisor_id, reporting_period DESC);
CREATE INDEX idx_payments_franchisee ON payments (franchisee_id, created_at DESC);

-- Partial indexes for active records
CREATE INDEX idx_active_franchisees ON franchisees (franchisor_id) WHERE status NOT IN ('terminated', 'transferred');
CREATE INDEX idx_open_corrective_actions ON corrective_actions (franchisor_id, due_date) WHERE status NOT IN ('resolved', 'verified');
CREATE INDEX idx_unpaid_invoices ON invoices (franchisor_id, due_date) WHERE status NOT IN ('paid', 'written_off');
```

---

## Pros and Cons

### Pros

1. **Complete referential integrity.** Every relationship is enforced by foreign keys. Orphaned records, dangling references, and inconsistent state are impossible at the database level. For a franchise network where financial calculations depend on agreement terms linking to specific franchisees and locations, this correctness guarantee is essential.

2. **Mature tooling ecosystem.** PostgreSQL has decades of production-hardened tooling: pg_dump for backups, pg_restore for recovery, pgBouncer for connection pooling, pgAdmin for visual management, and hundreds of ORMs and migration tools across every major language.

3. **Row-level security for multi-tenancy.** RLS policies are enforced at the database engine level, not the application layer. Even if application code has a bug that omits a WHERE clause, franchisor A can never see franchisor B's data. This is critical for a platform serving competing franchise networks.

4. **PostGIS for territory management.** Territory polygons, location points, proximity queries, and overlap detection are handled natively with spatial indexes. No external geospatial service is needed.

5. **Strong query flexibility.** Complex reporting queries -- year-over-year same-store sales, tiered royalty calculations, multi-location audit score comparisons -- are expressed naturally in SQL with window functions, CTEs, and aggregations.

6. **ACID compliance for financial data.** Royalty calculations, invoice generation, and payment recording demand transactional guarantees. A single transaction can atomically calculate royalties, create invoice line items, and update account balances.

7. **Full-text search built in.** The `tsvector`/`tsquery` system with GIN indexes provides adequate search across SOPs, training content, and communications without requiring a separate search service like Elasticsearch.

### Cons

1. **Schema rigidity.** Adding a new field to audit checklists, a new KPI metric, or a new compliance requirement demands a schema migration. For a platform that must accommodate dozens of different franchise industries (food service, retail, hospitality, services), the variety of required fields can lead to frequent ALTER TABLE operations.

2. **Horizontal scaling limitations.** PostgreSQL scales vertically well but horizontal sharding is complex. A franchise platform serving thousands of franchisors with millions of locations will eventually need partitioning strategies (e.g., by franchisor_id range) or external sharding solutions like Citus.

3. **High table count increases operational complexity.** This schema has 50+ tables. Schema migrations must be coordinated across all tables, and developers need deep familiarity with the data model. Join-heavy queries across many tables can be slow without careful index management.

4. **No native event history.** The audit_log table captures changes, but reconstructing the full history of a franchisee or computing "what was the state on date X" requires parsing JSONB change logs rather than replaying typed events. This limits the power of historical analysis and temporal queries.

5. **SCORM data awkwardness.** SCORM's `suspend_data` and `cmi_location` fields are opaque blobs that don't benefit from relational normalization. Storing them in VARCHAR columns works but loses the self-describing nature of document storage.

6. **Report generation can be expensive.** Network-wide benchmarking queries that aggregate across thousands of locations and multiple KPIs require careful query optimization, materialized views, or pre-computed summary tables to avoid full table scans during dashboard rendering.

---

## Migration and Scaling Considerations

### Initial Deployment (1-50 Franchisors)
- Single PostgreSQL instance with read replica
- Connection pooling via PgBouncer (100-500 connections)
- Daily pg_dump backups with point-in-time recovery enabled
- Materialized views for dashboard KPIs, refreshed hourly

### Growth Phase (50-500 Franchisors)
- Move to managed PostgreSQL (AWS RDS, GCP Cloud SQL, Azure Flexible Server)
- Add read replicas for reporting workloads
- Partition `audit_log`, `kpi_snapshots`, and `revenue_reports` by date range
- Introduce Redis caching for frequently-accessed dashboards
- Consider Citus extension for distributed queries

### Scale Phase (500+ Franchisors)
- Partition large tables by `franchisor_id` using PostgreSQL declarative partitioning
- Deploy Citus for horizontal sharding across nodes
- Offload analytics queries to a data warehouse (BigQuery, Snowflake) via CDC
- Implement connection routing to direct heavy reporting to dedicated replicas
- Consider extracting the LMS module into its own database if SCORM tracking volume dominates

### Migration Strategy
- Use Flyway or Liquibase for versioned, repeatable migrations
- All migrations must be backward-compatible (add columns as nullable, rename via views)
- Blue-green deployment with schema compatibility windows
- Feature flags to gradually adopt new schema structures
- Maintain migration test suite that applies all migrations from scratch on every CI run

### Data Sovereignty
- PostgreSQL's logical replication can sync subsets of data to regional replicas
- RLS policies ensure data isolation even if multiple regions share infrastructure
- For strict data sovereignty (EU data stays in EU), deploy separate PostgreSQL clusters per region with application-level routing based on franchisor jurisdiction

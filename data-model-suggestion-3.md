# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL with JSONB)

## Overview

This model uses PostgreSQL as a single database engine but strategically blends normalized relational columns for stable, frequently-queried data with JSONB columns for flexible, variable, and evolving data. The core insight is that franchise management spans many industries (food service, retail, hospitality, services) and each franchisor has distinct operational standards, audit checklists, KPI definitions, and compliance requirements. A purely normalized schema either becomes impossibly wide (hundreds of nullable columns) or impossibly deep (entity-attribute-value anti-pattern). JSONB solves this by letting the variable parts flex while the relational structure enforces integrity on the stable parts.

This hybrid approach gives the platform the ability to onboard a pizza franchise, a hair salon chain, and a hotel group onto the same schema -- each with radically different audit criteria, training requirements, and performance metrics -- without schema migrations.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Primary Database | PostgreSQL 16+ | Native JSONB with GIN indexing, JSON path queries, pg_jsonschema validation |
| JSONB Validation | pg_jsonschema extension | Enforce JSON Schema constraints on JSONB columns at the database level |
| Geospatial | PostGIS 3.4+ | Territory polygon storage and spatial queries |
| Connection Pooling | PgBouncer | Handle multi-location concurrent connections |
| ORM | Prisma (with JSON type support) or TypeORM | Typed access to both relational and JSONB fields |
| Search | PostgreSQL tsvector + jsonb_to_tsvector | Full-text search across both relational and JSONB content |
| Caching | Redis | Dashboard caching, session management |
| Migrations | Flyway + custom JSONB schema versioning | Relational migrations + JSON Schema evolution tracking |

---

## Design Principles

### What Goes in Relational Columns
- Identity fields (IDs, names, emails)
- Status and lifecycle fields (queried in WHERE clauses, used in indexes)
- Foreign keys and relationships
- Financial amounts (royalties, payments -- demand precision and aggregation)
- Dates and timestamps (range queries, sorting)
- Fields used in JOINs or GROUP BY operations

### What Goes in JSONB Columns
- Industry-specific attributes (restaurant seating vs. retail shelf layout vs. hotel room count)
- Configurable form data (audit checklist responses, onboarding questionnaires)
- Settings and preferences (per-franchisor, per-location customization)
- Integration payloads (data from POS systems, accounting software, HR platforms)
- Template definitions (audit templates, training program structures)
- Historical snapshots (point-in-time state captures)

---

## Schema Definition

### 1. Organization & Multi-Tenancy

```sql
-- ============================================================
-- ORGANIZATION & TENANCY
-- ============================================================

CREATE TABLE franchisors (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    legal_name          VARCHAR(500),
    industry_sector     VARCHAR(100) NOT NULL,
    headquarters_country VARCHAR(3) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    subscription_tier   VARCHAR(50) NOT NULL DEFAULT 'standard',

    -- JSONB: Industry-specific settings, branding preferences, feature flags
    -- Schema varies by industry and subscription tier
    settings            JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example for food service:
    -- {
    --   "branding": {"primary_color": "#FF0000", "font_family": "Inter"},
    --   "features": {"ai_anomaly_detection": true, "scorm_lms": true, "territory_mapping": true},
    --   "defaults": {"royalty_period": "monthly", "audit_frequency_days": 90, "currency": "USD"},
    --   "integrations": {"quickbooks": {"enabled": true, "company_id": "..."}, "toast_pos": {"enabled": true}},
    --   "compliance": {"gdpr_enabled": true, "data_retention_days": 2555, "jurisdictions": ["US", "CAN", "GBR"]}
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Validate settings structure
ALTER TABLE franchisors ADD CONSTRAINT chk_franchisor_settings
    CHECK (jsonb_typeof(settings) = 'object');

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    email               VARCHAR(320) NOT NULL,
    password_hash       VARCHAR(255),
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    role                VARCHAR(50) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',

    -- JSONB: Profile preferences, notification settings, UI state
    profile             JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "phone": "+1-555-0123",
    --   "timezone": "America/New_York",
    --   "locale": "en-US",
    --   "notifications": {"email": true, "sms": false, "push": true},
    --   "dashboard_layout": {"widgets": ["revenue", "audits", "training"]},
    --   "sso": {"provider": "okta", "external_id": "..."}
    -- }

    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (franchisor_id, email)
);

-- RBAC remains fully relational -- permissions are stable and query-critical
CREATE TABLE roles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    name                VARCHAR(100) NOT NULL,
    permissions         TEXT[] NOT NULL DEFAULT '{}',  -- array of permission codes
    is_system_role      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (franchisor_id, name)
);

CREATE TABLE user_roles (
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id             UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    scope_franchisee_id UUID,  -- NULL means global within franchisor
    granted_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, role_id)
);
```

### 2. Franchisee Lifecycle

```sql
-- ============================================================
-- FRANCHISEE LIFECYCLE
-- ============================================================

CREATE TABLE franchise_brands (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    name                VARCHAR(255) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',

    -- JSONB: Brand-specific configuration that varies widely by industry
    config              JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "fee_structure": {
    --     "initial_franchise_fee": 35000,
    --     "royalty_pct": 6.5,
    --     "ad_fund_pct": 2.0,
    --     "technology_fee_monthly": 500,
    --     "investment_range": {"min": 250000, "max": 750000}
    --   },
    --   "brand_guidelines_url": "https://...",
    --   "operating_model": {
    --     "store_types": ["standalone", "inline", "kiosk", "drive_thru"],
    --     "typical_sqft": {"min": 1200, "max": 2400},
    --     "typical_staff": {"min": 8, "max": 25}
    --   },
    --   "onboarding_defaults": {
    --     "training_weeks": 6,
    --     "pre_opening_weeks": 12
    --   }
    -- }

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
    source              VARCHAR(100),
    lead_score          NUMERIC(5,2),
    pipeline_stage      VARCHAR(50) NOT NULL DEFAULT 'inquiry',
    assigned_to         UUID REFERENCES users(id),

    -- JSONB: Prospect qualification data, application answers, discovery day notes
    -- Structure varies by brand's qualification process
    qualification_data  JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "phone": "+1-555-0456",
    --   "company_name": "Smith Holdings LLC",
    --   "investment_capacity": 500000,
    --   "desired_territories": ["Chicago North", "Chicago West"],
    --   "prior_franchise_experience": true,
    --   "background": {"industry": "food_service", "years": 12, "current_role": "restaurant_gm"},
    --   "application_answers": {
    --     "q1_motivation": "...",
    --     "q2_management_experience": "...",
    --     "q3_financial_disclosure": {...}
    --   },
    --   "discovery_day": {"attended": true, "date": "2025-03-15", "feedback": "very positive"},
    --   "fdd_review": {"received": true, "acknowledged": true, "attorney_reviewed": false}
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_prospects_stage ON franchise_prospects (franchisor_id, pipeline_stage);
CREATE INDEX idx_prospects_score ON franchise_prospects (franchisor_id, lead_score DESC NULLS LAST);

CREATE TABLE franchisees (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    brand_id            UUID NOT NULL REFERENCES franchise_brands(id),
    entity_name         VARCHAR(500) NOT NULL,
    primary_owner_id    UUID REFERENCES users(id),
    status              VARCHAR(30) NOT NULL DEFAULT 'onboarding',
    risk_score          NUMERIC(5,2),
    opening_date        DATE,
    renewal_date        DATE,

    -- JSONB: Entity details, owner information, risk factors
    -- Varies by jurisdiction and entity type
    entity_details      JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "entity_type": "llc",
    --   "tax_id": "XX-XXXXXXX",
    --   "state_of_formation": "Delaware",
    --   "owners": [
    --     {"name": "John Smith", "ownership_pct": 60, "is_managing_member": true},
    --     {"name": "Jane Smith", "ownership_pct": 40, "is_managing_member": false}
    --   ],
    --   "banking": {"bank_name": "Chase", "routing_number": "...", "account_last4": "1234"},
    --   "insurance": {"provider": "State Farm", "policy_number": "...", "expiration": "2026-01-15"},
    --   "risk_factors": ["declining_revenue_3m", "overdue_corrective_actions", "expired_food_safety_cert"]
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_franchisees_status ON franchisees (franchisor_id, status);
CREATE INDEX idx_franchisees_risk ON franchisees (franchisor_id, risk_score DESC NULLS LAST);

CREATE TABLE franchise_agreements (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    brand_id            UUID NOT NULL REFERENCES franchise_brands(id),
    agreement_number    VARCHAR(100) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    start_date          DATE NOT NULL,
    end_date            DATE NOT NULL,

    -- JSONB: Full financial terms -- complex, varies per agreement
    -- Structured this way because fee structures differ: flat vs. tiered vs. graduated vs. minimum
    financial_terms     JSONB NOT NULL,
    -- {
    --   "royalty": {
    --     "type": "tiered",
    --     "tiers": [
    --       {"floor": 0, "ceiling": 500000, "rate": 0.065},
    --       {"floor": 500000, "ceiling": 1000000, "rate": 0.055},
    --       {"floor": 1000000, "ceiling": null, "rate": 0.045}
    --     ],
    --     "minimum_monthly": 2000,
    --     "basis": "gross_revenue"
    --   },
    --   "ad_fund": {"type": "percentage", "rate": 0.02, "basis": "gross_revenue"},
    --   "technology_fee": {"type": "fixed", "amount": 500, "frequency": "monthly"},
    --   "initial_fee": 35000,
    --   "renewal_fee": 15000,
    --   "transfer_fee": 10000
    -- }

    -- JSONB: Legal terms, territory rights, operational obligations
    terms_and_conditions JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "territory_rights": "exclusive",
    --   "renewal_options": 2,
    --   "renewal_term_years": 10,
    --   "non_compete_radius_miles": 5,
    --   "non_compete_years_after_term": 2,
    --   "required_operating_hours": {"weekday": {"open": "06:00", "close": "22:00"}, "weekend": {"open": "07:00", "close": "23:00"}},
    --   "minimum_staffing": 8,
    --   "required_certifications": ["food_safety_manager", "alcohol_service"]
    -- }

    document_url        TEXT,
    signed_at           TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE locations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    name                VARCHAR(255) NOT NULL,
    store_number        VARCHAR(50),
    status              VARCHAR(20) NOT NULL DEFAULT 'planned',
    -- Address fields remain relational for geocoding and standard queries
    address_line1       VARCHAR(255) NOT NULL,
    city                VARCHAR(100) NOT NULL,
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20),
    country             VARCHAR(3) NOT NULL,
    geom                GEOMETRY(Point, 4326),
    opened_date         DATE,

    -- JSONB: Location-specific attributes that vary by industry
    attributes          JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Food service example:
    -- {
    --   "square_footage": 1800,
    --   "seating_capacity": 45,
    --   "drive_thru": true,
    --   "delivery_enabled": true,
    --   "delivery_radius_miles": 5,
    --   "equipment": ["fryer_double", "oven_conveyor", "pos_toast"],
    --   "lease": {"landlord": "ABC Properties", "term_end": "2030-06-30", "monthly_rent": 4500},
    --   "operating_hours": {"mon-fri": {"open": "06:00", "close": "22:00"}, "sat-sun": {"open": "07:00", "close": "23:00"}},
    --   "pos_system": {"vendor": "Toast", "integration_id": "..."},
    --   "licenses": [
    --     {"type": "food_service", "number": "FS-12345", "expires": "2026-03-15"},
    --     {"type": "liquor", "number": "LQ-67890", "expires": "2026-06-30"}
    --   ]
    -- }
    --
    -- Hotel example:
    -- {
    --   "room_count": 120,
    --   "meeting_rooms": 4,
    --   "pool": true,
    --   "fitness_center": true,
    --   "star_rating": 3,
    --   "pms_system": {"vendor": "Opera", "property_id": "..."},
    --   "amenities": ["wifi", "breakfast", "parking", "ev_charging"]
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_locations_geom ON locations USING GIST (geom);
CREATE INDEX idx_locations_status ON locations (franchisor_id, status);
-- GIN index on JSONB for attribute queries
CREATE INDEX idx_locations_attrs ON locations USING GIN (attributes jsonb_path_ops);

CREATE TABLE territories (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    brand_id            UUID NOT NULL REFERENCES franchise_brands(id),
    name                VARCHAR(255) NOT NULL,
    territory_type      VARCHAR(50) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'available',
    assigned_franchisee_id UUID REFERENCES franchisees(id),
    boundary            GEOMETRY(MultiPolygon, 4326),

    -- JSONB: Territory demographics, market analysis, assignment history
    demographics        JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "population": 85000,
    --   "households": 32000,
    --   "median_income": 72000,
    --   "age_distribution": {"18_24": 0.12, "25_34": 0.18, "35_44": 0.22, "45_54": 0.20, "55_plus": 0.28},
    --   "zip_codes": ["60601", "60602", "60603"],
    --   "market_potential_score": 78.5,
    --   "competitor_density": 3,
    --   "nearest_existing_location_miles": 12.4,
    --   "assignment_history": [
    --     {"franchisee_id": "...", "from": "2020-01-01", "to": "2023-06-30", "reason": "transfer"}
    --   ]
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_territories_boundary ON territories USING GIST (boundary);
```

### 3. Operations & Compliance

```sql
-- ============================================================
-- OPERATIONS & COMPLIANCE
-- ============================================================

-- Document management stays mostly relational (standard CRUD with versioning)
CREATE TABLE operations_documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    title               VARCHAR(500) NOT NULL,
    document_type       VARCHAR(50) NOT NULL,
    version             INTEGER NOT NULL DEFAULT 1,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    content_body        TEXT,
    file_url            TEXT,
    effective_date      DATE,
    search_vector       TSVECTOR,

    -- JSONB: Document metadata, category hierarchy, access controls
    metadata            JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "category_path": ["Operations", "Food Safety", "Temperature Monitoring"],
    --   "tags": ["food_safety", "haccp", "temperature"],
    --   "applies_to_brands": ["brand-uuid-1"],
    --   "applies_to_roles": ["franchisee_owner", "franchisee_manager"],
    --   "requires_acknowledgement": true,
    --   "review_schedule": {"frequency_months": 12, "next_review": "2026-01-15"}
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ops_docs_search ON operations_documents USING GIN (search_vector);
CREATE INDEX idx_ops_docs_metadata ON operations_documents USING GIN (metadata jsonb_path_ops);

-- Audit templates use JSONB extensively because every franchisor has
-- completely different audit criteria, scoring methods, and question types
CREATE TABLE audit_templates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    name                VARCHAR(255) NOT NULL,
    audit_type          VARCHAR(50) NOT NULL,
    version             INTEGER NOT NULL DEFAULT 1,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,

    -- JSONB: The entire template definition -- sections, items, scoring rules
    -- This is the primary use case for JSONB: audit templates vary enormously
    -- between franchisors and change frequently
    template_definition JSONB NOT NULL,
    -- {
    --   "passing_score": 80,
    --   "max_score": 100,
    --   "scoring_method": "weighted",  // or "simple", "pass_fail"
    --   "sections": [
    --     {
    --       "id": "sec-1",
    --       "name": "Food Safety & Hygiene",
    --       "weight": 0.35,
    --       "items": [
    --         {
    --           "id": "item-1-1",
    --           "question": "Are all cold storage units at or below 40F (4C)?",
    --           "type": "yes_no",
    --           "is_critical": true,
    --           "max_points": 10,
    --           "requires_photo": true,
    --           "guidance": "Check thermometer in each refrigerator and walk-in cooler",
    --           "regulatory_reference": "FDA Food Code 3-501.16"
    --         },
    --         {
    --           "id": "item-1-2",
    --           "question": "Rate the cleanliness of food preparation surfaces",
    --           "type": "score_1_5",
    --           "is_critical": false,
    --           "max_points": 5,
    --           "requires_photo": false,
    --           "scoring_rubric": {
    --             "1": "Visibly dirty, immediate attention needed",
    --             "2": "Below standard, needs cleaning",
    --             "3": "Acceptable, meets minimum requirements",
    --             "4": "Good, well-maintained",
    --             "5": "Excellent, spotless"
    --           }
    --         },
    --         {
    --           "id": "item-1-3",
    --           "question": "Which food safety certifications are current?",
    --           "type": "multi_select",
    --           "options": ["ServSafe Manager", "ServSafe Food Handler", "HACCP", "Allergen Awareness"],
    --           "min_required": 1,
    --           "max_points": 4
    --         }
    --       ]
    --     },
    --     {
    --       "id": "sec-2",
    --       "name": "Brand Standards & Presentation",
    --       "weight": 0.25,
    --       "items": [...]
    --     }
    --   ]
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Audits: relational for queryable fields, JSONB for responses
CREATE TABLE audits (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    template_id         UUID NOT NULL REFERENCES audit_templates(id),
    template_version    INTEGER NOT NULL,  -- snapshot which version was used
    location_id         UUID NOT NULL REFERENCES locations(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    auditor_id          UUID NOT NULL REFERENCES users(id),
    status              VARCHAR(30) NOT NULL DEFAULT 'scheduled',
    scheduled_date      DATE NOT NULL,
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    total_score         NUMERIC(7,2),
    max_possible_score  NUMERIC(7,2),
    score_percentage    NUMERIC(5,2),
    passed              BOOLEAN,

    -- JSONB: All responses stored as a single document
    -- This avoids the need for separate audit_responses and audit_evidence tables
    -- and keeps the full audit as an atomic, self-contained document
    responses           JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [
    --   {
    --     "item_id": "item-1-1",
    --     "section_id": "sec-1",
    --     "response_value": "yes",
    --     "score": 10,
    --     "is_non_compliant": false,
    --     "notes": "All units checked, within range",
    --     "evidence": [
    --       {"type": "photo", "url": "https://...", "caption": "Walk-in cooler thermometer", "captured_at": "2025-03-15T10:30:00Z", "lat": 41.8781, "lng": -87.6298}
    --     ]
    --   },
    --   {
    --     "item_id": "item-1-2",
    --     "section_id": "sec-1",
    --     "response_value": "4",
    --     "score": 4,
    --     "is_non_compliant": false,
    --     "notes": null,
    --     "evidence": []
    --   }
    -- ]

    -- JSONB: Section-level score summaries for quick dashboard queries
    section_scores      JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [
    --   {"section_id": "sec-1", "name": "Food Safety & Hygiene", "score": 42, "max_score": 50, "pct": 84.0},
    --   {"section_id": "sec-2", "name": "Brand Standards", "score": 22, "max_score": 30, "pct": 73.3}
    -- ]

    overall_notes       TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audits_location ON audits (franchisor_id, location_id, scheduled_date DESC);
CREATE INDEX idx_audits_score ON audits (franchisor_id, score_percentage);
-- GIN index for querying specific responses within the JSONB
CREATE INDEX idx_audits_responses ON audits USING GIN (responses jsonb_path_ops);

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
    resolved_at         TIMESTAMPTZ,

    -- JSONB: Resolution details, escalation history, evidence
    details             JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "description": "Cold storage unit in kitchen at 45F, exceeds 40F maximum",
    --   "audit_item_id": "item-1-1",
    --   "regulatory_reference": "FDA Food Code 3-501.16",
    --   "root_cause": "Compressor failing, unit too old",
    --   "resolution": {
    --     "action_taken": "Replaced refrigeration unit",
    --     "completed_at": "2025-03-20T14:00:00Z",
    --     "cost": 2500,
    --     "evidence": [{"type": "photo", "url": "https://...", "caption": "New unit installed"}]
    --   },
    --   "escalation_history": [
    --     {"level": 1, "escalated_to": "user-uuid", "reason": "Not resolved within 48 hours", "at": "2025-03-18T09:00:00Z"},
    --     {"level": 2, "escalated_to": "user-uuid", "reason": "Critical food safety issue", "at": "2025-03-19T09:00:00Z"}
    --   ],
    --   "verification": {
    --     "verified_by": "user-uuid",
    --     "verified_at": "2025-03-22T10:00:00Z",
    --     "notes": "New unit confirmed at 38F"
    --   }
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_corrective_actions_status ON corrective_actions (franchisor_id, status, due_date);
```

### 4. Training & LMS

```sql
-- ============================================================
-- TRAINING & LEARNING MANAGEMENT
-- ============================================================

CREATE TABLE training_programs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    brand_id            UUID REFERENCES franchise_brands(id),
    name                VARCHAR(255) NOT NULL,
    program_type        VARCHAR(50) NOT NULL,
    is_mandatory        BOOLEAN NOT NULL DEFAULT FALSE,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',

    -- JSONB: Program structure, prerequisites, role targeting
    config              JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "target_roles": ["franchisee_owner", "franchisee_manager"],
    --   "estimated_hours": 40,
    --   "prerequisites": ["program-uuid-basic-training"],
    --   "completion_criteria": {
    --     "all_courses_completed": true,
    --     "min_overall_score": 80,
    --     "max_completion_days": 90
    --   },
    --   "certification": {
    --     "name": "Certified Franchise Operator",
    --     "validity_months": 24,
    --     "renewal_required": true
    --   }
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE courses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    training_program_id UUID REFERENCES training_programs(id),
    title               VARCHAR(500) NOT NULL,
    course_type         VARCHAR(30) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    sort_order          INTEGER NOT NULL DEFAULT 0,

    -- JSONB: Course content structure, SCORM metadata, quiz definitions
    content             JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- For SCORM courses:
    -- {
    --   "scorm": {
    --     "package_url": "https://...",
    --     "version": "scorm_2004_4th",
    --     "launch_url": "index.html",
    --     "mastery_score": 80,
    --     "max_attempts": 3
    --   },
    --   "duration_minutes": 45,
    --   "modules": [
    --     {"sco_id": "sco-001", "title": "Introduction to Food Safety", "launch_url": "mod1/index.html"}
    --   ]
    -- }
    --
    -- For quiz courses:
    -- {
    --   "duration_minutes": 30,
    --   "passing_score": 80,
    --   "max_attempts": 3,
    --   "randomize_questions": true,
    --   "questions": [
    --     {
    --       "id": "q1",
    --       "type": "multiple_choice",
    --       "text": "What is the maximum safe temperature for cold storage?",
    --       "options": ["32F", "40F", "45F", "50F"],
    --       "correct_answer": "40F",
    --       "points": 10,
    --       "explanation": "FDA Food Code requires cold storage at 40F (4C) or below"
    --     }
    --   ]
    -- }

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
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    score               NUMERIC(5,2),
    attempts            INTEGER NOT NULL DEFAULT 0,
    time_spent_seconds  INTEGER NOT NULL DEFAULT 0,

    -- JSONB: SCORM tracking data, quiz responses, session history
    -- SCORM's CMI data model maps naturally to JSONB
    tracking_data       JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "scorm": {
    --     "current_sco": "sco-001",
    --     "scos": {
    --       "sco-001": {
    --         "completion_status": "completed",
    --         "success_status": "passed",
    --         "score": {"raw": 92, "min": 0, "max": 100, "scaled": 0.92},
    --         "total_time": "PT2H15M30S",
    --         "location": "page_42",
    --         "suspend_data": "base64encodedstring...",
    --         "progress_measure": 1.0,
    --         "interactions": [
    --           {"id": "q1", "type": "choice", "response": "40F", "result": "correct", "latency": "PT45S"}
    --         ]
    --       }
    --     }
    --   },
    --   "session_history": [
    --     {"started": "2025-03-15T10:00:00Z", "ended": "2025-03-15T10:45:00Z", "duration_seconds": 2700},
    --     {"started": "2025-03-16T14:00:00Z", "ended": "2025-03-16T15:30:00Z", "duration_seconds": 5400}
    --   ]
    -- }

    UNIQUE (user_id, course_id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_enrollments_status ON enrollments (franchisee_id, status);

CREATE TABLE user_certifications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    franchisor_id       UUID NOT NULL,
    certification_name  VARCHAR(255) NOT NULL,
    issued_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at          TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    certificate_number  VARCHAR(100),
    certificate_url     TEXT,

    -- JSONB: Certification details, what was completed to earn it
    details             JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "program_id": "program-uuid",
    --   "courses_completed": ["course-uuid-1", "course-uuid-2"],
    --   "final_score": 88.5,
    --   "issuing_authority": "FranchiseCo Training Department"
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 5. Financial & Royalties

```sql
-- ============================================================
-- FINANCIAL & ROYALTIES
-- ============================================================

-- Revenue reports: mostly relational for aggregation and financial precision
CREATE TABLE revenue_reports (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    location_id         UUID NOT NULL REFERENCES locations(id),
    reporting_period    DATE NOT NULL,
    period_type         VARCHAR(20) NOT NULL DEFAULT 'monthly',
    gross_revenue       NUMERIC(14,2) NOT NULL,
    net_revenue         NUMERIC(14,2),
    source              VARCHAR(30) NOT NULL DEFAULT 'manual',
    status              VARCHAR(20) NOT NULL DEFAULT 'submitted',

    -- JSONB: Revenue breakdown varies by industry
    -- A restaurant tracks food vs. beverage vs. catering
    -- A hotel tracks rooms vs. F&B vs. events vs. spa
    -- A retail franchise tracks categories/departments
    revenue_breakdown   JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Restaurant example:
    -- {
    --   "food": 85000, "beverage": 22000, "catering": 8000, "delivery": 15000,
    --   "cost_of_goods": 38000, "labor_cost": 32000, "occupancy_cost": 6500,
    --   "transaction_count": 4200, "average_ticket": 31.43,
    --   "daypart_breakdown": {"breakfast": 18000, "lunch": 42000, "dinner": 55000, "late_night": 15000}
    -- }

    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    verified_at         TIMESTAMPTZ,
    verified_by         UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (location_id, reporting_period, period_type)
);

CREATE INDEX idx_revenue_period ON revenue_reports (franchisor_id, reporting_period DESC);

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

    -- JSONB: Calculation breakdown showing how we arrived at the amounts
    calculation_details JSONB NOT NULL,
    -- {
    --   "royalty": {
    --     "method": "tiered",
    --     "tiers_applied": [
    --       {"tier": 1, "revenue_range": [0, 500000], "rate": 0.065, "amount": 5850},
    --       {"tier": 2, "revenue_range": [500000, 750000], "rate": 0.055, "amount": 1375}
    --     ],
    --     "minimum_check": {"minimum": 2000, "calculated": 7225, "minimum_applied": false}
    --   },
    --   "deductions": [
    --     {"type": "new_store_discount", "amount": -500, "reason": "First 6 months: 50% royalty reduction"}
    --   ],
    --   "agreement_terms_snapshot": {...}  // frozen copy of terms at calculation time
    -- }

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

    -- JSONB: Line items, payment history, adjustments
    line_items          JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [
    --   {"type": "royalty", "description": "Royalty Fee - March 2025", "amount": 7225.00, "calc_id": "calc-uuid"},
    --   {"type": "ad_fund", "description": "Advertising Fund - March 2025", "amount": 1500.00},
    --   {"type": "technology_fee", "description": "Technology Fee - March 2025", "amount": 500.00},
    --   {"type": "late_fee", "description": "Late fee from February invoice", "amount": 150.00}
    -- ]

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_invoices_status ON invoices (franchisor_id, status, due_date);

CREATE TABLE payments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    franchisee_id       UUID NOT NULL REFERENCES franchisees(id),
    invoice_id          UUID REFERENCES invoices(id),
    amount              NUMERIC(12,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD',
    payment_method      VARCHAR(30) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    processed_at        TIMESTAMPTZ,

    -- JSONB: Payment gateway details, reconciliation info
    payment_details     JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "gateway": "stripe",
    --   "external_txn_id": "txn_abc123",
    --   "payment_reference": "CHK-4567",
    --   "bank_account_last4": "1234",
    --   "reconciliation": {"matched": true, "matched_at": "2025-04-02T10:00:00Z"}
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 6. Communication & Support

```sql
-- ============================================================
-- COMMUNICATION & SUPPORT
-- ============================================================

CREATE TABLE announcements (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    author_id           UUID NOT NULL REFERENCES users(id),
    title               VARCHAR(500) NOT NULL,
    body                TEXT NOT NULL,
    priority            VARCHAR(20) NOT NULL DEFAULT 'normal',
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    published_at        TIMESTAMPTZ,
    expires_at          TIMESTAMPTZ,

    -- JSONB: Targeting rules, delivery tracking
    targeting           JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "audience": "custom",
    --   "brands": ["brand-uuid-1"],
    --   "regions": ["midwest", "northeast"],
    --   "roles": ["franchisee_owner"],
    --   "requires_acknowledgement": true,
    --   "delivery_channels": ["in_app", "email"],
    --   "delivery_stats": {
    --     "total_recipients": 145,
    --     "delivered": 142,
    --     "read": 98,
    --     "acknowledged": 85
    --   }
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE support_tickets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    ticket_number       VARCHAR(50) NOT NULL,
    created_by          UUID NOT NULL REFERENCES users(id),
    franchisee_id       UUID REFERENCES franchisees(id),
    location_id         UUID REFERENCES locations(id),
    category            VARCHAR(100) NOT NULL,
    subject             VARCHAR(500) NOT NULL,
    priority            VARCHAR(20) NOT NULL DEFAULT 'medium',
    status              VARCHAR(30) NOT NULL DEFAULT 'open',
    assigned_to         UUID REFERENCES users(id),
    resolved_at         TIMESTAMPTZ,

    -- JSONB: Ticket thread, attachments, SLA tracking
    thread              JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [
    --   {
    --     "id": "comment-uuid-1",
    --     "author_id": "user-uuid",
    --     "author_name": "John Smith",
    --     "body": "Our POS system is not syncing revenue reports...",
    --     "is_internal": false,
    --     "attachments": [{"name": "error_screenshot.png", "url": "https://..."}],
    --     "created_at": "2025-03-15T10:30:00Z"
    --   },
    --   {
    --     "id": "comment-uuid-2",
    --     "author_id": "user-uuid",
    --     "author_name": "Support Agent",
    --     "body": "I can see the integration error. Resyncing now...",
    --     "is_internal": false,
    --     "attachments": [],
    --     "created_at": "2025-03-15T11:45:00Z"
    --   }
    -- ]

    -- JSONB: SLA and satisfaction metrics
    metrics             JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "sla": {"response_target_hours": 4, "resolution_target_hours": 24},
    --   "first_response_at": "2025-03-15T11:45:00Z",
    --   "first_response_minutes": 75,
    --   "sla_response_met": true,
    --   "resolution_minutes": 180,
    --   "sla_resolution_met": true,
    --   "satisfaction_rating": 4,
    --   "satisfaction_comment": "Quick resolution, thank you!"
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (franchisor_id, ticket_number)
);

CREATE INDEX idx_tickets_status ON support_tickets (franchisor_id, status, priority);
```

### 7. Analytics & KPI

```sql
-- ============================================================
-- ANALYTICS & KPI TRACKING
-- ============================================================

CREATE TABLE kpi_definitions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL REFERENCES franchisors(id),
    code                VARCHAR(100) NOT NULL,
    name                VARCHAR(255) NOT NULL,
    category            VARCHAR(50),
    higher_is_better    BOOLEAN NOT NULL DEFAULT TRUE,

    -- JSONB: KPI configuration -- thresholds, formulas, display settings
    config              JSONB NOT NULL,
    -- {
    --   "unit": "percentage",
    --   "decimal_places": 1,
    --   "calculation": {
    --     "formula": "(current_period_revenue - prior_year_period_revenue) / prior_year_period_revenue * 100",
    --     "inputs": ["current_period_revenue", "prior_year_period_revenue"]
    --   },
    --   "thresholds": {
    --     "target": 5.0,
    --     "warning": 0.0,
    --     "critical": -5.0
    --   },
    --   "display": {
    --     "chart_type": "line",
    --     "color_scheme": "traffic_light",
    --     "show_trend": true,
    --     "benchmark_comparison": true
    --   }
    -- }

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
    status              VARCHAR(20),

    -- JSONB: Benchmark comparisons, contributing factors
    context             JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "network_avg": 4.2,
    --   "network_median": 3.8,
    --   "top_quartile": 7.5,
    --   "rank": 15,
    --   "rank_of": 200,
    --   "quartile": 2,
    --   "contributing_factors": ["new_menu_launch", "seasonal_peak"]
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (kpi_definition_id, location_id, period_start)
);

CREATE INDEX idx_kpi_period ON kpi_snapshots (franchisor_id, period_start DESC, kpi_definition_id);

-- ============================================================
-- PLATFORM AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    franchisor_id       UUID NOT NULL,
    user_id             UUID,
    action              VARCHAR(50) NOT NULL,
    entity_type         VARCHAR(100) NOT NULL,
    entity_id           UUID,
    -- JSONB: old_data, new_data, and request context
    change_data         JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {
    --   "old": {"status": "open", "assigned_to": null},
    --   "new": {"status": "in_progress", "assigned_to": "user-uuid"},
    --   "context": {"ip": "192.168.1.1", "user_agent": "...", "request_id": "..."}
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_log_entity ON audit_log (franchisor_id, entity_type, entity_id);
CREATE INDEX idx_audit_log_time ON audit_log (franchisor_id, created_at DESC);
```

---

## JSONB Schema Validation

Use pg_jsonschema to enforce structure on critical JSONB columns:

```sql
-- Install the extension
CREATE EXTENSION IF NOT EXISTS pg_jsonschema;

-- Validate financial terms in franchise agreements
ALTER TABLE franchise_agreements ADD CONSTRAINT chk_financial_terms
    CHECK (pg_jsonschema_is_valid(financial_terms, '{
        "type": "object",
        "required": ["royalty"],
        "properties": {
            "royalty": {
                "type": "object",
                "required": ["type"],
                "properties": {
                    "type": {"type": "string", "enum": ["percentage", "tiered", "fixed", "graduated"]},
                    "rate": {"type": "number", "minimum": 0, "maximum": 1},
                    "minimum_monthly": {"type": "number", "minimum": 0}
                }
            },
            "initial_fee": {"type": "number", "minimum": 0},
            "ad_fund": {"type": "object"},
            "technology_fee": {"type": "object"}
        }
    }'::jsonb));

-- Validate audit template structure
ALTER TABLE audit_templates ADD CONSTRAINT chk_template_definition
    CHECK (pg_jsonschema_is_valid(template_definition, '{
        "type": "object",
        "required": ["sections", "scoring_method"],
        "properties": {
            "passing_score": {"type": "number", "minimum": 0, "maximum": 100},
            "scoring_method": {"type": "string", "enum": ["weighted", "simple", "pass_fail"]},
            "sections": {
                "type": "array",
                "minItems": 1,
                "items": {
                    "type": "object",
                    "required": ["id", "name", "items"],
                    "properties": {
                        "id": {"type": "string"},
                        "name": {"type": "string"},
                        "weight": {"type": "number", "minimum": 0, "maximum": 1},
                        "items": {"type": "array", "minItems": 1}
                    }
                }
            }
        }
    }'::jsonb));
```

---

## Query Examples

```sql
-- Find all locations with drive-thru capability
SELECT id, name, store_number
FROM locations
WHERE franchisor_id = $1
  AND attributes @> '{"drive_thru": true}'::jsonb;

-- Calculate average audit score by section across all locations
SELECT
    s.value->>'name' AS section_name,
    AVG((s.value->>'pct')::numeric) AS avg_score_pct,
    MIN((s.value->>'pct')::numeric) AS min_score,
    MAX((s.value->>'pct')::numeric) AS max_score
FROM audits a,
     jsonb_array_elements(a.section_scores) AS s(value)
WHERE a.franchisor_id = $1
  AND a.completed_at >= NOW() - INTERVAL '12 months'
GROUP BY s.value->>'name'
ORDER BY avg_score_pct;

-- Find franchisees with tiered royalty agreements where minimum was applied
SELECT f.entity_name, rc.billing_period, rc.total_due,
       rc.calculation_details->'royalty'->>'method' AS method
FROM royalty_calculations rc
JOIN franchisees f ON f.id = rc.franchisee_id
WHERE rc.franchisor_id = $1
  AND rc.calculation_details->'royalty'->'minimum_check'->>'minimum_applied' = 'true';

-- Get SCORM progress for a specific learner
SELECT c.title,
       e.tracking_data->'scorm'->'scos' AS sco_progress,
       e.score,
       e.status
FROM enrollments e
JOIN courses c ON c.id = e.course_id
WHERE e.user_id = $1
  AND c.course_type = 'scorm';
```

---

## Pros and Cons

### Pros

1. **Single database engine, no additional infrastructure.** Everything runs in PostgreSQL. No separate document database, no search cluster, no time-series database. Operations teams manage one database, one backup strategy, one monitoring setup. This dramatically reduces deployment complexity compared to polyglot persistence.

2. **Industry flexibility without schema migrations.** A pizza franchise stores `seating_capacity` and `drive_thru` in location attributes. A hotel stores `room_count` and `star_rating`. A service franchise stores `service_area_radius` and `mobile_units`. All use the same `locations` table with different JSONB content. Onboarding a new industry vertical requires zero DDL changes.

3. **Configurable audit templates without EAV anti-pattern.** Audit template definitions stored as JSONB documents avoid the explosion of tables (audit_template_items, audit_template_sections, audit_template_options, audit_item_scoring_rubrics) that a fully normalized approach requires. A single `template_definition` JSONB column holds the entire checklist structure, and a single `responses` JSONB column holds all answers for an audit instance.

4. **Financial precision preserved.** Revenue amounts, royalty calculations, and payment amounts remain as NUMERIC columns with proper precision. Aggregation queries (SUM, AVG) work correctly. Only the calculation breakdown details (how tiers were applied, what deductions applied) go into JSONB.

5. **SCORM data fits JSONB naturally.** SCORM's CMI data model (suspend_data, interactions, objectives) is inherently hierarchical and variable. Storing it as JSONB in the enrollment's `tracking_data` column maps cleanly to the SCORM API's get/set operations without needing to normalize into relational tables.

6. **Referential integrity on critical relationships.** Foreign keys still enforce that an audit references a valid location, a payment references a valid invoice, and an enrollment references a valid user and course. The JSONB flexibility applies to attributes, not to entity relationships.

7. **Queryable JSONB with GIN indexes.** PostgreSQL's `jsonb_path_ops` GIN indexes make JSONB containment queries (`@>`) fast. Finding "all locations with drive-thru" or "all agreements with tiered royalty" performs well even with large datasets.

### Cons

1. **JSONB columns resist type safety.** Despite pg_jsonschema validation, JSONB content is not type-checked by the database the way relational columns are. A developer can insert `"revenue": "not_a_number"` into a revenue_breakdown JSONB field without the database objecting (unless explicit JSON Schema constraints are added for every field). This shifts validation responsibility to the application layer.

2. **JSONB updates rewrite the entire document.** Updating a single field in a large JSONB column (e.g., adding one audit response to a `responses` array with 200 items) rewrites the whole column value. For frequently-updated JSONB documents like SCORM tracking data during an active learning session, this creates write amplification.

3. **Complex JSONB queries are verbose and slow.** While simple containment queries are fast, complex JSONB operations (joining data across JSONB fields in different tables, aggregating nested arrays, filtering on deeply nested paths) produce verbose SQL and can be slow if the query cannot use a GIN index. Developers need to understand which JSONB query patterns are indexable.

4. **Schema evolution is implicit.** When the structure of a JSONB column changes (e.g., audit template v1 has `max_points` per item, v2 adds `scoring_rubric`), there is no database migration to apply. Old documents coexist with new documents. Application code must handle both shapes, and there is no central registry of what shapes exist.

5. **Reporting across JSONB requires ETL.** Business intelligence tools (Tableau, Looker, Metabase) work best with flat relational tables. Generating reports that span JSONB content (e.g., "average food cost percentage across all locations" where food cost is inside revenue_breakdown JSONB) requires either complex SQL views that extract JSONB fields or an ETL pipeline that flattens JSONB into relational tables.

6. **No foreign key enforcement inside JSONB.** The `responses` array in an audit references `item_id` values from the template, but the database cannot enforce that these IDs actually exist. Similarly, `assignment_history` in territory demographics references `franchisee_id` values without foreign key protection. Application logic must maintain these references.

7. **Testing complexity.** Unit and integration tests must cover both relational constraint validation and JSONB content validation. Test fixtures become more complex because they must produce valid JSONB structures for every JSONB column.

---

## Migration and Scaling Considerations

### JSONB Schema Versioning Strategy

```sql
-- Track JSONB schema versions per table/column
CREATE TABLE jsonb_schema_versions (
    id                  SERIAL PRIMARY KEY,
    table_name          VARCHAR(100) NOT NULL,
    column_name         VARCHAR(100) NOT NULL,
    version             INTEGER NOT NULL,
    json_schema         JSONB NOT NULL,  -- the JSON Schema definition
    migration_notes     TEXT,
    applied_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (table_name, column_name, version)
);

-- When reading, application code checks:
-- 1. Does the JSONB document have a _schema_version field?
-- 2. If not, assume v1 and upcast to current version
-- 3. If yes, apply upcasters for versions between document version and current
```

### Partitioning Strategy
- Partition `audit_log` by month (high-volume, time-ordered)
- Partition `kpi_snapshots` by quarter (time-series data)
- Partition `revenue_reports` by year (financial archive)
- Keep entity tables unpartitioned until they exceed 10M rows per franchisor

### Read Scaling
- Create materialized views for dashboard aggregations
- Refresh materialized views on a schedule (hourly for KPIs, daily for benchmarks)
- Use pg_cron for automated refresh
- Add read replicas for reporting workloads

### JSONB Performance Optimization
- Use `jsonb_path_ops` GIN indexes for containment queries on large JSONB columns
- Use expression indexes on frequently-queried JSONB paths: `CREATE INDEX idx_location_sqft ON locations ((attributes->>'square_footage')::int)`
- Avoid deeply nested JSONB structures (max 3-4 levels)
- Consider extracting hot JSONB fields into relational columns when query patterns stabilize

### Migration from Pure Relational
1. Add JSONB columns to existing tables with DEFAULT '{}'
2. Backfill JSONB from existing relational columns/tables
3. Update application code to read from JSONB
4. Verify data consistency between relational and JSONB representations
5. Drop redundant relational columns/tables once stable
6. Add pg_jsonschema constraints on critical JSONB columns

### Data Sovereignty
- Same approach as pure relational: regional PostgreSQL clusters with application-level routing
- JSONB content follows the same RLS policies as relational columns
- Export/delete operations for GDPR compliance must traverse JSONB content (use jsonb_path_query to find PII in JSONB columns)

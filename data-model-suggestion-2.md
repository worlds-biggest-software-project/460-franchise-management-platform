# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Overview

This model applies Event Sourcing and Command Query Responsibility Segregation (CQRS) to the franchise management platform. Instead of storing the current state of entities in mutable rows, every state change is captured as an immutable event in an append-only event store. Read models (projections) are built by replaying these events into optimized query-side views.

This approach is particularly compelling for franchise management because the domain is inherently event-driven: a franchisee moves through lifecycle stages, audits produce findings that trigger corrective actions, training completions unlock certifications, revenue reports flow into royalty calculations, and every step has compliance and audit trail implications. Event sourcing makes this temporal evolution a first-class citizen rather than an afterthought.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Event Store | EventStoreDB or PostgreSQL (append-only) | Purpose-built event persistence with stream support |
| Command Service | Node.js/TypeScript or Java/Kotlin with Axon Framework | Strong CQRS/ES framework support |
| Read Database | PostgreSQL 16+ | Relational projections for complex queries |
| Search Projections | Elasticsearch / OpenSearch | Full-text search across SOPs, training, communications |
| KPI Projections | TimescaleDB | Time-series optimized views for performance metrics |
| Geospatial Projections | PostgreSQL + PostGIS | Territory and location queries |
| Message Broker | Apache Kafka or NATS JetStream | Event distribution to projection builders and external systems |
| Caching | Redis | Materialized view caching, session management |
| Schema Registry | Confluent Schema Registry or custom | Event schema versioning and evolution |

---

## Event Store Schema

The event store uses PostgreSQL as the backing store for maximum operational familiarity, with an append-only design enforced by application-level controls and database triggers.

```sql
-- ============================================================
-- EVENT STORE (Write Side)
-- ============================================================

CREATE TABLE event_streams (
    stream_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type         VARCHAR(100) NOT NULL,  -- aggregate type: 'Franchisee', 'Audit', 'Course', 'Invoice'
    aggregate_id        UUID NOT NULL,
    franchisor_id       UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (stream_type, aggregate_id)
);

CREATE INDEX idx_streams_franchisor ON event_streams (franchisor_id, stream_type);

CREATE TABLE events (
    global_position     BIGSERIAL PRIMARY KEY,  -- monotonically increasing, total order
    stream_id           UUID NOT NULL REFERENCES event_streams(stream_id),
    stream_position     INTEGER NOT NULL,        -- position within this stream (optimistic concurrency)
    event_type          VARCHAR(200) NOT NULL,    -- e.g., 'FranchiseeOnboarded', 'AuditCompleted'
    event_data          JSONB NOT NULL,           -- the event payload
    metadata            JSONB NOT NULL DEFAULT '{}',  -- causation_id, correlation_id, user_id, ip, etc.
    franchisor_id       UUID NOT NULL,
    occurred_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    schema_version      INTEGER NOT NULL DEFAULT 1,
    UNIQUE (stream_id, stream_position)
);

CREATE INDEX idx_events_type ON events (event_type, occurred_at);
CREATE INDEX idx_events_franchisor ON events (franchisor_id, occurred_at);
CREATE INDEX idx_events_global ON events (global_position);
CREATE INDEX idx_events_correlation ON events USING GIN ((metadata->'correlation_id'));

-- Prevent updates and deletes on events (immutability)
CREATE OR REPLACE FUNCTION prevent_event_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'Events are immutable. UPDATE and DELETE operations are not permitted.';
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_event_immutability
    BEFORE UPDATE OR DELETE ON events
    FOR EACH ROW EXECUTE FUNCTION prevent_event_mutation();

-- Snapshot table for aggregate state restoration without full replay
CREATE TABLE snapshots (
    stream_id           UUID NOT NULL REFERENCES event_streams(stream_id),
    stream_position     INTEGER NOT NULL,
    snapshot_data       JSONB NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (stream_id, stream_position)
);

-- Projection checkpoint tracking
CREATE TABLE projection_checkpoints (
    projection_name     VARCHAR(200) PRIMARY KEY,
    last_global_position BIGINT NOT NULL DEFAULT 0,
    last_processed_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status              VARCHAR(20) NOT NULL DEFAULT 'running',  -- 'running', 'paused', 'rebuilding', 'error'
    error_message       TEXT,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Dead letter queue for failed event processing
CREATE TABLE dead_letter_events (
    id                  BIGSERIAL PRIMARY KEY,
    global_position     BIGINT NOT NULL,
    projection_name     VARCHAR(200) NOT NULL,
    event_type          VARCHAR(200) NOT NULL,
    error_message       TEXT NOT NULL,
    retry_count         INTEGER NOT NULL DEFAULT 0,
    max_retries         INTEGER NOT NULL DEFAULT 5,
    next_retry_at       TIMESTAMPTZ,
    resolved            BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Event Type Catalog

### Franchisee Lifecycle Events

```typescript
// ── Prospect Events ──────────────────────────────────────────
interface ProspectRegistered {
  type: 'ProspectRegistered';
  data: {
    prospectId: string;
    franchisorId: string;
    firstName: string;
    lastName: string;
    email: string;
    source: 'website' | 'referral' | 'trade_show' | 'broker';
    desiredTerritory?: string;
    investmentCapacity?: number;
  };
}

interface ProspectQualified {
  type: 'ProspectQualified';
  data: {
    prospectId: string;
    leadScore: number;
    qualifiedBy: string;
    qualificationNotes: string;
  };
}

interface ProspectAdvancedStage {
  type: 'ProspectAdvancedStage';
  data: {
    prospectId: string;
    fromStage: string;
    toStage: string;
    advancedBy: string;
    notes?: string;
  };
}

// ── Franchisee Events ────────────────────────────────────────
interface FranchiseeOnboarded {
  type: 'FranchiseeOnboarded';
  data: {
    franchiseeId: string;
    franchisorId: string;
    brandId: string;
    prospectId?: string;
    entityName: string;
    entityType: string;
    primaryOwnerId: string;
    agreementId: string;
  };
}

interface FranchiseAgreementExecuted {
  type: 'FranchiseAgreementExecuted';
  data: {
    agreementId: string;
    franchiseeId: string;
    agreementNumber: string;
    royaltyRate: number;
    royaltyType: string;
    adFundRate: number;
    termYears: number;
    startDate: string;
    endDate: string;
    initialFee: number;
    tiers?: Array<{revenueFloor: number; revenueCeiling?: number; rate: number}>;
  };
}

interface FranchiseeStatusChanged {
  type: 'FranchiseeStatusChanged';
  data: {
    franchiseeId: string;
    fromStatus: string;
    toStatus: string;
    reason: string;
    changedBy: string;
  };
}

interface FranchiseeRiskScoreUpdated {
  type: 'FranchiseeRiskScoreUpdated';
  data: {
    franchiseeId: string;
    previousScore: number;
    newScore: number;
    riskFactors: string[];
    modelVersion: string;
  };
}

interface TerritoryAssigned {
  type: 'TerritoryAssigned';
  data: {
    territoryId: string;
    franchiseeId: string;
    territoryType: 'exclusive' | 'protected' | 'non_exclusive';
    boundaryGeoJSON: object;
    populationCount?: number;
    marketPotential?: number;
  };
}

interface LocationOpened {
  type: 'LocationOpened';
  data: {
    locationId: string;
    franchiseeId: string;
    name: string;
    storeNumber: string;
    address: object;
    coordinates: { latitude: number; longitude: number };
    openedDate: string;
  };
}
```

### Operations & Compliance Events

```typescript
// ── Audit Events ─────────────────────────────────────────────
interface AuditScheduled {
  type: 'AuditScheduled';
  data: {
    auditId: string;
    templateId: string;
    locationId: string;
    franchiseeId: string;
    auditorId: string;
    scheduledDate: string;
    auditType: string;
  };
}

interface AuditStarted {
  type: 'AuditStarted';
  data: {
    auditId: string;
    startedAt: string;
    auditorId: string;
  };
}

interface AuditResponseRecorded {
  type: 'AuditResponseRecorded';
  data: {
    auditId: string;
    templateItemId: string;
    responseValue: string;
    score: number;
    isNonCompliant: boolean;
    notes?: string;
    evidenceUrls?: string[];
  };
}

interface AuditCompleted {
  type: 'AuditCompleted';
  data: {
    auditId: string;
    totalScore: number;
    maxPossibleScore: number;
    scorePercentage: number;
    passed: boolean;
    completedAt: string;
    overallNotes?: string;
    nonCompliantItemCount: number;
  };
}

interface CorrectiveActionCreated {
  type: 'CorrectiveActionCreated';
  data: {
    actionId: string;
    auditId?: string;
    locationId: string;
    franchiseeId: string;
    title: string;
    description: string;
    severity: 'critical' | 'major' | 'minor' | 'observation';
    assignedTo: string;
    dueDate: string;
  };
}

interface CorrectiveActionResolved {
  type: 'CorrectiveActionResolved';
  data: {
    actionId: string;
    resolvedAt: string;
    resolutionNotes: string;
    evidenceUrls?: string[];
  };
}

interface CorrectiveActionEscalated {
  type: 'CorrectiveActionEscalated';
  data: {
    actionId: string;
    escalationLevel: number;
    escalatedTo: string;
    reason: string;
  };
}

// ── Document Events ──────────────────────────────────────────
interface OperationsDocumentPublished {
  type: 'OperationsDocumentPublished';
  data: {
    documentId: string;
    title: string;
    documentType: string;
    version: number;
    publishedBy: string;
    effectiveDate: string;
  };
}

interface DocumentAcknowledged {
  type: 'DocumentAcknowledged';
  data: {
    documentId: string;
    userId: string;
    franchiseeId: string;
    versionAcknowledged: number;
  };
}
```

### Training & LMS Events

```typescript
// ── Training Events ──────────────────────────────────────────
interface LearnerEnrolled {
  type: 'LearnerEnrolled';
  data: {
    enrollmentId: string;
    userId: string;
    courseId: string;
    franchiseeId: string;
    locationId?: string;
    enrolledBy: string;
    dueDate?: string;
    isMandatory: boolean;
  };
}

interface CourseStarted {
  type: 'CourseStarted';
  data: {
    enrollmentId: string;
    userId: string;
    courseId: string;
    startedAt: string;
  };
}

interface ScormDataUpdated {
  type: 'ScormDataUpdated';
  data: {
    enrollmentId: string;
    scoId: string;
    attemptNumber: number;
    completionStatus: string;
    successStatus: string;
    scoreRaw?: number;
    scoreScaled?: number;
    sessionTime?: string;
    totalTime?: string;
    suspendData?: string;
    location?: string;
    progressMeasure?: number;
  };
}

interface CourseCompleted {
  type: 'CourseCompleted';
  data: {
    enrollmentId: string;
    userId: string;
    courseId: string;
    completedAt: string;
    finalScore: number;
    passed: boolean;
    totalTimeSpentSeconds: number;
    attempts: number;
  };
}

interface CertificationIssued {
  type: 'CertificationIssued';
  data: {
    certificationId: string;
    userId: string;
    certificationName: string;
    issuedAt: string;
    expiresAt?: string;
    certificateNumber: string;
  };
}

interface CertificationExpired {
  type: 'CertificationExpired';
  data: {
    certificationId: string;
    userId: string;
    expiredAt: string;
  };
}
```

### Financial Events

```typescript
// ── Revenue & Royalty Events ─────────────────────────────────
interface RevenueReported {
  type: 'RevenueReported';
  data: {
    reportId: string;
    franchiseeId: string;
    locationId: string;
    reportingPeriod: string;
    periodType: 'weekly' | 'monthly';
    grossRevenue: number;
    netRevenue?: number;
    costOfGoods?: number;
    laborCost?: number;
    transactionCount?: number;
    averageTicket?: number;
    source: 'manual' | 'pos_import' | 'accounting_sync';
  };
}

interface RevenueReportVerified {
  type: 'RevenueReportVerified';
  data: {
    reportId: string;
    verifiedBy: string;
    adjustedGrossRevenue?: number;
    verificationNotes?: string;
  };
}

interface RoyaltyCalculated {
  type: 'RoyaltyCalculated';
  data: {
    calculationId: string;
    franchiseeId: string;
    agreementId: string;
    revenueReportId: string;
    billingPeriod: string;
    grossRevenue: number;
    applicableRevenue: number;
    royaltyRate: number;
    royaltyAmount: number;
    adFundAmount: number;
    technologyFee: number;
    totalDue: number;
    minimumApplied: boolean;
    tierApplied?: string;
  };
}

interface InvoiceGenerated {
  type: 'InvoiceGenerated';
  data: {
    invoiceId: string;
    franchiseeId: string;
    invoiceNumber: string;
    billingPeriod: string;
    issueDate: string;
    dueDate: string;
    lineItems: Array<{
      description: string;
      lineType: string;
      amount: number;
    }>;
    totalAmount: number;
    currency: string;
  };
}

interface PaymentReceived {
  type: 'PaymentReceived';
  data: {
    paymentId: string;
    franchiseeId: string;
    invoiceId: string;
    amount: number;
    paymentMethod: string;
    paymentReference?: string;
    processedAt: string;
  };
}

interface InvoiceOverdue {
  type: 'InvoiceOverdue';
  data: {
    invoiceId: string;
    franchiseeId: string;
    dueDate: string;
    amountDue: number;
    daysOverdue: number;
    lateFeeApplied: number;
  };
}
```

### Communication Events

```typescript
// ── Communication Events ─────────────────────────────────────
interface AnnouncementPublished {
  type: 'AnnouncementPublished';
  data: {
    announcementId: string;
    title: string;
    priority: string;
    targetAudience: string;
    targetBrands?: string[];
    requiresAcknowledgement: boolean;
    recipientCount: number;
  };
}

interface SupportTicketCreated {
  type: 'SupportTicketCreated';
  data: {
    ticketId: string;
    ticketNumber: string;
    createdBy: string;
    franchiseeId?: string;
    category: string;
    subject: string;
    priority: string;
  };
}

interface SupportTicketResolved {
  type: 'SupportTicketResolved';
  data: {
    ticketId: string;
    resolvedBy: string;
    resolutionTime: number;  // seconds from creation to resolution
    satisfactionRating?: number;
  };
}
```

---

## Read Model Projections (Query Side)

Each projection is a PostgreSQL table or view optimized for specific query patterns. Projections are rebuilt by replaying events from the event store.

```sql
-- ============================================================
-- READ MODEL PROJECTIONS
-- ============================================================

-- ── Franchisee Dashboard Projection ──────────────────────────
CREATE TABLE rm_franchisee_dashboard (
    franchisee_id       UUID PRIMARY KEY,
    franchisor_id       UUID NOT NULL,
    brand_id            UUID NOT NULL,
    entity_name         VARCHAR(500) NOT NULL,
    status              VARCHAR(30) NOT NULL,
    risk_score          NUMERIC(5,2),
    risk_factors        TEXT[],
    primary_owner_name  VARCHAR(200),
    primary_owner_email VARCHAR(320),
    location_count      INTEGER NOT NULL DEFAULT 0,
    open_location_count INTEGER NOT NULL DEFAULT 0,
    territory_count     INTEGER NOT NULL DEFAULT 0,
    onboarding_started  DATE,
    opening_date        DATE,
    agreement_end_date  DATE,
    royalty_rate         NUMERIC(5,3),
    -- Latest period financials
    latest_period_revenue NUMERIC(14,2),
    latest_period_royalty NUMERIC(12,2),
    ytd_revenue          NUMERIC(14,2),
    ytd_royalty_paid     NUMERIC(12,2),
    outstanding_balance  NUMERIC(12,2),
    -- Compliance summary
    last_audit_date     DATE,
    last_audit_score    NUMERIC(5,2),
    open_corrective_actions INTEGER NOT NULL DEFAULT 0,
    overdue_corrective_actions INTEGER NOT NULL DEFAULT 0,
    -- Training summary
    active_enrollments  INTEGER NOT NULL DEFAULT 0,
    overdue_trainings   INTEGER NOT NULL DEFAULT 0,
    active_certifications INTEGER NOT NULL DEFAULT 0,
    expiring_certifications INTEGER NOT NULL DEFAULT 0,
    -- Metadata
    last_updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_event_position BIGINT NOT NULL DEFAULT 0
);

CREATE INDEX idx_rm_franchisee_franchisor ON rm_franchisee_dashboard (franchisor_id, status);
CREATE INDEX idx_rm_franchisee_risk ON rm_franchisee_dashboard (franchisor_id, risk_score DESC);

-- ── Location Performance Projection ─────────────────────────
CREATE TABLE rm_location_performance (
    location_id         UUID PRIMARY KEY,
    franchisee_id       UUID NOT NULL,
    franchisor_id       UUID NOT NULL,
    name                VARCHAR(255) NOT NULL,
    store_number        VARCHAR(50),
    status              VARCHAR(20) NOT NULL,
    city                VARCHAR(100),
    state_province      VARCHAR(100),
    country             VARCHAR(3),
    -- Current period
    current_period      DATE,
    current_revenue     NUMERIC(14,2),
    current_transactions INTEGER,
    current_avg_ticket  NUMERIC(10,2),
    current_labor_pct   NUMERIC(5,2),
    current_cogs_pct    NUMERIC(5,2),
    -- Comparisons
    prior_period_revenue NUMERIC(14,2),
    prior_year_revenue  NUMERIC(14,2),
    same_store_growth_pct NUMERIC(7,2),
    -- Benchmarking
    network_avg_revenue NUMERIC(14,2),
    network_rank        INTEGER,
    quartile            INTEGER,  -- 1=top, 4=bottom
    -- Compliance
    last_audit_score    NUMERIC(5,2),
    compliance_status   VARCHAR(20),
    last_updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_location_franchisor ON rm_location_performance (franchisor_id, status);

-- ── Audit Compliance Projection ──────────────────────────────
CREATE TABLE rm_compliance_overview (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL,
    location_id         UUID NOT NULL,
    franchisee_id       UUID NOT NULL,
    audit_id            UUID NOT NULL,
    audit_type          VARCHAR(50) NOT NULL,
    audit_date          DATE NOT NULL,
    score_percentage    NUMERIC(5,2),
    passed              BOOLEAN,
    non_compliant_items INTEGER NOT NULL DEFAULT 0,
    critical_failures   INTEGER NOT NULL DEFAULT 0,
    open_corrective_actions INTEGER NOT NULL DEFAULT 0,
    overdue_corrective_actions INTEGER NOT NULL DEFAULT 0,
    days_since_last_audit INTEGER,
    trend_direction     VARCHAR(10),  -- 'improving', 'stable', 'declining'
    last_updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_compliance_location ON rm_compliance_overview (franchisor_id, location_id, audit_date DESC);

-- ── Royalty & Financial Projection ───────────────────────────
CREATE TABLE rm_financial_summary (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL,
    franchisee_id       UUID NOT NULL,
    billing_period      DATE NOT NULL,
    total_locations     INTEGER NOT NULL,
    total_gross_revenue NUMERIC(14,2) NOT NULL,
    total_royalty_due   NUMERIC(12,2) NOT NULL,
    total_ad_fund_due   NUMERIC(12,2),
    total_fees_due      NUMERIC(12,2),
    total_invoiced      NUMERIC(12,2) NOT NULL,
    total_paid          NUMERIC(12,2) NOT NULL,
    balance_outstanding NUMERIC(12,2) NOT NULL,
    invoices_overdue    INTEGER NOT NULL DEFAULT 0,
    avg_days_to_pay     NUMERIC(5,1),
    payment_compliance  VARCHAR(20),  -- 'current', 'late', 'delinquent'
    last_updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (franchisor_id, franchisee_id, billing_period)
);

-- ── Training Completion Projection ───────────────────────────
CREATE TABLE rm_training_dashboard (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL,
    franchisee_id       UUID NOT NULL,
    location_id         UUID,
    user_id             UUID NOT NULL,
    user_name           VARCHAR(200) NOT NULL,
    user_role           VARCHAR(50),
    total_assigned      INTEGER NOT NULL DEFAULT 0,
    total_completed     INTEGER NOT NULL DEFAULT 0,
    total_overdue       INTEGER NOT NULL DEFAULT 0,
    total_in_progress   INTEGER NOT NULL DEFAULT 0,
    completion_rate     NUMERIC(5,2),
    avg_score           NUMERIC(5,2),
    active_certifications INTEGER NOT NULL DEFAULT 0,
    expired_certifications INTEGER NOT NULL DEFAULT 0,
    next_expiration_date DATE,
    last_activity_at    TIMESTAMPTZ,
    last_updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_training_location ON rm_training_dashboard (franchisor_id, location_id);

-- ── Network-wide KPI Benchmarking Projection ─────────────────
CREATE TABLE rm_network_benchmarks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL,
    kpi_code            VARCHAR(100) NOT NULL,
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    location_count      INTEGER NOT NULL,
    network_average     NUMERIC(14,4),
    network_median      NUMERIC(14,4),
    top_quartile_threshold NUMERIC(14,4),
    bottom_quartile_threshold NUMERIC(14,4),
    network_min         NUMERIC(14,4),
    network_max         NUMERIC(14,4),
    std_deviation       NUMERIC(14,4),
    prior_period_avg    NUMERIC(14,4),
    change_pct          NUMERIC(7,2),
    last_updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (franchisor_id, kpi_code, period_start)
);

-- ── Anomaly Feed Projection ─────────────────────────────────
CREATE TABLE rm_anomaly_feed (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    franchisor_id       UUID NOT NULL,
    location_id         UUID NOT NULL,
    franchisee_id       UUID NOT NULL,
    location_name       VARCHAR(255),
    franchisee_name     VARCHAR(500),
    anomaly_type        VARCHAR(50) NOT NULL,
    severity            VARCHAR(20) NOT NULL,
    metric_name         VARCHAR(255) NOT NULL,
    expected_value      NUMERIC(14,4),
    actual_value        NUMERIC(14,4),
    deviation_pct       NUMERIC(7,2),
    description         TEXT,
    ai_recommendation   TEXT,
    acknowledged        BOOLEAN NOT NULL DEFAULT FALSE,
    detected_at         TIMESTAMPTZ NOT NULL,
    last_updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_anomaly_franchisor ON rm_anomaly_feed (franchisor_id, severity, acknowledged, detected_at DESC);

-- ── Prospect Pipeline Projection ────────────────────────────
CREATE TABLE rm_prospect_pipeline (
    prospect_id         UUID PRIMARY KEY,
    franchisor_id       UUID NOT NULL,
    brand_id            UUID,
    full_name           VARCHAR(200) NOT NULL,
    email               VARCHAR(320),
    source              VARCHAR(100),
    pipeline_stage      VARCHAR(50) NOT NULL,
    lead_score          NUMERIC(5,2),
    days_in_current_stage INTEGER,
    total_activities    INTEGER NOT NULL DEFAULT 0,
    last_activity_date  DATE,
    assigned_to_name    VARCHAR(200),
    created_at          TIMESTAMPTZ NOT NULL,
    last_updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_pipeline_franchisor ON rm_prospect_pipeline (franchisor_id, pipeline_stage);
```

---

## Projection Builder Architecture

```
                    ┌──────────────────┐
                    │  Command Service │
                    │ (validate+store) │
                    └────────┬─────────┘
                             │ append event
                             ▼
                    ┌──────────────────┐
                    │   Event Store    │
                    │  (append-only)   │
                    └────────┬─────────┘
                             │ event published
                             ▼
                    ┌──────────────────┐
                    │  Message Broker  │
                    │ (Kafka / NATS)   │
                    └──┬───┬───┬───┬───┘
                       │   │   │   │
          ┌────────────┘   │   │   └────────────┐
          ▼                ▼   ▼                ▼
   ┌─────────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐
   │ Franchisee  │  │ Compliance│  │ Financial│  │  Training │
   │ Dashboard   │  │ Overview  │  │ Summary  │  │ Dashboard │
   │ Projector   │  │ Projector │  │ Projector│  │ Projector │
   └──────┬──────┘  └─────┬────┘  └─────┬────┘  └─────┬─────┘
          │               │             │              │
          ▼               ▼             ▼              ▼
   ┌──────────────────────────────────────────────────────────┐
   │               Read Database (PostgreSQL)                 │
   │         rm_franchisee_dashboard, rm_compliance_overview, │
   │         rm_financial_summary, rm_training_dashboard, ... │
   └──────────────────────────────────────────────────────────┘
```

### Projection Rebuild Process

When a projection schema changes or data needs correction, the projection is rebuilt from the event store:

```sql
-- 1. Mark projection as rebuilding
UPDATE projection_checkpoints
SET status = 'rebuilding', last_global_position = 0
WHERE projection_name = 'franchisee_dashboard';

-- 2. Truncate the read model table
TRUNCATE rm_franchisee_dashboard;

-- 3. Replay all events from position 0
-- (handled by the projection builder service)

-- 4. Once caught up, mark as running
UPDATE projection_checkpoints
SET status = 'running'
WHERE projection_name = 'franchisee_dashboard';
```

---

## Command Examples

```typescript
// ── Command: Submit Revenue Report ───────────────────────────
interface SubmitRevenueReport {
  type: 'SubmitRevenueReport';
  payload: {
    franchiseeId: string;
    locationId: string;
    reportingPeriod: string;
    grossRevenue: number;
    netRevenue?: number;
    costOfGoods?: number;
    laborCost?: number;
  };
}

// Command handler validates:
// 1. Location belongs to franchisee
// 2. No duplicate report for same period
// 3. Revenue figures are non-negative
// 4. Reporting period is not in the future
// Then emits: RevenueReported event

// ── Command: Complete Audit ──────────────────────────────────
interface CompleteAudit {
  type: 'CompleteAudit';
  payload: {
    auditId: string;
    responses: Array<{
      templateItemId: string;
      responseValue: string;
      score: number;
      notes?: string;
      evidenceUrls?: string[];
    }>;
    overallNotes?: string;
  };
}

// Command handler validates:
// 1. Audit exists and is in 'in_progress' status
// 2. All required template items have responses
// 3. All critical items that require photos have evidence
// Then emits: AuditResponseRecorded (per item), AuditCompleted
// May also emit: CorrectiveActionCreated (for non-compliant items)

// ── Process Manager: Royalty Calculation Saga ────────────────
// Triggered by: RevenueReported
// 1. Load franchise agreement for the franchisee
// 2. Determine applicable royalty rate (flat or tiered)
// 3. Calculate royalty, ad fund, and technology fees
// 4. Emit: RoyaltyCalculated
// 5. Generate invoice
// 6. Emit: InvoiceGenerated
// 7. Schedule payment reminder if auto-debit not enabled
```

---

## Pros and Cons

### Pros

1. **Complete audit trail by design.** Every state change is permanently recorded as an immutable event. For franchise compliance, this means regulators, auditors, and legal teams can trace exactly what happened, when, and by whom. There is no need for a separate audit_log table -- the event store IS the audit log.

2. **Temporal queries are trivial.** "What was Franchisee X's status on January 15?" is answered by replaying events up to that date. "When did Location Y's audit score drop below 80%?" is a simple event stream query. Traditional CRUD systems require complex change-tracking mechanisms to answer these questions.

3. **Decoupled read and write scaling.** The command side (processing revenue reports, recording audit responses) scales independently from the query side (rendering dashboards, generating reports). During end-of-month reporting peaks when thousands of franchisees submit revenue reports simultaneously, the write side absorbs the load without degrading dashboard performance.

4. **Projection flexibility.** New read models can be created at any time by replaying the full event history. Need a new "franchisee health score" dashboard that combines financial, compliance, and training metrics? Build a new projector, replay events, and the new view is populated from day one without modifying any existing code.

5. **Natural fit for saga/process managers.** The royalty calculation flow (revenue report -> royalty calculation -> invoice generation -> payment tracking) is naturally modeled as a process manager that reacts to events, rather than a procedural script that orchestrates multiple service calls.

6. **Event-driven integrations.** External systems (accounting software, POS systems, HR platforms) subscribe to relevant events rather than polling APIs. QuickBooks integration subscribes to InvoiceGenerated and PaymentReceived; the POS system publishes RevenueReported.

7. **AI/ML pipeline friendly.** The event stream provides a rich, timestamped dataset for training predictive models (franchisee risk scoring, anomaly detection, lead scoring). No ETL pipeline is needed -- the event store is already structured as a time-ordered log.

### Cons

1. **Significant architectural complexity.** The system requires an event store, message broker, multiple projection builders, a dead letter queue, checkpoint management, and snapshot handling. A small team building an MVP will spend substantial engineering time on infrastructure before delivering any franchise management features.

2. **Eventual consistency in read models.** After a franchisee submits a revenue report, the financial summary projection may take milliseconds to seconds to update. Users accustomed to immediate consistency ("I just submitted, why isn't my dashboard updated?") will need UI accommodations like optimistic updates or "processing" indicators.

3. **Event schema evolution is hard.** When a FranchiseeOnboarded event v1 gains new required fields in v2, all projection builders must handle both versions. Over years of operation, event upcasters accumulate, and the mapping between old and new event shapes becomes a maintenance burden.

4. **Debugging is non-trivial.** When a franchisee dashboard shows incorrect data, the investigation path is: check the projection table, then trace back to the events that built it, then check the projection builder logic. This three-layer debugging is more complex than querying a single source-of-truth table.

5. **SCORM tracking overhead.** SCORM's communication pattern involves frequent micro-updates (location, suspend_data, session_time) during a learning session. Each update as a separate event creates high event volume with low informational density. Batching or throttling SCORM updates is necessary but adds complexity.

6. **Storage growth.** Events are never deleted. A franchise network with 1,000 locations submitting weekly revenue reports, daily SCORM interactions for 10,000 learners, and monthly audits will generate millions of events per year. Archival and compaction strategies are needed.

7. **Team skill requirements.** Event sourcing is unfamiliar to most developers. Hiring, training, and retaining engineers who can reason about event streams, projections, and sagas is harder than finding developers comfortable with CRUD/ORM patterns.

---

## Migration and Scaling Considerations

### Event Store Partitioning
- Partition the `events` table by `occurred_at` (monthly or quarterly ranges)
- Archive old partitions to cold storage (S3/GCS) while keeping recent partitions on fast SSDs
- Snapshots reduce replay time: snapshot every 100 events per aggregate

### Projection Scaling
- Each projection builder runs as an independent service
- Projections can be rebuilt independently without affecting other read models
- Deploy projection builders close to their read database for low latency
- Use consumer groups in Kafka/NATS for parallel event processing within a projection

### Multi-Region Deployment
- Event store is the single source of truth, deployed in the primary region
- Read model databases are replicated to regional read replicas
- For data sovereignty: route commands to the regional event store, sync events across regions with conflict resolution

### Migration from CRUD
If migrating from a traditional CRUD system:
1. Implement dual-write: write to both CRUD tables and event store
2. Build projections from events, compare with CRUD state for validation
3. Once projections match, switch read traffic to projections
4. Retire CRUD writes, keeping the event store as the sole write path
5. Phase the migration domain by domain (start with financials, then compliance, then training)

### Compaction Strategy
- **Hot tier** (0-12 months): Full event detail on SSD-backed storage
- **Warm tier** (1-3 years): Events on standard storage, snapshots cached
- **Cold tier** (3+ years): Events archived to object storage, snapshots retained in database for fast aggregate loading
- Consider event compaction for high-volume, low-value streams (e.g., SCORM session heartbeats) -- merge 100 ScormDataUpdated events into a single ScormSessionSummary event

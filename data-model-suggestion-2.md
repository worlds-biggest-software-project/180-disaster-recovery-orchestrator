# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Disaster Recovery Orchestrator · Created: 2026-05-20

## Philosophy

This model uses event sourcing as the foundational persistence strategy: every state change in the system — a DR plan being created, a runbook step completing, a failover being triggered, an asset's health status changing — is recorded as an immutable event in an append-only event store. Current state is derived by replaying events, and read-optimised materialised views (projections) serve query workloads. This is the CQRS (Command Query Responsibility Segregation) pattern.

The design is motivated by the core requirement of DR orchestration: a complete, tamper-proof audit trail. Regulatory frameworks (DORA, SOC 2, ISO 22301) demand evidence of every action taken during recovery operations. Event sourcing provides this natively — the audit trail is not a secondary concern bolted onto a CRUD schema; it IS the schema. Every state transition is captured with full context, enabling temporal queries ("what was the DR plan configuration at the exact moment the failover was triggered?") and forensic reconstruction of incidents.

This approach draws from how financial trading systems and healthcare EHRs handle audit requirements. Microsoft's Azure Architecture Center recommends event sourcing for systems where "a complete audit trail of changes is required" and "the system needs to be able to reconstruct past states." For a DR orchestrator handling life-or-death infrastructure operations under regulatory scrutiny, this is a natural fit.

**Best for:** Organisations with strict audit and compliance requirements (financial services, healthcare, government) where tamper-proof event history and temporal state reconstruction are essential.

**Trade-offs:**
- Pro: Complete, immutable audit trail is the primary data — not a secondary concern
- Pro: Temporal queries ("what was true at time X?") are native operations
- Pro: Event replay enables debugging, forensic analysis, and state reconstruction
- Pro: Natural fit for CloudEvents-based integration with observability pipelines
- Pro: Write performance scales well — append-only writes are fast
- Con: Read complexity — queries require materialised views that must be kept in sync
- Con: Event schema evolution requires careful versioning (upcasting old events)
- Con: Higher storage requirements — events are never deleted, only compacted
- Con: Steeper learning curve for developers unfamiliar with event sourcing
- Con: Eventual consistency between event store and projections requires careful handling

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 22301:2019 | Every DR plan change, test execution, and approval is an immutable event — perfect alignment with evidence retention requirements |
| DORA (EU 2022/2554) | Event store provides tamper-proof evidence of recovery actions within two-hour RTO windows |
| SOC 2 Availability TSC | Event stream IS the audit trail — auditors can replay any sequence of actions |
| NIST SP 800-34 | Plan lifecycle events map to the seven-step contingency planning process with full traceability |
| CloudEvents 1.0 | Event envelope structure directly aligns with CloudEvents specification — events can be published to external systems verbatim |
| NIS2 (EU 2022/2555) | Immutable event log satisfies DR test documentation and backup verification evidence requirements |
| AsyncAPI 2.x/3.x | Event schemas can be published as AsyncAPI specifications for external consumers |

---

## Event Store (Source of Truth)

```sql
-- The single source of truth: an append-only event store
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL, -- 'DrPlan','Asset','FailoverExecution','Environment','User','HealthCheck'
    stream_id       UUID NOT NULL, -- the aggregate/entity this event belongs to
    event_type      VARCHAR(200) NOT NULL, -- 'DrPlanCreated','RunbookStepCompleted','FailoverInitiated', etc.
    event_version   INTEGER NOT NULL, -- version within the stream (optimistic concurrency)
    data            JSONB NOT NULL, -- the event payload
    metadata        JSONB NOT NULL DEFAULT '{}', -- correlation_id, causation_id, user_id, ip_address
    -- CloudEvents-compatible fields
    ce_specversion  VARCHAR(10) NOT NULL DEFAULT '1.0',
    ce_source       VARCHAR(500) NOT NULL, -- e.g. '/dr-orchestrator/plans'
    ce_type         VARCHAR(200) NOT NULL, -- same as event_type but namespaced: 'com.dro.plan.created'
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, event_version) -- optimistic concurrency control
);

-- Primary query pattern: load all events for a specific aggregate
CREATE INDEX idx_events_stream ON events (stream_type, stream_id, event_version ASC);

-- Tenant isolation for event queries
CREATE INDEX idx_events_tenant ON events (tenant_id, created_at DESC);

-- Event type queries (e.g., "all failover events across all plans")
CREATE INDEX idx_events_type ON events (tenant_id, event_type, created_at DESC);

-- Time-based queries for compliance evidence windows
CREATE INDEX idx_events_time ON events (tenant_id, ce_time DESC);

-- Partition by month for performance at scale
-- CREATE TABLE events_2026_05 PARTITION OF events
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Example events stored in the data column:
--
-- DrPlanCreated:
-- {
--   "plan_name": "Production Full Failover",
--   "source_environment_id": "...",
--   "target_environment_id": "...",
--   "target_rto_seconds": 3600,
--   "target_rpo_seconds": 300,
--   "created_by": "user-uuid"
-- }
--
-- RunbookStepAdded:
-- {
--   "step_name": "Failover Primary Database",
--   "step_type": "automated",
--   "action_type": "failover_vm",
--   "step_order": 3,
--   "target_asset_id": "asset-uuid",
--   "timeout_seconds": 600
-- }
--
-- FailoverInitiated:
-- {
--   "plan_id": "plan-uuid",
--   "execution_type": "test",
--   "trigger_type": "scheduled",
--   "initiated_by": "user-uuid"
-- }
--
-- AssetHealthChanged:
-- {
--   "asset_id": "asset-uuid",
--   "previous_status": "healthy",
--   "new_status": "degraded",
--   "check_type": "replication_lag",
--   "measured_value": 45,
--   "threshold": 30
-- }
```

## Snapshots (Performance Optimisation)

```sql
-- Periodic snapshots of aggregate state to avoid replaying all events
CREATE TABLE snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    stream_id       UUID NOT NULL,
    event_version   INTEGER NOT NULL, -- snapshot taken at this event version
    state           JSONB NOT NULL, -- serialised aggregate state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, event_version)
);

CREATE INDEX idx_snapshots_stream ON snapshots (stream_type, stream_id, event_version DESC);

-- To rebuild state: load latest snapshot, then replay events after snapshot version
-- SELECT state FROM snapshots
--     WHERE stream_type = 'DrPlan' AND stream_id = $1
--     ORDER BY event_version DESC LIMIT 1;
-- SELECT data FROM events
--     WHERE stream_type = 'DrPlan' AND stream_id = $1 AND event_version > $snapshot_version
--     ORDER BY event_version ASC;
```

## Read Model Projections (Materialised Views)

```sql
-- ============================================================
-- PROJECTION: Current DR Plan State
-- Materialised from: DrPlanCreated, DrPlanUpdated, DrPlanApproved,
--                    RunbookStepAdded, RunbookStepRemoved, etc.
-- ============================================================
CREATE TABLE proj_dr_plans (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    plan_type       VARCHAR(50) NOT NULL,
    source_environment_id UUID NOT NULL,
    target_environment_id UUID NOT NULL,
    target_rto_seconds INTEGER NOT NULL,
    target_rpo_seconds INTEGER NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    version         INTEGER NOT NULL DEFAULT 1,
    step_count      INTEGER NOT NULL DEFAULT 0,
    asset_count     INTEGER NOT NULL DEFAULT 0,
    approved_by     UUID,
    approved_at     TIMESTAMPTZ,
    created_by      UUID NOT NULL,
    last_tested_at  TIMESTAMPTZ,
    last_test_result VARCHAR(50), -- 'passed','failed','partial'
    last_event_version INTEGER NOT NULL, -- tracks projection currency
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_plans_tenant ON proj_dr_plans (tenant_id);
CREATE INDEX idx_proj_plans_status ON proj_dr_plans (tenant_id, status);

-- ============================================================
-- PROJECTION: Current Asset Inventory
-- Materialised from: AssetDiscovered, AssetUpdated, AssetDecommissioned,
--                    DependencyAdded, DependencyRemoved, HealthChanged
-- ============================================================
CREATE TABLE proj_assets (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    environment_id  UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    asset_type      VARCHAR(50) NOT NULL,
    platform        VARCHAR(100),
    hostname        VARCHAR(255),
    ip_address      INET,
    tier            VARCHAR(50) NOT NULL DEFAULT 'standard',
    rto_seconds     INTEGER,
    rpo_seconds     INTEGER,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    current_health  VARCHAR(50) DEFAULT 'unknown',
    replication_status VARCHAR(50),
    current_rpo_seconds INTEGER, -- measured current RPO
    dependency_count_upstream INTEGER DEFAULT 0,
    dependency_count_downstream INTEGER DEFAULT 0,
    metadata        JSONB NOT NULL DEFAULT '{}',
    last_event_version INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_assets_tenant ON proj_assets (tenant_id);
CREATE INDEX idx_proj_assets_env ON proj_assets (environment_id);
CREATE INDEX idx_proj_assets_health ON proj_assets (tenant_id, current_health);

-- ============================================================
-- PROJECTION: Current Failover Execution Status
-- Materialised from: FailoverInitiated, StepStarted, StepCompleted,
--                    StepFailed, FailoverCompleted, FailoverFailed
-- ============================================================
CREATE TABLE proj_failover_executions (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    plan_id         UUID NOT NULL,
    plan_name       VARCHAR(255), -- denormalised for display
    execution_type  VARCHAR(50) NOT NULL,
    trigger_type    VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    initiated_by    UUID,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    actual_rto_seconds INTEGER,
    actual_rpo_seconds INTEGER,
    rto_met         BOOLEAN,
    rpo_met         BOOLEAN,
    total_steps     INTEGER DEFAULT 0,
    completed_steps INTEGER DEFAULT 0,
    failed_steps    INTEGER DEFAULT 0,
    current_step_name VARCHAR(255),
    failure_reason  TEXT,
    last_event_version INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_failover_tenant ON proj_failover_executions (tenant_id, status);
CREATE INDEX idx_proj_failover_started ON proj_failover_executions (started_at DESC);

-- ============================================================
-- PROJECTION: Compliance Dashboard
-- Materialised from: FailoverCompleted, ComplianceEvidenceGenerated,
--                    ComplianceEvidenceApproved
-- ============================================================
CREATE TABLE proj_compliance_status (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    framework_code  VARCHAR(50) NOT NULL, -- 'iso_22301','soc2','dora','nis2'
    requirement_code VARCHAR(50) NOT NULL,
    requirement_title VARCHAR(255) NOT NULL,
    status          VARCHAR(50) NOT NULL, -- 'compliant','at_risk','non_compliant','not_applicable'
    last_evidence_date DATE,
    next_required_date DATE,
    days_until_due  INTEGER,
    evidence_count  INTEGER DEFAULT 0,
    last_event_version INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_compliance_tenant ON proj_compliance_status (tenant_id, framework_code);
CREATE INDEX idx_proj_compliance_status ON proj_compliance_status (tenant_id, status);

-- ============================================================
-- PROJECTION: Asset Dependency Graph
-- Materialised from: DependencyAdded, DependencyRemoved,
--                    DependencyConfidenceUpdated
-- ============================================================
CREATE TABLE proj_asset_dependencies (
    upstream_asset_id   UUID NOT NULL,
    downstream_asset_id UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    dependency_type VARCHAR(50) NOT NULL,
    discovery_method VARCHAR(50) NOT NULL,
    confidence_score NUMERIC(3,2),
    created_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (upstream_asset_id, downstream_asset_id)
);

CREATE INDEX idx_proj_deps_downstream ON proj_asset_dependencies (downstream_asset_id);
CREATE INDEX idx_proj_deps_tenant ON proj_asset_dependencies (tenant_id);

-- ============================================================
-- PROJECTION: RTO/RPO Gap Analysis (time-series style)
-- Materialised from: FailoverCompleted, HealthCheckRecorded
-- ============================================================
CREATE TABLE proj_rto_rpo_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    plan_id         UUID,
    asset_id        UUID,
    measurement_type VARCHAR(50) NOT NULL, -- 'rto_actual','rpo_actual','rpo_current'
    target_seconds  INTEGER NOT NULL,
    actual_seconds  INTEGER NOT NULL,
    gap_seconds     INTEGER GENERATED ALWAYS AS (actual_seconds - target_seconds) STORED,
    met_target      BOOLEAN GENERATED ALWAYS AS (actual_seconds <= target_seconds) STORED,
    measured_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rto_rpo_history_plan ON proj_rto_rpo_history (tenant_id, plan_id, measured_at DESC);
CREATE INDEX idx_rto_rpo_history_asset ON proj_rto_rpo_history (tenant_id, asset_id, measured_at DESC);
```

## Projection Management

```sql
-- Tracks the last processed event for each projection (catch-up subscription)
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY, -- 'proj_dr_plans','proj_assets', etc.
    last_event_id   UUID NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for events that failed projection processing
CREATE TABLE projection_errors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(100) NOT NULL,
    event_id        UUID NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    max_retries     INTEGER NOT NULL DEFAULT 3,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending', -- 'pending','retrying','failed','resolved'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_proj_errors_status ON projection_errors (projection_name, status);
```

## Identity & Multi-Tenancy (Minimal Relational)

```sql
-- These tables are NOT event-sourced — they are operational CRUD tables
-- for authentication and session management only.

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID REFERENCES users(id),
    key_hash        VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    scopes          JSONB NOT NULL DEFAULT '[]', -- ['plan.read','failover.execute']
    expires_at      TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Example Queries

```sql
-- Reconstruct DR plan state at a specific point in time
SELECT data FROM events
WHERE stream_type = 'DrPlan'
  AND stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND ce_time <= '2026-03-15T14:30:00Z'
ORDER BY event_version ASC;

-- All failover-related events in the last 24 hours across a tenant
SELECT ce_type, ce_time, data, metadata
FROM events
WHERE tenant_id = '...'
  AND event_type LIKE 'Failover%'
  AND ce_time >= now() - INTERVAL '24 hours'
ORDER BY ce_time DESC;

-- Compliance evidence: all events for a specific failover execution
SELECT ce_type, ce_time, data
FROM events
WHERE stream_type = 'FailoverExecution'
  AND stream_id = '...'
ORDER BY event_version ASC;

-- RTO/RPO gap trend over the last 6 months
SELECT date_trunc('week', measured_at) AS week,
       AVG(gap_seconds) AS avg_gap,
       COUNT(*) FILTER (WHERE met_target) AS targets_met,
       COUNT(*) AS total_measurements
FROM proj_rto_rpo_history
WHERE tenant_id = '...' AND plan_id = '...'
  AND measured_at >= now() - INTERVAL '6 months'
GROUP BY week
ORDER BY week;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single events table (partitioned by month) |
| Snapshots | 1 | Periodic aggregate state snapshots |
| Projections (Read Models) | 7 | proj_dr_plans, proj_assets, proj_failover_executions, proj_compliance_status, proj_asset_dependencies, proj_rto_rpo_history, plus checkpoints |
| Projection Infrastructure | 2 | projection_checkpoints, projection_errors |
| Identity (CRUD) | 3 | tenants, users, api_keys |
| **Total** | **14** | Plus partition tables for events |

---

## Key Design Decisions

1. **Single events table as source of truth** — all domain state lives in one append-only table. This eliminates the "audit log drift" problem where a separate audit table can get out of sync with the operational tables. The events table IS the operational data.

2. **CloudEvents-compatible event envelope** — every event includes `ce_specversion`, `ce_source`, `ce_type`, and `ce_time` fields, making events directly publishable to CloudEvents-compatible systems (AWS EventBridge, Azure Event Grid, Kafka) without transformation.

3. **Optimistic concurrency via stream versioning** — the UNIQUE constraint on `(stream_type, stream_id, event_version)` prevents concurrent conflicting writes to the same aggregate. Writers must load current version before appending.

4. **Projections are disposable and rebuildable** — every `proj_*` table can be dropped and rebuilt by replaying events from the event store. The `projection_checkpoints` table tracks where each projection left off for catch-up processing.

5. **Snapshots prevent unbounded replay** — for aggregates with hundreds of events (e.g., a heavily-modified DR plan), periodic snapshots avoid replaying the entire event history. State reconstruction loads the latest snapshot then replays only subsequent events.

6. **Identity tables are NOT event-sourced** — user authentication and tenant management use traditional CRUD because they don't benefit from event sourcing (no temporal queries needed on user passwords), and mixing auth with event sourcing adds unnecessary complexity.

7. **Generated columns for gap analysis** — `proj_rto_rpo_history` uses PostgreSQL GENERATED ALWAYS AS columns for `gap_seconds` and `met_target`, keeping the calculation consistent and indexed without application logic.

8. **Event schema evolution handled by versioning** — the `metadata` JSONB field includes an `event_schema_version` key. When event structures change, projections include upcasting logic to transform old event formats to current structure during replay.

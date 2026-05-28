# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Disaster Recovery Orchestrator · Created: 2026-05-20

## Philosophy

This model follows a traditional normalized relational approach where every concept — organisations, infrastructure assets, DR plans, runbook steps, dependency relationships, test executions, compliance evidence — receives its own dedicated table with strict foreign key constraints. The schema enforces referential integrity at the database level, making it impossible to create orphaned records or invalid dependency references.

The design draws from how enterprise DR orchestrators like Veeam Recovery Orchestrator and Commvault structure their internal data: discrete entities for protection groups, recovery plans, plan steps, and test results, all linked by foreign keys. It aligns with ISO 22301's emphasis on documented, traceable recovery procedures where every element can be independently queried and audited.

This approach is best for teams that prioritise data integrity, complex cross-entity reporting (e.g., "show all untested runbooks across all tenants that protect assets in EU jurisdictions"), and regulatory environments where auditors need to query individual evidence records. It trades write simplicity for query power and constraint enforcement.

**Best for:** Regulated enterprises needing strong data integrity, complex cross-entity queries, and audit-friendly schema with clear foreign key trails.

**Trade-offs:**
- Pro: Maximum referential integrity — the database itself prevents invalid states
- Pro: Standard SQL queries work without JSONB operators or event replay
- Pro: Well-understood by DBAs and ORM frameworks; easy to build CRUD APIs
- Pro: Auditors can inspect individual tables directly
- Con: High table count (~45-50 tables) increases migration complexity
- Con: Schema changes require ALTER TABLE migrations for every new field
- Con: Jurisdiction-specific or asset-type-specific fields lead to sparse columns or additional junction tables
- Con: Write-heavy operations (telemetry ingestion, event logging) may bottleneck on foreign key constraint checks

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 22301:2019 | DR plan structure (plan → steps → dependencies) mirrors the standard's requirement for documented, testable recovery procedures |
| ISO/IEC 27001:2022 | Audit trail tables align with Annex A.17 evidence requirements for information security continuity |
| NIST SP 800-34 | Seven-step contingency planning process maps to plan lifecycle states (draft → approved → tested → maintained) |
| DORA (EU 2022/2554) | Dedicated compliance_evidence and test_execution tables support two-hour RTO monitoring and annual testing evidence |
| NIS2 (EU 2022/2555) | Backup verification and DR test documentation tables align with ENISA technical guidance requirements |
| SOC 2 Availability TSC | Test result capture with timestamps and approver sign-off directly serves SOC 2 audit evidence needs |
| CloudEvents 1.0 | Event log table schema aligns with CloudEvents envelope structure (specversion, type, source, id, time, data) |
| OpenAPI 3.1 | Normalized entities map cleanly to REST resources with standard CRUD endpoints |
| OAuth 2.0 / OIDC | Dedicated auth tables for API credentials, tokens, and RBAC roles |

---

## Core Identity & Multi-Tenancy

```sql
-- Tenant / Organisation
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard', -- 'free','standard','enterprise'
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants (slug);

-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255), -- null for SSO-only users
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local', -- 'local','google','azure_ad','okta'
    auth_provider_id VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users (tenant_id);
CREATE INDEX idx_users_email ON users (email);

-- Roles and Permissions (RBAC)
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT false, -- system roles cannot be deleted
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE, -- e.g. 'plan.create', 'failover.execute', 'test.run'
    description     TEXT,
    category        VARCHAR(50) NOT NULL -- 'plan','failover','test','compliance','admin'
);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);
```

## Infrastructure Assets

```sql
-- Environments (production, DR site, staging, etc.)
CREATE TABLE environments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    environment_type VARCHAR(50) NOT NULL, -- 'production','dr_target','staging','cleanroom'
    cloud_provider  VARCHAR(50), -- 'aws','azure','gcp','on_premises','hybrid'
    region          VARCHAR(100), -- e.g. 'us-east-1', 'westeurope'
    jurisdiction    VARCHAR(10), -- ISO 3166-1 alpha-2 country code for data residency
    status          VARCHAR(50) NOT NULL DEFAULT 'active', -- 'active','degraded','offline','decommissioned'
    connection_config JSONB, -- encrypted connection details for the environment
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_environments_tenant ON environments (tenant_id);
CREATE INDEX idx_environments_type ON environments (tenant_id, environment_type);

-- Infrastructure Assets (VMs, databases, services, containers)
CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    environment_id  UUID NOT NULL REFERENCES environments(id),
    name            VARCHAR(255) NOT NULL,
    asset_type      VARCHAR(50) NOT NULL, -- 'vm','database','service','container','load_balancer','storage','serverless'
    platform        VARCHAR(100), -- 'vmware','hyper_v','ec2','azure_vm','gke_pod','rds','aurora'
    hostname        VARCHAR(255),
    ip_address      INET,
    tier            VARCHAR(50) NOT NULL DEFAULT 'standard', -- 'critical','high','standard','low'
    rto_seconds     INTEGER, -- asset-level RTO target
    rpo_seconds     INTEGER, -- asset-level RPO target
    metadata        JSONB NOT NULL DEFAULT '{}', -- platform-specific metadata
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    discovered_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assets_tenant ON assets (tenant_id);
CREATE INDEX idx_assets_environment ON assets (environment_id);
CREATE INDEX idx_assets_type ON assets (tenant_id, asset_type);
CREATE INDEX idx_assets_tier ON assets (tenant_id, tier);

-- Asset Dependencies (DAG edges)
CREATE TABLE asset_dependencies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    upstream_asset_id   UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    downstream_asset_id UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    dependency_type VARCHAR(50) NOT NULL, -- 'hard','soft','data','network'
    discovery_method VARCHAR(50) NOT NULL DEFAULT 'manual', -- 'manual','ai_inferred','telemetry','iac_parsed'
    confidence_score NUMERIC(3,2), -- 0.00 to 1.00 for AI-discovered dependencies
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (upstream_asset_id, downstream_asset_id),
    CHECK (upstream_asset_id != downstream_asset_id)
);

CREATE INDEX idx_asset_deps_upstream ON asset_dependencies (upstream_asset_id);
CREATE INDEX idx_asset_deps_downstream ON asset_dependencies (downstream_asset_id);

-- Replication Configurations
CREATE TABLE replication_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    source_asset_id UUID NOT NULL REFERENCES assets(id),
    target_environment_id UUID NOT NULL REFERENCES environments(id),
    replication_engine VARCHAR(50) NOT NULL, -- 'veeam','zerto','aws_drs','azure_asr','custom'
    replication_type VARCHAR(50) NOT NULL, -- 'continuous','scheduled','snapshot'
    current_rpo_seconds INTEGER, -- measured current RPO
    target_rpo_seconds  INTEGER NOT NULL, -- configured RPO target
    status          VARCHAR(50) NOT NULL DEFAULT 'active', -- 'active','paused','failed','initialising'
    engine_config   JSONB NOT NULL DEFAULT '{}', -- engine-specific configuration
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_replication_source ON replication_configs (source_asset_id);
CREATE INDEX idx_replication_status ON replication_configs (tenant_id, status);
```

## DR Plans & Runbooks

```sql
-- DR Plans
CREATE TABLE dr_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    plan_type       VARCHAR(50) NOT NULL, -- 'full_failover','partial_failover','failback','test_only'
    source_environment_id UUID NOT NULL REFERENCES environments(id),
    target_environment_id UUID NOT NULL REFERENCES environments(id),
    target_rto_seconds INTEGER NOT NULL,
    target_rpo_seconds INTEGER NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft', -- 'draft','review','approved','active','archived'
    version         INTEGER NOT NULL DEFAULT 1,
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dr_plans_tenant ON dr_plans (tenant_id);
CREATE INDEX idx_dr_plans_status ON dr_plans (tenant_id, status);

-- DR Plan Asset Assignments (which assets are protected by which plan)
CREATE TABLE plan_assets (
    plan_id         UUID NOT NULL REFERENCES dr_plans(id) ON DELETE CASCADE,
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    failover_priority INTEGER NOT NULL DEFAULT 100, -- lower = higher priority
    PRIMARY KEY (plan_id, asset_id)
);

-- Runbook Steps
CREATE TABLE runbook_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id         UUID NOT NULL REFERENCES dr_plans(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    step_order      INTEGER NOT NULL,
    step_type       VARCHAR(50) NOT NULL, -- 'automated','manual','approval','validation','notification'
    action_type     VARCHAR(100), -- 'start_replication','failover_vm','configure_network','run_script','health_check'
    target_asset_id UUID REFERENCES assets(id),
    estimated_duration_seconds INTEGER,
    timeout_seconds INTEGER DEFAULT 3600,
    is_critical     BOOLEAN NOT NULL DEFAULT true,
    on_failure      VARCHAR(50) NOT NULL DEFAULT 'halt', -- 'halt','skip','retry','manual_override'
    max_retries     INTEGER NOT NULL DEFAULT 0,
    action_config   JSONB NOT NULL DEFAULT '{}', -- step-specific parameters
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_runbook_steps_plan ON runbook_steps (plan_id, step_order);

-- Step Dependencies (within a runbook — which steps must complete before others)
CREATE TABLE step_dependencies (
    step_id             UUID NOT NULL REFERENCES runbook_steps(id) ON DELETE CASCADE,
    depends_on_step_id  UUID NOT NULL REFERENCES runbook_steps(id) ON DELETE CASCADE,
    PRIMARY KEY (step_id, depends_on_step_id),
    CHECK (step_id != depends_on_step_id)
);
```

## Failover Operations

```sql
-- Failover Executions (an actual invocation of a DR plan)
CREATE TABLE failover_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    plan_id         UUID NOT NULL REFERENCES dr_plans(id),
    execution_type  VARCHAR(50) NOT NULL, -- 'test','planned_failover','unplanned_failover','failback'
    trigger_type    VARCHAR(50) NOT NULL, -- 'manual','scheduled','ai_predicted','alert_triggered'
    status          VARCHAR(50) NOT NULL DEFAULT 'pending', -- 'pending','running','paused','succeeded','failed','cancelled','rolled_back'
    initiated_by    UUID REFERENCES users(id),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    actual_rto_seconds INTEGER, -- measured RTO
    actual_rpo_seconds INTEGER, -- measured RPO
    rto_met         BOOLEAN,
    rpo_met         BOOLEAN,
    failure_reason  TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_failover_tenant ON failover_executions (tenant_id);
CREATE INDEX idx_failover_plan ON failover_executions (plan_id);
CREATE INDEX idx_failover_status ON failover_executions (tenant_id, status);
CREATE INDEX idx_failover_started ON failover_executions (started_at DESC);

-- Step Execution Results
CREATE TABLE step_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    failover_execution_id UUID NOT NULL REFERENCES failover_executions(id) ON DELETE CASCADE,
    step_id         UUID NOT NULL REFERENCES runbook_steps(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending', -- 'pending','running','succeeded','failed','skipped','timed_out'
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    duration_seconds INTEGER,
    output_log      TEXT,
    error_message   TEXT,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    executed_by     UUID REFERENCES users(id), -- for manual steps
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_step_exec_failover ON step_executions (failover_execution_id);
CREATE INDEX idx_step_exec_status ON step_executions (status);
```

## Compliance & Audit

```sql
-- Compliance Frameworks
CREATE TABLE compliance_frameworks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE, -- 'iso_22301','soc2','dora','nis2','nist_800_34'
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(50),
    description     TEXT
);

-- Compliance Requirements (controls within a framework)
CREATE TABLE compliance_requirements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id    UUID NOT NULL REFERENCES compliance_frameworks(id),
    requirement_code VARCHAR(50) NOT NULL, -- e.g. 'A.17.1.1' for ISO 27001
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    test_frequency_days INTEGER, -- how often testing is required
    UNIQUE (framework_id, requirement_code)
);

-- Tenant Compliance Obligations (which frameworks a tenant must comply with)
CREATE TABLE tenant_compliance (
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    framework_id    UUID NOT NULL REFERENCES compliance_frameworks(id),
    enabled         BOOLEAN NOT NULL DEFAULT true,
    next_audit_date DATE,
    PRIMARY KEY (tenant_id, framework_id)
);

-- Compliance Evidence Records
CREATE TABLE compliance_evidence (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    requirement_id  UUID NOT NULL REFERENCES compliance_requirements(id),
    failover_execution_id UUID REFERENCES failover_executions(id),
    evidence_type   VARCHAR(50) NOT NULL, -- 'test_result','approval_record','configuration_snapshot','attestation'
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending', -- 'pending','approved','rejected','expired'
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    valid_from      DATE NOT NULL,
    valid_until     DATE,
    attachments     JSONB DEFAULT '[]', -- [{filename, storage_path, content_type, size_bytes}]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_evidence_tenant ON compliance_evidence (tenant_id);
CREATE INDEX idx_compliance_evidence_requirement ON compliance_evidence (requirement_id);
CREATE INDEX idx_compliance_evidence_validity ON compliance_evidence (valid_from, valid_until);

-- Audit Log (immutable)
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL, -- 'plan.created','failover.initiated','test.completed','approval.granted'
    entity_type     VARCHAR(50) NOT NULL, -- 'dr_plan','failover_execution','asset','user'
    entity_id       UUID NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_tenant ON audit_log (tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_log_user ON audit_log (user_id, created_at DESC);

-- Partition audit_log by month for performance
-- CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

## Monitoring & Alerting

```sql
-- Health Checks (recurring probes against assets)
CREATE TABLE health_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    check_type      VARCHAR(50) NOT NULL, -- 'replication_lag','service_health','network_latency','storage_health'
    interval_seconds INTEGER NOT NULL DEFAULT 300,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_check_at   TIMESTAMPTZ,
    last_status     VARCHAR(50), -- 'healthy','degraded','critical','unknown'
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_health_checks_asset ON health_checks (asset_id);

-- Health Check Results
CREATE TABLE health_check_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    health_check_id UUID NOT NULL REFERENCES health_checks(id) ON DELETE CASCADE,
    status          VARCHAR(50) NOT NULL, -- 'healthy','degraded','critical','error'
    response_time_ms INTEGER,
    details         JSONB NOT NULL DEFAULT '{}',
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_health_results_check ON health_check_results (health_check_id, checked_at DESC);

-- Alert Rules
CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    condition_type  VARCHAR(50) NOT NULL, -- 'rpo_breach','rto_risk','replication_lag','health_degraded'
    threshold_config JSONB NOT NULL, -- {metric, operator, value, duration_seconds}
    severity        VARCHAR(50) NOT NULL DEFAULT 'warning', -- 'info','warning','critical'
    notification_channels JSONB NOT NULL DEFAULT '[]', -- [{type:'email',target:'...'},{type:'slack',webhook:'...'}]
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Alert Instances
CREATE TABLE alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    alert_rule_id   UUID NOT NULL REFERENCES alert_rules(id),
    asset_id        UUID REFERENCES assets(id),
    severity        VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'open', -- 'open','acknowledged','resolved','suppressed'
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_tenant_status ON alerts (tenant_id, status);
CREATE INDEX idx_alerts_created ON alerts (created_at DESC);
```

## AI & Automation

```sql
-- AI Predictions (failure predictions, dependency inferences)
CREATE TABLE ai_predictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    prediction_type VARCHAR(50) NOT NULL, -- 'failure_prediction','dependency_inference','rto_gap','drift_detection'
    target_asset_id UUID REFERENCES assets(id),
    target_plan_id  UUID REFERENCES dr_plans(id),
    confidence_score NUMERIC(3,2) NOT NULL, -- 0.00 to 1.00
    severity        VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    recommended_action TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending', -- 'pending','accepted','dismissed','acted_on'
    model_version   VARCHAR(100),
    input_features  JSONB, -- what telemetry data was used
    acted_on_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_predictions_tenant ON ai_predictions (tenant_id, status);
CREATE INDEX idx_ai_predictions_asset ON ai_predictions (target_asset_id);

-- IaC Imports (parsed Terraform, CloudFormation, ARM templates)
CREATE TABLE iac_imports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    source_type     VARCHAR(50) NOT NULL, -- 'terraform_state','terraform_plan','cloudformation','arm_template'
    source_path     VARCHAR(500),
    import_status   VARCHAR(50) NOT NULL DEFAULT 'pending', -- 'pending','parsing','completed','failed'
    assets_discovered INTEGER DEFAULT 0,
    dependencies_discovered INTEGER DEFAULT 0,
    error_message   TEXT,
    raw_content_hash VARCHAR(64), -- SHA-256 for change detection
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_iac_imports_tenant ON iac_imports (tenant_id);
```

## Notifications & Integrations

```sql
-- Integration Connections (ServiceNow, Jira, Slack, etc.)
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    integration_type VARCHAR(50) NOT NULL, -- 'servicenow','jira','slack','teams','pagerduty','email','webhook'
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}', -- encrypted connection config
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Notification History
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    integration_id  UUID REFERENCES integrations(id),
    notification_type VARCHAR(50) NOT NULL, -- 'alert','test_result','failover_status','compliance_reminder'
    channel         VARCHAR(50) NOT NULL, -- 'email','slack','teams','webhook','sms'
    recipient       VARCHAR(320),
    subject         VARCHAR(255),
    body            TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending', -- 'pending','sent','failed','bounced'
    sent_at         TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_tenant ON notifications (tenant_id, created_at DESC);
```

## Row-Level Security (Multi-Tenancy)

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE environments ENABLE ROW LEVEL SECURITY;
ALTER TABLE assets ENABLE ROW LEVEL SECURITY;
ALTER TABLE dr_plans ENABLE ROW LEVEL SECURITY;
ALTER TABLE failover_executions ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;

-- Example RLS policy (repeated for each table)
CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation ON assets
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- Set tenant context at connection time:
-- SET app.current_tenant_id = '<tenant-uuid>';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 5 | tenants, users, roles, permissions, role_permissions, user_roles |
| Infrastructure Assets | 4 | environments, assets, asset_dependencies, replication_configs |
| DR Plans & Runbooks | 4 | dr_plans, plan_assets, runbook_steps, step_dependencies |
| Failover Operations | 2 | failover_executions, step_executions |
| Compliance & Audit | 5 | compliance_frameworks, compliance_requirements, tenant_compliance, compliance_evidence, audit_log |
| Monitoring & Alerting | 4 | health_checks, health_check_results, alert_rules, alerts |
| AI & Automation | 2 | ai_predictions, iac_imports |
| Notifications & Integrations | 2 | integrations, notifications |
| **Total** | **28** | Plus RLS policies on all tenant-scoped tables |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation across microservices and avoids sequential ID enumeration attacks on a security-sensitive DR platform.

2. **Explicit tenant_id on every table with RLS** — PostgreSQL Row-Level Security enforces tenant isolation at the database level, providing defence-in-depth beyond application logic. The `current_setting('app.current_tenant_id')` pattern is set at connection time.

3. **DAG-based dependency modeling with cycle prevention** — `asset_dependencies` uses a CHECK constraint to prevent self-references. Cycle detection must be handled at the application layer using recursive CTEs before inserting edges. The `discovery_method` and `confidence_score` columns support AI-inferred dependencies alongside manually defined ones.

4. **Separate step_dependencies table** — rather than relying solely on `step_order`, explicit step dependencies allow parallel execution of independent steps within a runbook while enforcing ordering where required.

5. **Immutable audit_log with time-based partitioning** — the audit log is append-only with no UPDATE or DELETE operations permitted. Monthly partitioning keeps query performance manageable as the log grows. The schema aligns with CloudEvents structure for potential event streaming.

6. **Compliance evidence linked to test executions** — `compliance_evidence` references both a `compliance_requirement` and optionally a `failover_execution`, creating a direct traceability chain from regulatory control to test evidence.

7. **Replication engine abstraction** — `replication_configs` uses a `replication_engine` discriminator with engine-specific config in JSONB, allowing support for Veeam, Zerto, AWS DRS, and Azure ASR without engine-specific tables.

8. **Measured vs. target RTO/RPO at multiple levels** — RTO/RPO targets are set on both assets and DR plans, while actual measured values are captured on each failover execution, enabling gap analysis queries.

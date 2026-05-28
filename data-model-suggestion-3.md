# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Disaster Recovery Orchestrator · Created: 2026-05-20

## Philosophy

This model keeps a lean relational core for entities that are queried frequently and have stable schemas (tenants, users, DR plans, failover executions), while using PostgreSQL JSONB columns extensively for domain-specific variability: asset metadata that differs by platform (VMware vs. EC2 vs. GKE), replication engine configurations that vary by vendor (Veeam vs. Zerto vs. AWS DRS), jurisdiction-specific compliance fields, and runbook step parameters that differ by action type.

The rationale is pragmatic: a DR orchestrator must support dozens of infrastructure platforms, multiple replication engines, several compliance frameworks, and varying deployment contexts — all within a single schema. A fully normalised model would require hundreds of columns (most null for any given row) or dozens of type-specific subtables. A fully document-oriented model would sacrifice the query power needed for cross-cutting analytics (e.g., "all assets with RPO breach across all tenants"). The hybrid approach splits the difference: relational structure for what is universal, JSONB for what varies.

This pattern is widely used in modern SaaS platforms. GitHub stores issue metadata as JSONB alongside relational core fields. Stripe uses JSONB for payment method details that vary by type. The approach enables rapid iteration — new asset types, replication engines, or compliance frameworks can be supported by adding JSONB field conventions rather than schema migrations.

**Best for:** Rapid MVP development and teams building a multi-cloud, multi-engine orchestrator where platform-specific and jurisdiction-specific fields vary widely and change frequently.

**Trade-offs:**
- Pro: Fewest tables (~18) — dramatically simpler than a fully normalised model
- Pro: No ALTER TABLE migrations for new asset types, engines, or compliance fields
- Pro: JSONB GIN indexes enable fast containment queries on variable fields
- Pro: Platform-specific and jurisdiction-specific data coexists without sparse columns
- Pro: Rapid prototyping — add new data without schema changes
- Con: JSONB fields lack database-level type enforcement — validation must happen in application code
- Con: JSONB field naming conventions require documentation discipline to avoid drift
- Con: Complex JSONB queries (nested path operations) are less readable than simple column queries
- Con: Less discoverable for new developers — the schema alone does not document all fields
- Con: ORM support for JSONB varies; some frameworks treat it as an opaque blob

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 22301:2019 | DR plan lifecycle tracked via relational status fields; test evidence in structured JSONB with standard field conventions |
| ISO 3166-1 | `jurisdiction` fields use ISO 3166-1 alpha-2 codes; jurisdiction-specific compliance config stored in JSONB |
| DORA (EU 2022/2554) | DORA-specific requirements stored as JSONB compliance_config entries; two-hour RTO target in relational rto_seconds |
| SOC 2 Availability TSC | Test results captured as relational records with JSONB details for framework-specific evidence fields |
| CloudEvents 1.0 | Event log entries use CloudEvents-compatible JSONB structure |
| OpenAPI 3.1 | REST resources map to relational tables; JSONB fields documented as JSON Schema within OpenAPI spec |
| Terraform Provider Schema | IaC parser outputs stored as JSONB, matching Terraform state file structure |

---

## Core Tables

```sql
-- Tenants
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_rto_seconds": 3600,
    --   "default_rpo_seconds": 300,
    --   "compliance_frameworks": ["iso_22301", "soc2", "dora"],
    --   "notification_defaults": {"email": true, "slack": false},
    --   "data_residency": "EU"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users with RBAC
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    roles           JSONB NOT NULL DEFAULT '["viewer"]',
    -- roles example: ["admin", "plan_editor", "failover_operator"]
    -- Permissions derived from roles at application layer
    preferences     JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users (tenant_id);
CREATE INDEX idx_users_roles ON users USING GIN (roles);
```

## Environments & Assets

```sql
-- Environments
CREATE TABLE environments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    environment_type VARCHAR(50) NOT NULL, -- 'production','dr_target','staging','cleanroom'
    cloud_provider  VARCHAR(50), -- 'aws','azure','gcp','on_premises'
    region          VARCHAR(100),
    jurisdiction    VARCHAR(10), -- ISO 3166-1 alpha-2
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config varies by cloud_provider:
    -- AWS:   {"account_id":"123456","vpc_id":"vpc-abc","assume_role_arn":"arn:aws:iam::..."}
    -- Azure: {"subscription_id":"...","resource_group":"...","recovery_vault":"..."}
    -- GCP:   {"project_id":"...","network":"...","service_account":"..."}
    -- On-prem: {"vcenter_host":"...","datacenter":"dc1","cluster":"prod-cluster"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_environments_tenant ON environments (tenant_id);

-- Assets (unified table with JSONB for platform-specific fields)
CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    environment_id  UUID NOT NULL REFERENCES environments(id),
    name            VARCHAR(255) NOT NULL,
    asset_type      VARCHAR(50) NOT NULL, -- 'vm','database','service','container','load_balancer','serverless'
    platform        VARCHAR(100) NOT NULL, -- 'vmware','ec2','azure_vm','gke_pod','rds','aurora','on_prem_physical'
    tier            VARCHAR(50) NOT NULL DEFAULT 'standard',
    rto_seconds     INTEGER,
    rpo_seconds     INTEGER,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    health_status   VARCHAR(50) NOT NULL DEFAULT 'unknown',
    -- Platform-specific details in JSONB
    platform_details JSONB NOT NULL DEFAULT '{}',
    -- VMware:   {"vcenter_id":"...","vm_uuid":"...","datastore":"ds1","cpu":4,"memory_gb":16,"disk_gb":500}
    -- EC2:      {"instance_id":"i-abc123","instance_type":"m5.xlarge","ami_id":"ami-xxx","security_groups":["sg-1"]}
    -- RDS:      {"db_instance_id":"prod-db","engine":"postgres","engine_version":"15.4","multi_az":true}
    -- GKE Pod:  {"cluster":"prod","namespace":"default","deployment":"api-server","replicas":3}
    -- Physical: {"serial_number":"SN123","rack":"R42","os":"rhel9","bmc_ip":"10.0.1.5"}
    --
    -- Replication status (updated by health checks)
    replication_status JSONB NOT NULL DEFAULT '{}',
    -- {"engine":"zerto","status":"active","current_rpo_seconds":5,"journal_size_gb":120,"last_sync":"2026-05-20T10:00:00Z"}
    --
    tags            JSONB NOT NULL DEFAULT '[]', -- ["production","tier-1","pci-scope"]
    discovered_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assets_tenant ON assets (tenant_id);
CREATE INDEX idx_assets_environment ON assets (environment_id);
CREATE INDEX idx_assets_type ON assets (tenant_id, asset_type);
CREATE INDEX idx_assets_platform ON assets (tenant_id, platform);
CREATE INDEX idx_assets_health ON assets (tenant_id, health_status);
CREATE INDEX idx_assets_tags ON assets USING GIN (tags);
CREATE INDEX idx_assets_platform_details ON assets USING GIN (platform_details);

-- Asset Dependencies
CREATE TABLE asset_dependencies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    upstream_asset_id   UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    downstream_asset_id UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    dependency_type VARCHAR(50) NOT NULL, -- 'hard','soft','data','network'
    discovery_method VARCHAR(50) NOT NULL DEFAULT 'manual',
    confidence_score NUMERIC(3,2),
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- {"discovered_from":"network_flow","flow_count":1523,"first_seen":"2026-04-01","ports":[5432,443]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (upstream_asset_id, downstream_asset_id),
    CHECK (upstream_asset_id != downstream_asset_id)
);

CREATE INDEX idx_deps_upstream ON asset_dependencies (upstream_asset_id);
CREATE INDEX idx_deps_downstream ON asset_dependencies (downstream_asset_id);
```

## DR Plans & Runbooks

```sql
-- DR Plans
CREATE TABLE dr_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    plan_type       VARCHAR(50) NOT NULL,
    source_environment_id UUID NOT NULL REFERENCES environments(id),
    target_environment_id UUID NOT NULL REFERENCES environments(id),
    target_rto_seconds INTEGER NOT NULL,
    target_rpo_seconds INTEGER NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    version         INTEGER NOT NULL DEFAULT 1,
    -- Assets included in this plan (denormalised for fast access)
    asset_ids       JSONB NOT NULL DEFAULT '[]', -- ["uuid1","uuid2","uuid3"]
    -- Network mapping configuration
    network_config  JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "subnet_mappings": [{"source":"10.0.1.0/24","target":"10.1.1.0/24"}],
    --   "dns_updates": [{"record":"api.prod.internal","target_ip":"10.1.1.10"}],
    --   "security_group_mappings": [{"source":"sg-prod","target":"sg-dr"}]
    -- }
    --
    -- Compliance requirements applicable to this plan
    compliance_config JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "frameworks": ["iso_22301","dora"],
    --   "test_frequency_days": 90,
    --   "max_rto_seconds": 7200,
    --   "requires_approval": true,
    --   "dora_specific": {"two_hour_rto_required": true, "immutable_backup_required": true}
    -- }
    --
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES users(id),
    last_tested_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plans_tenant ON dr_plans (tenant_id);
CREATE INDEX idx_plans_status ON dr_plans (tenant_id, status);

-- Runbook Steps (JSONB for action-type-specific parameters)
CREATE TABLE runbook_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id         UUID NOT NULL REFERENCES dr_plans(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    step_order      INTEGER NOT NULL,
    step_type       VARCHAR(50) NOT NULL, -- 'automated','manual','approval','validation','notification'
    action_type     VARCHAR(100) NOT NULL, -- 'failover_vm','configure_dns','run_health_check','notify_stakeholders'
    target_asset_id UUID REFERENCES assets(id),
    is_critical     BOOLEAN NOT NULL DEFAULT true,
    on_failure      VARCHAR(50) NOT NULL DEFAULT 'halt',
    timeout_seconds INTEGER DEFAULT 3600,
    -- Dependencies: which step IDs must complete before this one
    depends_on      JSONB NOT NULL DEFAULT '[]', -- ["step-uuid-1","step-uuid-2"]
    -- Action-specific parameters in JSONB
    action_params   JSONB NOT NULL DEFAULT '{}',
    -- failover_vm:     {"replication_engine":"veeam","target_host":"esxi-dr-01","power_on":true}
    -- configure_dns:   {"provider":"route53","zone_id":"Z123","records":[{"name":"api","type":"A","value":"10.1.1.10"}]}
    -- run_health_check: {"check_type":"http","url":"https://api.dr.example.com/health","expected_status":200,"timeout_ms":5000}
    -- run_script:       {"interpreter":"bash","script":"#!/bin/bash\necho 'checking db...'","target":"ssh://dr-db-01"}
    -- notify_stakeholders: {"channels":["slack","email"],"template":"failover_progress","recipients":["team-infra"]}
    --
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_steps_plan ON runbook_steps (plan_id, step_order);
```

## Failover Executions

```sql
-- Failover Executions
CREATE TABLE failover_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    plan_id         UUID NOT NULL REFERENCES dr_plans(id),
    execution_type  VARCHAR(50) NOT NULL,
    trigger_type    VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    initiated_by    UUID REFERENCES users(id),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    actual_rto_seconds INTEGER,
    actual_rpo_seconds INTEGER,
    rto_met         BOOLEAN,
    rpo_met         BOOLEAN,
    -- Per-step results stored as JSONB array (denormalised for single-query access)
    step_results    JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {
    --     "step_id": "uuid",
    --     "step_name": "Failover Primary DB",
    --     "status": "succeeded",
    --     "started_at": "2026-05-20T10:00:00Z",
    --     "completed_at": "2026-05-20T10:03:22Z",
    --     "duration_seconds": 202,
    --     "output_log": "VM powered on successfully...",
    --     "retry_count": 0
    --   },
    --   ...
    -- ]
    --
    -- Compliance evidence snapshot at time of execution
    compliance_evidence JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "frameworks_tested": ["iso_22301","dora"],
    --   "rto_target": 3600, "rto_actual": 1847, "rto_met": true,
    --   "rpo_target": 300, "rpo_actual": 12, "rpo_met": true,
    --   "approved_by": "user-uuid", "approved_at": "2026-05-20T09:55:00Z",
    --   "attestation_text": "DR test completed successfully within DORA 2-hour RTO requirement..."
    -- }
    --
    failure_reason  TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_executions_tenant ON failover_executions (tenant_id, status);
CREATE INDEX idx_executions_plan ON failover_executions (plan_id);
CREATE INDEX idx_executions_started ON failover_executions (started_at DESC);
CREATE INDEX idx_executions_compliance ON failover_executions USING GIN (compliance_evidence);
```

## Event Log & Monitoring

```sql
-- Event Log (CloudEvents-compatible, append-only)
CREATE TABLE event_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    -- CloudEvents envelope
    ce_type         VARCHAR(200) NOT NULL, -- 'com.dro.failover.initiated','com.dro.health.degraded'
    ce_source       VARCHAR(500) NOT NULL,
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    ce_subject      VARCHAR(500), -- entity identifier
    data            JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}'
    -- metadata: {"user_id":"...","correlation_id":"...","ip_address":"..."}
);

CREATE INDEX idx_event_log_tenant ON event_log (tenant_id, ce_time DESC);
CREATE INDEX idx_event_log_type ON event_log (tenant_id, ce_type, ce_time DESC);
CREATE INDEX idx_event_log_subject ON event_log (ce_subject, ce_time DESC);

-- AI Predictions & Insights
CREATE TABLE ai_insights (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    insight_type    VARCHAR(50) NOT NULL, -- 'failure_prediction','dependency_inference','rto_gap','drift_detection'
    severity        VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    confidence_score NUMERIC(3,2) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    -- Flexible target and details
    target          JSONB NOT NULL DEFAULT '{}',
    -- {"asset_id":"...","plan_id":"...","environment_id":"..."}
    details         JSONB NOT NULL DEFAULT '{}',
    -- failure_prediction: {"indicators":["disk_latency_spike","replication_lag_increase"],"predicted_failure_window":"2h","recommended_action":"initiate_pre_emptive_failover"}
    -- dependency_inference: {"upstream":"asset-1","downstream":"asset-2","evidence":"network_flow","flow_count":2341}
    -- rto_gap: {"plan_id":"...","target_rto":3600,"simulated_rto":5200,"gap_pct":44}
    model_version   VARCHAR(100),
    acted_on_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_insights_tenant ON ai_insights (tenant_id, status);
CREATE INDEX idx_insights_type ON ai_insights (tenant_id, insight_type);

-- Integrations (ServiceNow, Jira, Slack, replication engines)
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    integration_type VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- All connection config in JSONB (encrypted at rest)
    config          JSONB NOT NULL DEFAULT '{}',
    -- servicenow: {"instance_url":"https://xxx.service-now.com","client_id":"...","client_secret":"encrypted:..."}
    -- slack:      {"webhook_url":"https://hooks.slack.com/...","channel":"#dr-alerts"}
    -- veeam:      {"api_url":"https://vro.corp.local","client_id":"...","client_secret":"encrypted:..."}
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_integrations_tenant ON integrations (tenant_id);
```

## Row-Level Security

```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE environments ENABLE ROW LEVEL SECURITY;
ALTER TABLE assets ENABLE ROW LEVEL SECURITY;
ALTER TABLE dr_plans ENABLE ROW LEVEL SECURITY;
ALTER TABLE failover_executions ENABLE ROW LEVEL SECURITY;
ALTER TABLE event_log ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON assets
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
-- (repeated for each table)
```

## Example Queries

```sql
-- Find all EC2 instances with replication lag > 60 seconds
SELECT name, platform_details->>'instance_id' AS instance_id,
       (replication_status->>'current_rpo_seconds')::INT AS rpo_seconds
FROM assets
WHERE tenant_id = '...'
  AND platform = 'ec2'
  AND (replication_status->>'current_rpo_seconds')::INT > 60;

-- Find all assets tagged with 'pci-scope'
SELECT name, asset_type, platform
FROM assets
WHERE tenant_id = '...'
  AND tags @> '["pci-scope"]';

-- DR plans requiring DORA compliance
SELECT name, status, compliance_config->'dora_specific'->>'two_hour_rto_required' AS dora_2hr
FROM dr_plans
WHERE tenant_id = '...'
  AND compliance_config->'frameworks' @> '"dora"';

-- Failover execution with inline step results — no JOIN needed
SELECT id, status, actual_rto_seconds,
       jsonb_array_length(step_results) AS total_steps,
       (SELECT COUNT(*) FROM jsonb_array_elements(step_results) s
        WHERE s->>'status' = 'succeeded') AS succeeded_steps
FROM failover_executions
WHERE plan_id = '...'
ORDER BY started_at DESC
LIMIT 5;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenants, users (RBAC roles in JSONB) |
| Infrastructure | 3 | environments, assets, asset_dependencies |
| DR Plans & Runbooks | 2 | dr_plans, runbook_steps (step deps in JSONB) |
| Failover Operations | 1 | failover_executions (step results in JSONB) |
| Event Log & Monitoring | 1 | event_log (CloudEvents-compatible) |
| AI & Automation | 1 | ai_insights |
| Integrations | 1 | integrations |
| **Total** | **11** | Minimal table count; extensibility via JSONB fields |

---

## Key Design Decisions

1. **JSONB for platform-specific asset details** — a single `assets` table handles VMware VMs, EC2 instances, RDS databases, GKE pods, and physical servers. The `platform_details` JSONB column holds platform-specific fields, avoiding a sprawl of `assets_vmware`, `assets_ec2`, `assets_rds` subtables. GIN indexes on `platform_details` enable efficient queries.

2. **Roles as JSONB array on users** — for an MVP, storing roles as `["admin","plan_editor"]` in a JSONB column avoids 3 junction tables (roles, permissions, user_roles). Full RBAC with fine-grained permissions can be added later with dedicated tables if needed.

3. **Step results embedded in failover_executions** — each failover execution stores its step-by-step results as a JSONB array directly on the execution record. This means a single query retrieves the full execution history without JOINs. The trade-off is that querying across step results of multiple executions requires JSONB array operators.

4. **Step dependencies as JSONB array** — `depends_on` stores step UUIDs as a JSONB array rather than a separate junction table, simplifying the schema while still supporting DAG-based step ordering.

5. **Compliance config on DR plans** — rather than a separate compliance framework/requirement/obligation table hierarchy, compliance requirements are stored as JSONB on the DR plan itself. This is simpler but means updating compliance requirements across all plans requires a batch JSONB update.

6. **Separate event_log table (not event-sourced)** — the event log is an append-only record for observability and compliance, but it is NOT the source of truth. Current state lives in the relational tables. This is simpler than full event sourcing while still providing audit capabilities.

7. **Integration config in JSONB** — each integration type (ServiceNow, Slack, Veeam API) has different connection parameters. JSONB avoids needing type-specific config tables while keeping all integration management in one place.

8. **GIN indexes on JSONB columns** — PostgreSQL GIN indexes on `tags`, `platform_details`, and `compliance_evidence` enable efficient containment queries (`@>` operator) without full table scans, making JSONB queries performant at scale.

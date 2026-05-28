# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Disaster Recovery Orchestrator · Created: 2026-05-20

## Philosophy

This model centres the design around the infrastructure dependency graph — the most critical and complex data structure in a DR orchestrator. Service dependencies form a directed acyclic graph (DAG) that determines failover sequencing: database servers must come up before application servers, load balancers must be configured before traffic can be routed, and DNS changes must propagate before health checks pass. Getting this graph wrong means a failed recovery.

The design uses a property graph layer (generic `graph_nodes` and `graph_edges` tables) alongside traditional relational tables for operational CRUD. The graph layer models assets, services, environments, and their relationships using a flexible node/edge structure that supports arbitrary relationship types, weighted edges, and path queries. Relational tables handle DR plans, failover executions, users, and compliance — entities that benefit from strict schemas and foreign key constraints.

This approach draws from how infrastructure observability platforms (Datadog, ServiceNow CMDB, PagerDuger) model service maps and dependency topologies. It is particularly well-suited for a DR orchestrator because the dependency graph is not static — AI-inferred dependencies from network flows have different confidence levels than manually declared ones, and the graph evolves as infrastructure changes. A graph-native model makes traversal queries (impact analysis, failover ordering, blast radius calculation) natural rather than requiring complex recursive CTEs.

**Best for:** Organisations with complex, deeply interdependent infrastructure where dependency graph traversal, impact analysis, and intelligent failover sequencing are the primary value drivers.

**Trade-offs:**
- Pro: Dependency graph queries (shortest path, impact radius, topological sort) are natural and performant
- Pro: Flexible node/edge model accommodates new entity types and relationship types without schema changes
- Pro: AI-inferred dependencies integrate naturally as weighted edges with confidence scores
- Pro: Supports multiple concurrent graph versions (planned vs. actual vs. AI-predicted topology)
- Pro: Impact analysis ("if this server fails, what is affected?") is a simple graph traversal
- Con: Graph layer adds complexity — developers must understand both relational and graph query patterns
- Con: Recursive CTEs for deep graph traversal can be expensive on very large graphs (1000+ nodes)
- Con: Foreign key integrity between graph layer and relational layer requires careful application logic
- Con: Less standardised tooling compared to pure relational — fewer ORM patterns available
- Con: Reporting queries that span both graph and relational data require JOINs between the two layers

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 22301:2019 | Dependency graph provides the documented recovery procedure ordering required by the standard |
| NIST SP 800-34 | Service dependency mapping aligns with NIST's requirement for documented system interconnections |
| ITIL 4 Service Continuity | Graph model mirrors ITIL Configuration Management Database (CMDB) patterns for service dependency tracking |
| ISO/IEC 27001:2022 | Graph-based impact analysis supports Annex A risk assessment by quantifying dependency chains |
| DORA (EU 2022/2554) | Graph traversal enables automated identification of critical service paths requiring two-hour RTO |
| CloudEvents 1.0 | Event log uses CloudEvents-compatible structure for graph change events |
| JSON-LD 1.1 | Graph data can be serialised as JSON-LD for interoperability with external CMDB/topology systems |

---

## Graph Layer

```sql
-- ============================================================
-- GRAPH NODES: Any entity that participates in the dependency graph
-- ============================================================
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    -- Node types:
    -- 'asset'         — VM, database, service, container, load balancer
    -- 'environment'   — production site, DR target, cleanroom
    -- 'service_group' — logical grouping of assets (e.g. "payment processing tier")
    -- 'network_zone'  — network segment (VPC, subnet, VLAN)
    -- 'data_store'    — shared storage, S3 bucket, NFS mount
    -- 'external_dep'  — third-party dependency (Stripe API, DNS provider)
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    -- 'active','degraded','offline','decommissioned','planned'
    tier            VARCHAR(50) NOT NULL DEFAULT 'standard',
    -- 'critical','high','standard','low'
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Flexible properties vary by node_type:
    -- asset:          {"asset_type":"vm","platform":"ec2","instance_id":"i-abc","ip":"10.0.1.5","cpu":4,"memory_gb":16}
    -- environment:    {"cloud_provider":"aws","region":"us-east-1","jurisdiction":"US","vpc_id":"vpc-xxx"}
    -- service_group:  {"description":"Payment processing","owner":"team-payments","sla_tier":"platinum"}
    -- network_zone:   {"cidr":"10.0.1.0/24","firewall_rules":["allow-443","allow-5432"]}
    -- external_dep:   {"provider":"stripe","api_endpoint":"https://api.stripe.com","health_check_url":"https://status.stripe.com"}
    health_status   VARCHAR(50) NOT NULL DEFAULT 'unknown',
    rto_seconds     INTEGER,
    rpo_seconds     INTEGER,
    -- Replication state (for asset nodes)
    replication     JSONB DEFAULT '{}',
    -- {"engine":"zerto","status":"active","current_rpo_seconds":5,"target_environment_id":"uuid"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_tenant ON graph_nodes (tenant_id);
CREATE INDEX idx_graph_nodes_type ON graph_nodes (tenant_id, node_type);
CREATE INDEX idx_graph_nodes_status ON graph_nodes (tenant_id, status);
CREATE INDEX idx_graph_nodes_tier ON graph_nodes (tenant_id, tier);
CREATE INDEX idx_graph_nodes_health ON graph_nodes (tenant_id, health_status);
CREATE INDEX idx_graph_nodes_props ON graph_nodes USING GIN (properties);

-- ============================================================
-- GRAPH EDGES: Relationships between nodes
-- ============================================================
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    -- Edge types:
    -- 'depends_on'      — source depends on target (target must be up first)
    -- 'replicates_to'   — source is replicated to target
    -- 'hosts'           — environment hosts asset
    -- 'member_of'       — asset is member of service group
    -- 'connects_to'     — network connectivity (data flow)
    -- 'backed_up_by'    — source is backed up by target storage
    -- 'load_balances'   — load balancer distributes to target
    direction       VARCHAR(10) NOT NULL DEFAULT 'directed', -- 'directed','bidirectional'
    weight          NUMERIC(5,2) NOT NULL DEFAULT 1.0,
    -- Weight semantics vary by edge_type:
    -- depends_on: confidence score (0.0-1.0) for AI-inferred, 1.0 for manual
    -- connects_to: traffic volume normalised (0.0-1.0)
    -- replicates_to: priority (lower = higher priority)
    discovery_method VARCHAR(50) NOT NULL DEFAULT 'manual',
    -- 'manual','ai_inferred','network_flow','iac_parsed','api_discovered'
    properties      JSONB NOT NULL DEFAULT '{}',
    -- depends_on:    {"dependency_type":"hard","ports":[5432],"protocol":"tcp","latency_ms":2}
    -- connects_to:   {"bandwidth_mbps":1000,"flow_count":15234,"first_seen":"2026-04-01"}
    -- replicates_to: {"engine":"aws_drs","replication_type":"continuous","journal_hours":72}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_node_id, target_node_id, edge_type),
    CHECK (source_node_id != target_node_id)
);

CREATE INDEX idx_graph_edges_source ON graph_edges (source_node_id);
CREATE INDEX idx_graph_edges_target ON graph_edges (target_node_id);
CREATE INDEX idx_graph_edges_tenant ON graph_edges (tenant_id);
CREATE INDEX idx_graph_edges_type ON graph_edges (tenant_id, edge_type);
CREATE INDEX idx_graph_edges_discovery ON graph_edges (tenant_id, discovery_method);

-- ============================================================
-- GRAPH SNAPSHOTS: Point-in-time captures of the full graph
-- Used for: before/after comparison, drift detection, rollback
-- ============================================================
CREATE TABLE graph_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    snapshot_type   VARCHAR(50) NOT NULL, -- 'scheduled','pre_failover','post_change','manual'
    description     TEXT,
    node_count      INTEGER NOT NULL,
    edge_count      INTEGER NOT NULL,
    nodes           JSONB NOT NULL, -- serialised array of all nodes at snapshot time
    edges           JSONB NOT NULL, -- serialised array of all edges at snapshot time
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_snapshots_tenant ON graph_snapshots (tenant_id, created_at DESC);
```

## Graph Query Examples

```sql
-- ============================================================
-- TOPOLOGICAL SORT: Determine failover ordering for a DR plan
-- Returns assets in dependency order (leaf nodes first)
-- ============================================================
WITH RECURSIVE dependency_order AS (
    -- Start with assets that have no dependencies (leaf nodes)
    SELECT gn.id, gn.name, gn.node_type, 0 AS depth
    FROM graph_nodes gn
    LEFT JOIN graph_edges ge ON ge.source_node_id = gn.id AND ge.edge_type = 'depends_on' AND ge.is_active = true
    WHERE gn.tenant_id = $1
      AND gn.id = ANY($2::UUID[]) -- array of asset IDs in the DR plan
      AND ge.id IS NULL -- no outgoing depends_on edges
    UNION ALL
    -- Walk up the dependency chain
    SELECT gn.id, gn.name, gn.node_type, do.depth + 1
    FROM dependency_order do
    JOIN graph_edges ge ON ge.target_node_id = do.id AND ge.edge_type = 'depends_on' AND ge.is_active = true
    JOIN graph_nodes gn ON gn.id = ge.source_node_id
    WHERE gn.id = ANY($2::UUID[])
)
SELECT id, name, node_type, MAX(depth) AS failover_order
FROM dependency_order
GROUP BY id, name, node_type
ORDER BY failover_order ASC;

-- ============================================================
-- BLAST RADIUS: If a node fails, what downstream services are affected?
-- ============================================================
WITH RECURSIVE blast_radius AS (
    SELECT $1::UUID AS node_id, 0 AS distance
    UNION ALL
    SELECT ge.source_node_id, br.distance + 1
    FROM blast_radius br
    JOIN graph_edges ge ON ge.target_node_id = br.node_id
        AND ge.edge_type = 'depends_on'
        AND ge.is_active = true
    WHERE br.distance < 10 -- safety limit
)
SELECT gn.id, gn.name, gn.node_type, gn.tier, br.distance
FROM blast_radius br
JOIN graph_nodes gn ON gn.id = br.node_id
WHERE gn.tenant_id = $2
ORDER BY br.distance, gn.tier;

-- ============================================================
-- CRITICAL PATH: Find the longest dependency chain (determines minimum RTO)
-- ============================================================
WITH RECURSIVE critical_path AS (
    SELECT gn.id, gn.name, COALESCE(gn.rto_seconds, 0) AS cumulative_rto,
           ARRAY[gn.id] AS path, 1 AS chain_length
    FROM graph_nodes gn
    LEFT JOIN graph_edges ge ON ge.source_node_id = gn.id AND ge.edge_type = 'depends_on' AND ge.is_active = true
    WHERE gn.tenant_id = $1 AND ge.id IS NULL -- leaf nodes
    UNION ALL
    SELECT gn.id, gn.name,
           cp.cumulative_rto + COALESCE(gn.rto_seconds, 0),
           cp.path || gn.id,
           cp.chain_length + 1
    FROM critical_path cp
    JOIN graph_edges ge ON ge.target_node_id = cp.id AND ge.edge_type = 'depends_on' AND ge.is_active = true
    JOIN graph_nodes gn ON gn.id = ge.source_node_id
    WHERE NOT gn.id = ANY(cp.path) -- cycle prevention
)
SELECT path, cumulative_rto, chain_length
FROM critical_path
ORDER BY cumulative_rto DESC
LIMIT 5;
```

## Relational Layer (Operational CRUD)

```sql
-- Tenants
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    roles           JSONB NOT NULL DEFAULT '["viewer"]',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users (tenant_id);

-- DR Plans (reference graph nodes for asset inclusion)
CREATE TABLE dr_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    plan_type       VARCHAR(50) NOT NULL,
    source_environment_node_id UUID NOT NULL REFERENCES graph_nodes(id),
    target_environment_node_id UUID NOT NULL REFERENCES graph_nodes(id),
    target_rto_seconds INTEGER NOT NULL,
    target_rpo_seconds INTEGER NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    version         INTEGER NOT NULL DEFAULT 1,
    -- Failover ordering derived from graph at plan approval time
    computed_failover_order JSONB DEFAULT '[]',
    -- [{"node_id":"uuid","name":"Primary DB","order":1,"estimated_rto":120},
    --  {"node_id":"uuid","name":"App Server 1","order":2,"estimated_rto":60}, ...]
    computed_critical_path_rto INTEGER, -- sum of RTO along longest dependency chain
    graph_snapshot_id UUID REFERENCES graph_snapshots(id), -- graph state when plan was approved
    compliance_config JSONB NOT NULL DEFAULT '{}',
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES users(id),
    last_tested_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plans_tenant ON dr_plans (tenant_id, status);

-- Plan Node Assignments (which graph nodes are in this plan)
CREATE TABLE plan_nodes (
    plan_id         UUID NOT NULL REFERENCES dr_plans(id) ON DELETE CASCADE,
    node_id         UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    include_dependencies BOOLEAN NOT NULL DEFAULT true, -- auto-include upstream dependencies
    PRIMARY KEY (plan_id, node_id)
);

-- Runbook Steps (can reference graph nodes)
CREATE TABLE runbook_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id         UUID NOT NULL REFERENCES dr_plans(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    step_order      INTEGER NOT NULL,
    step_type       VARCHAR(50) NOT NULL,
    action_type     VARCHAR(100) NOT NULL,
    target_node_id  UUID REFERENCES graph_nodes(id), -- graph node this step acts on
    is_critical     BOOLEAN NOT NULL DEFAULT true,
    on_failure      VARCHAR(50) NOT NULL DEFAULT 'halt',
    timeout_seconds INTEGER DEFAULT 3600,
    depends_on      JSONB NOT NULL DEFAULT '[]',
    action_params   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_steps_plan ON runbook_steps (plan_id, step_order);

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
    -- Snapshot of graph state at execution start
    graph_snapshot_id UUID REFERENCES graph_snapshots(id),
    -- Computed blast radius at time of execution
    blast_radius    JSONB DEFAULT '{}',
    -- {"affected_nodes":15,"critical_nodes":3,"estimated_impact":"high","affected_service_groups":["payments","auth"]}
    step_results    JSONB NOT NULL DEFAULT '[]',
    compliance_evidence JSONB NOT NULL DEFAULT '{}',
    failure_reason  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_executions_tenant ON failover_executions (tenant_id, status);
CREATE INDEX idx_executions_started ON failover_executions (started_at DESC);
```

## Event Log & Integrations

```sql
-- Event Log (append-only, CloudEvents-compatible)
CREATE TABLE event_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    ce_type         VARCHAR(200) NOT NULL,
    ce_source       VARCHAR(500) NOT NULL,
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    ce_subject      VARCHAR(500),
    data            JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}'
);

CREATE INDEX idx_event_log_tenant ON event_log (tenant_id, ce_time DESC);
CREATE INDEX idx_event_log_type ON event_log (tenant_id, ce_type);

-- Integrations
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    integration_type VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- AI Insights (graph-aware)
CREATE TABLE ai_insights (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    insight_type    VARCHAR(50) NOT NULL,
    severity        VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    confidence_score NUMERIC(3,2) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    -- Graph-specific insight details
    affected_nodes  JSONB DEFAULT '[]', -- ["node-uuid-1","node-uuid-2"]
    affected_edges  JSONB DEFAULT '[]', -- ["edge-uuid-1"]
    suggested_edges JSONB DEFAULT '[]',
    -- For dependency inference:
    -- [{"source":"node-1","target":"node-2","edge_type":"depends_on","confidence":0.87,"evidence":"network_flow"}]
    details         JSONB NOT NULL DEFAULT '{}',
    model_version   VARCHAR(100),
    acted_on_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_insights_tenant ON ai_insights (tenant_id, status);
```

## Row-Level Security

```sql
ALTER TABLE graph_nodes ENABLE ROW LEVEL SECURITY;
ALTER TABLE graph_edges ENABLE ROW LEVEL SECURITY;
ALTER TABLE dr_plans ENABLE ROW LEVEL SECURITY;
ALTER TABLE failover_executions ENABLE ROW LEVEL SECURITY;
ALTER TABLE event_log ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON graph_nodes
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
-- (repeated for each table)
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 3 | graph_nodes, graph_edges, graph_snapshots |
| Identity | 2 | tenants, users |
| DR Plans & Runbooks | 3 | dr_plans, plan_nodes, runbook_steps |
| Failover Operations | 1 | failover_executions |
| Event Log & Monitoring | 1 | event_log |
| AI & Integrations | 2 | ai_insights, integrations |
| **Total** | **12** | Graph layer replaces separate asset/environment/dependency tables |

---

## Key Design Decisions

1. **Generic graph_nodes replaces asset-specific tables** — environments, assets, service groups, network zones, and external dependencies are all `graph_nodes` with a `node_type` discriminator. This means any entity can participate in the dependency graph without schema changes, and new entity types (e.g., serverless functions, managed databases) are added by defining a new `node_type` value.

2. **Typed, weighted edges** — `graph_edges` support multiple relationship types (`depends_on`, `replicates_to`, `hosts`, `connects_to`) between the same pair of nodes. Edge weights carry semantic meaning that varies by type: confidence score for AI-inferred dependencies, traffic volume for network connections, priority for replication targets.

3. **Graph snapshots for temporal comparison** — before each failover execution, a snapshot of the entire graph is captured. This enables post-incident analysis ("the dependency graph at failover time was different from today's graph — a new dependency was added after the plan was last tested").

4. **Computed failover ordering on DR plans** — when a DR plan is approved, the topological sort of its included nodes is computed from the live graph and stored as `computed_failover_order` JSONB. This avoids recalculating the graph traversal during a live failover (when performance is critical) while the `graph_snapshot_id` records which graph version the ordering was computed from.

5. **Blast radius as a first-class concept** — failover executions include a `blast_radius` field computed by graph traversal at execution time, quantifying the downstream impact. This supports DORA and ISO 22301 requirements for documenting the scope of recovery operations.

6. **Foreign keys bridge graph and relational layers** — `dr_plans.source_environment_node_id` and `runbook_steps.target_node_id` reference `graph_nodes(id)`, maintaining referential integrity between the graph layer and the relational operational tables.

7. **Recursive CTEs for graph traversal** — PostgreSQL's recursive CTE support handles topological sorting, blast radius calculation, and critical path analysis without requiring a separate graph database. For very large graphs (10,000+ nodes), materialised views or periodic batch computation may be needed.

8. **AI insights are graph-aware** — the `ai_insights` table includes `affected_nodes`, `affected_edges`, and `suggested_edges` fields, enabling AI models to propose new dependency edges (from network flow analysis) that operators can review and accept into the live graph.

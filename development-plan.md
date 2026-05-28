# Disaster Recovery Orchestrator — Phased Development Plan

> Project: 180-disaster-recovery-orchestrator · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | TypeScript (Node.js) | API-heavy platform with real-time dashboard needs; strong async I/O for concurrent health checks and replication polling; first-class OpenAPI tooling via tRPC or Fastify; shared types with the frontend reduce impedance mismatch |
| API framework | Fastify + @fastify/swagger | High-performance HTTP framework with native OpenAPI 3.1 schema generation, JSON Schema validation, and plugin architecture for modular route registration — aligns with standards.md requirement to publish OpenAPI spec |
| Database | PostgreSQL 16 | Required by all four data model suggestions; supports JSONB for platform-specific fields, recursive CTEs for dependency graph traversal, Row-Level Security for multi-tenancy, and table partitioning for audit logs |
| ORM / query builder | Drizzle ORM | Type-safe SQL builder that generates migrations from TypeScript schema definitions; avoids the abstraction overhead of Prisma while maintaining full PostgreSQL feature access (JSONB, RLS, CTEs) |
| Task queue | BullMQ (Redis-backed) | Handles async workloads: health check polling, AI inference jobs, failover step execution, notification dispatch; supports delayed jobs, retries, concurrency limits, and progress tracking |
| Cache / pub-sub | Redis 7 | Shared infrastructure with BullMQ; used for real-time dashboard pub/sub (failover progress), session caching, and rate limiting |
| Frontend | React 18 + Vite + Tailwind CSS | Dashboard-heavy application requiring real-time updates (failover progress, health status); React ecosystem has mature graph visualisation libraries (React Flow) for dependency DAGs |
| Real-time updates | Server-Sent Events (SSE) | Simpler than WebSocket for unidirectional server-to-client updates (health status, failover progress); natively supported by browsers; falls back gracefully through proxies |
| Graph visualisation | React Flow | Renders the infrastructure dependency DAG with interactive drag-and-drop; supports custom node/edge types for different asset types and dependency strengths |
| Authentication | Passport.js + JWT (RFC 7519) | OAuth 2.0 / OIDC support for enterprise SSO (Azure AD, Okta, Google); JWT tokens for API authentication — aligns with standards.md OAuth 2.0 requirements |
| AI / LLM integration | Anthropic Claude API (via @anthropic-ai/sdk) | Dependency inference from telemetry, natural language runbook generation from IaC, post-incident report generation, failure prediction — core AI-native differentiators |
| IaC parsing | @cdktf/hcl2json + custom parsers | Parses Terraform HCL and state files to infer infrastructure topology and dependency graphs; CloudFormation and ARM template parsing via JSON/YAML standard parsers |
| Event format | CloudEvents 1.0 (cloudevents-sdk) | Standardised event envelope for all system events; enables native integration with AWS EventBridge, Azure Event Grid, and Kafka — aligns with standards.md specification |
| Testing | Vitest + Supertest + Testcontainers | Vitest for unit tests (fast, ESM-native); Supertest for API integration tests; Testcontainers for PostgreSQL and Redis integration tests with real dependencies |
| Code quality | ESLint + Prettier + tsc --noEmit | Standard TypeScript quality toolchain; strict mode TypeScript for maximum type safety on a security-sensitive platform |
| Containerisation | Docker + docker-compose | Multi-container deployment (API, worker, Redis, PostgreSQL); self-hosted deployment target as specified in README.md |
| Package manager | pnpm | Fast, disk-efficient; workspace support for potential monorepo structure (API + frontend + shared types) |
| API documentation | Scalar (via @scalar/fastify-api-reference) | Modern OpenAPI documentation UI served from the running application; replaces Swagger UI with better DX |

### Data Model Selection

The **Hybrid Relational + JSONB** model (data-model-suggestion-3) is selected as the foundation, with the **Graph-Relational** model (data-model-suggestion-4) providing the dependency graph layer. Rationale:

- The hybrid model's 11-table core keeps the MVP lean while accommodating multi-cloud platform variability via JSONB
- The graph layer (graph_nodes, graph_edges, graph_snapshots) from model 4 is essential for the AI-driven dependency discovery and intelligent failover sequencing differentiators
- Event sourcing (model 2) is deferred — the append-only event_log table provides audit capabilities without the full CQRS complexity
- The normalised model (model 1) is too rigid for the platform variability across VMware, EC2, Azure VM, GKE, and physical servers

Combined table count: ~16 tables (hybrid core + graph layer).

### Project Structure

```
disaster-recovery-orchestrator/
├── package.json
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── drizzle.config.ts
├── packages/
│   └── shared/                       # Shared types & constants
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
│           ├── types/
│           │   ├── tenant.ts
│           │   ├── asset.ts
│           │   ├── dr-plan.ts
│           │   ├── failover.ts
│           │   ├── graph.ts
│           │   ├── compliance.ts
│           │   └── events.ts
│           ├── constants/
│           │   ├── asset-types.ts
│           │   ├── plan-statuses.ts
│           │   └── compliance-frameworks.ts
│           └── index.ts
├── apps/
│   ├── api/                          # Fastify API server
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── server.ts             # Fastify app bootstrap
│   │       ├── config.ts             # Environment config with defaults
│   │       ├── db/
│   │       │   ├── schema.ts         # Drizzle schema definitions
│   │       │   ├── migrations/       # Generated SQL migrations
│   │       │   ├── seed.ts           # Dev seed data
│   │       │   └── client.ts         # Database connection pool
│   │       ├── routes/
│   │       │   ├── auth.ts
│   │       │   ├── tenants.ts
│   │       │   ├── environments.ts
│   │       │   ├── assets.ts
│   │       │   ├── graph.ts
│   │       │   ├── dr-plans.ts
│   │       │   ├── runbook-steps.ts
│   │       │   ├── failover.ts
│   │       │   ├── health-checks.ts
│   │       │   ├── compliance.ts
│   │       │   ├── integrations.ts
│   │       │   ├── ai-insights.ts
│   │       │   └── events.ts
│   │       ├── services/
│   │       │   ├── auth.service.ts
│   │       │   ├── asset.service.ts
│   │       │   ├── graph.service.ts
│   │       │   ├── dr-plan.service.ts
│   │       │   ├── failover.service.ts
│   │       │   ├── health-check.service.ts
│   │       │   ├── compliance.service.ts
│   │       │   ├── notification.service.ts
│   │       │   └── ai/
│   │       │       ├── dependency-inference.ts
│   │       │       ├── failure-prediction.ts
│   │       │       ├── runbook-generator.ts
│   │       │       └── report-generator.ts
│   │       ├── workers/
│   │       │   ├── health-check.worker.ts
│   │       │   ├── failover.worker.ts
│   │       │   ├── ai-inference.worker.ts
│   │       │   └── notification.worker.ts
│   │       ├── plugins/
│   │       │   ├── auth.plugin.ts
│   │       │   ├── tenant.plugin.ts
│   │       │   └── sse.plugin.ts
│   │       ├── adapters/
│   │       │   ├── replication/
│   │       │   │   ├── adapter.interface.ts
│   │       │   │   ├── veeam.adapter.ts
│   │       │   │   ├── zerto.adapter.ts
│   │       │   │   ├── aws-drs.adapter.ts
│   │       │   │   └── azure-asr.adapter.ts
│   │       │   └── iac/
│   │       │       ├── parser.interface.ts
│   │       │       ├── terraform.parser.ts
│   │       │       ├── cloudformation.parser.ts
│   │       │       └── arm-template.parser.ts
│   │       └── lib/
│   │           ├── cloud-events.ts
│   │           ├── graph-algorithms.ts
│   │           ├── rto-rpo-calculator.ts
│   │           └── crypto.ts
│   ├── web/                          # React frontend
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── vite.config.ts
│   │   ├── index.html
│   │   └── src/
│   │       ├── main.tsx
│   │       ├── App.tsx
│   │       ├── api/                  # API client (generated from OpenAPI)
│   │       ├── components/
│   │       │   ├── layout/
│   │       │   ├── dashboard/
│   │       │   ├── assets/
│   │       │   ├── graph/
│   │       │   ├── dr-plans/
│   │       │   ├── failover/
│   │       │   ├── compliance/
│   │       │   └── common/
│   │       ├── hooks/
│   │       ├── pages/
│   │       └── stores/
│   └── worker/                       # BullMQ worker process
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
│           ├── index.ts
│           └── processors/
├── tests/
│   ├── fixtures/
│   │   ├── terraform-state/
│   │   ├── cloudformation-templates/
│   │   └── sample-topologies/
│   ├── unit/
│   ├── integration/
│   └── e2e/
└── docs/
    └── api/
```

---

## Phase 1: Foundation and Core Infrastructure

### Purpose

Establish the project scaffolding, database schema, authentication, multi-tenancy, and the core CRUD API for tenants, users, and environments. After this phase, the application boots, connects to PostgreSQL and Redis, serves an authenticated API, and enforces tenant isolation at the database level. Everything else builds on this foundation.

### Tasks

#### 1.1 — Project Scaffolding and Configuration

**What**: Initialise the pnpm workspace, configure TypeScript, Docker, and environment-based configuration.

**Design**:

```typescript
// packages/shared/src/types/config.ts
export interface AppConfig {
  port: number;                    // default: 3000
  host: string;                    // default: '0.0.0.0'
  nodeEnv: 'development' | 'test' | 'production';
  database: {
    url: string;                   // DATABASE_URL
    poolMin: number;               // default: 2
    poolMax: number;               // default: 10
    ssl: boolean;                  // default: false
  };
  redis: {
    url: string;                   // REDIS_URL
  };
  jwt: {
    secret: string;                // JWT_SECRET
    accessTokenTtl: string;        // default: '15m'
    refreshTokenTtl: string;       // default: '7d'
  };
  log: {
    level: 'debug' | 'info' | 'warn' | 'error'; // default: 'info'
  };
}
```

```typescript
// apps/api/src/config.ts
import { z } from 'zod';

const configSchema = z.object({
  PORT: z.coerce.number().default(3000),
  HOST: z.string().default('0.0.0.0'),
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  JWT_ACCESS_TOKEN_TTL: z.string().default('15m'),
  JWT_REFRESH_TOKEN_TTL: z.string().default('7d'),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

export function loadConfig() {
  return configSchema.parse(process.env);
}
```

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports: ["3000:3000"]
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: dro
      POSTGRES_USER: dro
      POSTGRES_PASSWORD: dro_dev
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dro"]
      interval: 5s
      timeout: 3s
      retries: 5
    volumes:
      - pgdata:/var/lib/postgresql/data
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
volumes:
  pgdata:
```

**Testing**:
- `Unit: loadConfig with all required env vars -> valid AppConfig object`
- `Unit: loadConfig missing DATABASE_URL -> ZodError with path ['DATABASE_URL']`
- `Unit: loadConfig with invalid LOG_LEVEL -> ZodError listing valid options`
- `Integration: docker-compose up -> all services healthy within 30s`
- `Integration: API process starts and binds to configured port`

#### 1.2 — Database Schema and Migrations

**What**: Define the Drizzle ORM schema for the hybrid relational + graph data model and generate initial migrations.

**Design**:

```typescript
// apps/api/src/db/schema.ts
import { pgTable, uuid, varchar, text, boolean, integer, timestamp,
         jsonb, inet, numeric, unique, check, index } from 'drizzle-orm/pg-core';

// === Identity & Multi-Tenancy ===

export const tenants = pgTable('tenants', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  subscriptionTier: varchar('subscription_tier', { length: 50 }).notNull().default('standard'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  email: varchar('email', { length: 320 }).notNull(),
  displayName: varchar('display_name', { length: 255 }).notNull(),
  passwordHash: varchar('password_hash', { length: 255 }),
  authProvider: varchar('auth_provider', { length: 50 }).notNull().default('local'),
  roles: jsonb('roles').notNull().default(['viewer']),
  isActive: boolean('is_active').notNull().default(true),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueTenantEmail: unique().on(table.tenantId, table.email),
  tenantIdx: index('idx_users_tenant').on(table.tenantId),
}));

// === Environments ===

export const environments = pgTable('environments', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  name: varchar('name', { length: 255 }).notNull(),
  environmentType: varchar('environment_type', { length: 50 }).notNull(),
  cloudProvider: varchar('cloud_provider', { length: 50 }),
  region: varchar('region', { length: 100 }),
  jurisdiction: varchar('jurisdiction', { length: 10 }),
  status: varchar('status', { length: 50 }).notNull().default('active'),
  config: jsonb('config').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  tenantIdx: index('idx_environments_tenant').on(table.tenantId),
}));

// === Graph Layer ===

export const graphNodes = pgTable('graph_nodes', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  nodeType: varchar('node_type', { length: 50 }).notNull(),
  name: varchar('name', { length: 255 }).notNull(),
  status: varchar('status', { length: 50 }).notNull().default('active'),
  tier: varchar('tier', { length: 50 }).notNull().default('standard'),
  properties: jsonb('properties').notNull().default({}),
  healthStatus: varchar('health_status', { length: 50 }).notNull().default('unknown'),
  rtoSeconds: integer('rto_seconds'),
  rpoSeconds: integer('rpo_seconds'),
  replication: jsonb('replication').default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  tenantIdx: index('idx_graph_nodes_tenant').on(table.tenantId),
  typeIdx: index('idx_graph_nodes_type').on(table.tenantId, table.nodeType),
  healthIdx: index('idx_graph_nodes_health').on(table.tenantId, table.healthStatus),
}));

export const graphEdges = pgTable('graph_edges', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  sourceNodeId: uuid('source_node_id').notNull().references(() => graphNodes.id, { onDelete: 'cascade' }),
  targetNodeId: uuid('target_node_id').notNull().references(() => graphNodes.id, { onDelete: 'cascade' }),
  edgeType: varchar('edge_type', { length: 50 }).notNull(),
  weight: numeric('weight', { precision: 5, scale: 2 }).notNull().default('1.0'),
  discoveryMethod: varchar('discovery_method', { length: 50 }).notNull().default('manual'),
  properties: jsonb('properties').notNull().default({}),
  isActive: boolean('is_active').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueEdge: unique().on(table.sourceNodeId, table.targetNodeId, table.edgeType),
  sourceIdx: index('idx_graph_edges_source').on(table.sourceNodeId),
  targetIdx: index('idx_graph_edges_target').on(table.targetNodeId),
  tenantIdx: index('idx_graph_edges_tenant').on(table.tenantId),
}));
```

Row-Level Security is applied via a post-migration SQL script:

```sql
-- migrations/0001_rls.sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE environments ENABLE ROW LEVEL SECURITY;
ALTER TABLE graph_nodes ENABLE ROW LEVEL SECURITY;
ALTER TABLE graph_edges ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON environments
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON graph_nodes
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON graph_edges
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

**Testing**:
- `Unit: Drizzle schema compiles without type errors (tsc --noEmit)`
- `Integration (Testcontainers): migrations run against fresh PostgreSQL -> all tables created`
- `Integration (Testcontainers): RLS policy blocks queries without tenant context set`
- `Integration (Testcontainers): RLS policy allows queries with correct tenant context`
- `Integration (Testcontainers): foreign key constraints prevent orphaned records`
- `Integration (Testcontainers): unique constraint on (tenant_id, email) prevents duplicate users`

#### 1.3 — Authentication and User Management API

**What**: Implement JWT-based authentication with local credentials and SSO support via Passport.js, plus user CRUD endpoints.

**Design**:

```typescript
// apps/api/src/services/auth.service.ts
export interface AuthService {
  register(tenantId: string, email: string, password: string, displayName: string): Promise<User>;
  login(email: string, password: string): Promise<{ accessToken: string; refreshToken: string }>;
  refreshToken(refreshToken: string): Promise<{ accessToken: string; refreshToken: string }>;
  verifyToken(token: string): Promise<JwtPayload>;
  changePassword(userId: string, currentPassword: string, newPassword: string): Promise<void>;
}

export interface JwtPayload {
  sub: string;           // user ID
  tenantId: string;
  email: string;
  roles: string[];
  iat: number;
  exp: number;
}
```

API endpoints:

| Method | Path | Request Body | Response | Auth |
|--------|------|-------------|----------|------|
| POST | `/api/v1/auth/register` | `{ tenantSlug, email, password, displayName }` | `{ user, accessToken, refreshToken }` | None |
| POST | `/api/v1/auth/login` | `{ email, password }` | `{ accessToken, refreshToken }` | None |
| POST | `/api/v1/auth/refresh` | `{ refreshToken }` | `{ accessToken, refreshToken }` | None |
| GET | `/api/v1/auth/me` | — | `{ user }` | Bearer JWT |
| GET | `/api/v1/users` | — | `{ users: User[] }` | Bearer JWT (admin) |
| PATCH | `/api/v1/users/:id` | `{ displayName?, roles?, isActive? }` | `{ user }` | Bearer JWT (admin) |

```typescript
// apps/api/src/plugins/tenant.plugin.ts
// Fastify plugin that sets PostgreSQL RLS context on every request
import fp from 'fastify-plugin';

export default fp(async (fastify) => {
  fastify.addHook('preHandler', async (request) => {
    if (request.user?.tenantId) {
      await request.server.db.execute(
        sql`SET app.current_tenant_id = ${request.user.tenantId}`
      );
    }
  });
});
```

**Testing**:
- `Unit: password hashing with bcrypt -> hash differs from plaintext`
- `Unit: JWT token generation -> token contains correct claims (sub, tenantId, roles)`
- `Unit: JWT token verification with expired token -> throws TokenExpiredError`
- `Integration: POST /auth/register with valid data -> 201, user created in DB`
- `Integration: POST /auth/register with duplicate email -> 409 Conflict`
- `Integration: POST /auth/login with correct credentials -> 200, valid JWT returned`
- `Integration: POST /auth/login with wrong password -> 401 Unauthorized`
- `Integration: GET /auth/me with valid token -> 200, user profile returned`
- `Integration: GET /auth/me without token -> 401 Unauthorized`
- `Integration: GET /users as non-admin -> 403 Forbidden`
- `Integration: tenant isolation -> user in tenant A cannot see users in tenant B`

#### 1.4 — Tenant and Environment CRUD API

**What**: Implement CRUD endpoints for tenants and environments with validation and tenant-scoped queries.

**Design**:

| Method | Path | Request Body | Response |
|--------|------|-------------|----------|
| POST | `/api/v1/tenants` | `{ name, slug, subscriptionTier? }` | `{ tenant }` |
| GET | `/api/v1/tenants/:id` | — | `{ tenant }` |
| PATCH | `/api/v1/tenants/:id` | `{ name?, settings? }` | `{ tenant }` |
| POST | `/api/v1/environments` | `{ name, environmentType, cloudProvider?, region?, jurisdiction?, config? }` | `{ environment }` |
| GET | `/api/v1/environments` | query: `?type=production` | `{ environments: Environment[] }` |
| GET | `/api/v1/environments/:id` | — | `{ environment }` |
| PATCH | `/api/v1/environments/:id` | `{ name?, status?, config? }` | `{ environment }` |
| DELETE | `/api/v1/environments/:id` | — | `204 No Content` |

```typescript
// packages/shared/src/types/environment.ts
export type EnvironmentType = 'production' | 'dr_target' | 'staging' | 'cleanroom';
export type CloudProvider = 'aws' | 'azure' | 'gcp' | 'on_premises' | 'hybrid';
export type EnvironmentStatus = 'active' | 'degraded' | 'offline' | 'decommissioned';

export interface Environment {
  id: string;
  tenantId: string;
  name: string;
  environmentType: EnvironmentType;
  cloudProvider: CloudProvider | null;
  region: string | null;
  jurisdiction: string | null;      // ISO 3166-1 alpha-2
  status: EnvironmentStatus;
  config: Record<string, unknown>;  // cloud-provider-specific connection config
  createdAt: string;
  updatedAt: string;
}
```

**Testing**:
- `Unit: environment type validation -> rejects invalid types`
- `Unit: jurisdiction validation -> accepts ISO 3166-1 alpha-2 codes, rejects others`
- `Integration: POST /environments with valid data -> 201, environment created`
- `Integration: GET /environments?type=production -> only production environments returned`
- `Integration: DELETE /environments/:id with linked assets -> 409 Conflict`
- `Integration: tenant isolation -> environments from other tenants are invisible`

#### 1.5 — Event Logging Foundation

**What**: Implement the append-only CloudEvents-compatible event log table and a service for recording system events.

**Design**:

```typescript
// apps/api/src/db/schema.ts (addition)
export const eventLog = pgTable('event_log', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  ceType: varchar('ce_type', { length: 200 }).notNull(),
  ceSource: varchar('ce_source', { length: 500 }).notNull(),
  ceTime: timestamp('ce_time', { withTimezone: true }).notNull().defaultNow(),
  ceSubject: varchar('ce_subject', { length: 500 }),
  data: jsonb('data').notNull(),
  metadata: jsonb('metadata').notNull().default({}),
}, (table) => ({
  tenantTimeIdx: index('idx_event_log_tenant').on(table.tenantId, table.ceTime),
  typeIdx: index('idx_event_log_type').on(table.tenantId, table.ceType, table.ceTime),
}));

// apps/api/src/lib/cloud-events.ts
import { CloudEvent } from 'cloudevents';

export function createEvent(params: {
  tenantId: string;
  type: string;         // e.g. 'com.dro.environment.created'
  source: string;       // e.g. '/api/v1/environments'
  subject?: string;     // e.g. environment ID
  data: Record<string, unknown>;
  userId?: string;
}): CloudEvent<Record<string, unknown>> {
  return new CloudEvent({
    specversion: '1.0',
    type: params.type,
    source: params.source,
    subject: params.subject,
    data: params.data,
    time: new Date().toISOString(),
  });
}
```

**Testing**:
- `Unit: createEvent produces valid CloudEvents 1.0 envelope`
- `Unit: event type follows namespacing convention (com.dro.<domain>.<action>)`
- `Integration: event written to event_log table after environment creation`
- `Integration: event_log is append-only — UPDATE and DELETE fail (enforced by policy)`
- `Integration: events queryable by tenant, type, and time range`

---

## Phase 2: Infrastructure Asset Inventory and Dependency Graph

### Purpose

Implement the asset management layer and the dependency graph — the core data structure that drives failover sequencing. After this phase, users can register infrastructure assets (VMs, databases, services), define dependencies between them, and query the dependency graph for topological ordering, blast radius analysis, and critical path calculations. This is the foundation for all DR plan and failover logic.

### Tasks

#### 2.1 — Asset CRUD API with Platform-Specific JSONB

**What**: Implement asset registration, listing, filtering, and update endpoints with JSONB platform details.

**Design**:

```typescript
// packages/shared/src/types/asset.ts
export type AssetType = 'vm' | 'database' | 'service' | 'container' | 'load_balancer' | 'storage' | 'serverless';
export type AssetPlatform = 'vmware' | 'hyper_v' | 'ec2' | 'azure_vm' | 'gke_pod' | 'rds' | 'aurora' | 'on_prem_physical';
export type AssetTier = 'critical' | 'high' | 'standard' | 'low';
export type HealthStatus = 'healthy' | 'degraded' | 'critical' | 'unknown';

export interface Asset {
  id: string;
  tenantId: string;
  environmentId: string;
  name: string;
  assetType: AssetType;
  platform: AssetPlatform;
  tier: AssetTier;
  rtoSeconds: number | null;
  rpoSeconds: number | null;
  status: string;
  healthStatus: HealthStatus;
  platformDetails: Record<string, unknown>;
  replicationStatus: Record<string, unknown>;
  tags: string[];
  discoveredAt: string | null;
  createdAt: string;
  updatedAt: string;
}

// Platform-specific detail schemas for validation
export interface Ec2PlatformDetails {
  instanceId: string;
  instanceType: string;
  amiId: string;
  securityGroups: string[];
  vpcId: string;
  subnetId: string;
}

export interface VmwarePlatformDetails {
  vcenterId: string;
  vmUuid: string;
  datastore: string;
  cpu: number;
  memoryGb: number;
  diskGb: number;
}

export interface RdsPlatformDetails {
  dbInstanceId: string;
  engine: string;
  engineVersion: string;
  multiAz: boolean;
  storageGb: number;
}
```

API endpoints:

| Method | Path | Request Body | Response |
|--------|------|-------------|----------|
| POST | `/api/v1/assets` | `{ name, assetType, platform, environmentId, tier?, rtoSeconds?, rpoSeconds?, platformDetails?, tags? }` | `{ asset }` |
| GET | `/api/v1/assets` | query: `?type=vm&platform=ec2&tier=critical&tag=pci-scope&health=degraded` | `{ assets: Asset[], total: number }` |
| GET | `/api/v1/assets/:id` | — | `{ asset }` |
| PATCH | `/api/v1/assets/:id` | `{ name?, tier?, rtoSeconds?, platformDetails?, tags? }` | `{ asset }` |
| DELETE | `/api/v1/assets/:id` | — | `204 No Content` |
| GET | `/api/v1/assets/:id/dependencies` | — | `{ upstream: Asset[], downstream: Asset[] }` |

**Testing**:
- `Unit: asset type validation -> rejects invalid types`
- `Unit: EC2 platform details validation -> requires instanceId, instanceType`
- `Integration: POST /assets with EC2 details -> 201, platformDetails stored as JSONB`
- `Integration: GET /assets?tag=pci-scope -> returns only tagged assets (GIN index used)`
- `Integration: GET /assets?health=degraded&tier=critical -> correct filtered results`
- `Integration: DELETE /assets/:id with dependencies -> 409 Conflict (must remove dependencies first)`
- `Integration: tenant isolation -> assets from other tenants invisible`

#### 2.2 — Dependency Graph API (Nodes and Edges)

**What**: Implement the graph node and edge management API for creating, querying, and visualising the dependency graph.

**Design**:

When assets are created (task 2.1), a corresponding graph_node is automatically created via a service-layer hook. Graph edges represent dependencies between nodes.

```typescript
// apps/api/src/services/graph.service.ts
export interface GraphService {
  // Node management
  createNode(tenantId: string, data: CreateGraphNodeInput): Promise<GraphNode>;
  getNode(tenantId: string, nodeId: string): Promise<GraphNode>;
  listNodes(tenantId: string, filters?: GraphNodeFilters): Promise<GraphNode[]>;
  updateNode(tenantId: string, nodeId: string, data: UpdateGraphNodeInput): Promise<GraphNode>;
  deleteNode(tenantId: string, nodeId: string): Promise<void>;

  // Edge management
  addEdge(tenantId: string, data: CreateGraphEdgeInput): Promise<GraphEdge>;
  removeEdge(tenantId: string, edgeId: string): Promise<void>;
  getEdgesForNode(tenantId: string, nodeId: string, direction: 'incoming' | 'outgoing' | 'both'): Promise<GraphEdge[]>;

  // Graph queries
  topologicalSort(tenantId: string, nodeIds: string[]): Promise<GraphNode[]>;
  blastRadius(tenantId: string, nodeId: string, maxDepth?: number): Promise<BlastRadiusResult>;
  criticalPath(tenantId: string, nodeIds: string[]): Promise<CriticalPathResult>;
  detectCycles(tenantId: string, sourceNodeId: string, targetNodeId: string): Promise<boolean>;
}

export interface BlastRadiusResult {
  affectedNodes: Array<{ node: GraphNode; distance: number }>;
  totalAffected: number;
  criticalCount: number;
  affectedServiceGroups: string[];
}

export interface CriticalPathResult {
  path: GraphNode[];
  cumulativeRtoSeconds: number;
  chainLength: number;
}
```

API endpoints:

| Method | Path | Response |
|--------|------|----------|
| GET | `/api/v1/graph/nodes` | `{ nodes: GraphNode[] }` |
| POST | `/api/v1/graph/edges` | `{ edge }` |
| DELETE | `/api/v1/graph/edges/:id` | `204` |
| GET | `/api/v1/graph/topology?nodeIds=a,b,c` | `{ orderedNodes: GraphNode[] }` |
| GET | `/api/v1/graph/blast-radius/:nodeId` | `{ blastRadius: BlastRadiusResult }` |
| GET | `/api/v1/graph/critical-path?nodeIds=a,b,c` | `{ criticalPath: CriticalPathResult }` |

```typescript
// apps/api/src/lib/graph-algorithms.ts

/** Topological sort using Kahn's algorithm via recursive CTE */
export async function topologicalSort(
  db: DrizzleClient,
  tenantId: string,
  nodeIds: string[]
): Promise<{ id: string; name: string; order: number }[]> {
  // Uses the recursive CTE from data-model-suggestion-4
  const result = await db.execute(sql`
    WITH RECURSIVE dependency_order AS (
      SELECT gn.id, gn.name, gn.node_type, 0 AS depth
      FROM graph_nodes gn
      LEFT JOIN graph_edges ge ON ge.source_node_id = gn.id
        AND ge.edge_type = 'depends_on' AND ge.is_active = true
      WHERE gn.tenant_id = ${tenantId}
        AND gn.id = ANY(${nodeIds}::UUID[])
        AND ge.id IS NULL
      UNION ALL
      SELECT gn.id, gn.name, gn.node_type, do.depth + 1
      FROM dependency_order do
      JOIN graph_edges ge ON ge.target_node_id = do.id
        AND ge.edge_type = 'depends_on' AND ge.is_active = true
      JOIN graph_nodes gn ON gn.id = ge.source_node_id
      WHERE gn.id = ANY(${nodeIds}::UUID[])
    )
    SELECT id, name, node_type, MAX(depth) AS failover_order
    FROM dependency_order
    GROUP BY id, name, node_type
    ORDER BY failover_order ASC
  `);
  return result.rows;
}
```

**Testing**:
- `Unit: cycle detection -> returns true for A->B->C->A`
- `Unit: cycle detection -> returns false for A->B->C (no cycle)`
- `Unit: topological sort of A->B->C -> [C, B, A] (leaves first)`
- `Unit: topological sort of diamond graph A->B, A->C, B->D, C->D -> D first, A last`
- `Integration: POST /graph/edges with cycle -> 400 Bad Request with cycle path`
- `Integration: POST /graph/edges with self-reference -> 400 Bad Request`
- `Integration: GET /graph/blast-radius for database node -> returns all dependent app servers`
- `Integration: GET /graph/critical-path -> returns longest dependency chain with cumulative RTO`
- `Integration: GET /graph/topology -> correct topological ordering`
- `Fixture-based: sample 20-node topology from tests/fixtures/sample-topologies/ -> correct ordering`

#### 2.3 — Graph Snapshot Capture

**What**: Implement point-in-time snapshots of the dependency graph for drift detection and pre-failover state capture.

**Design**:

```typescript
// apps/api/src/db/schema.ts (addition)
export const graphSnapshots = pgTable('graph_snapshots', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  name: varchar('name', { length: 255 }).notNull(),
  snapshotType: varchar('snapshot_type', { length: 50 }).notNull(),
  description: text('description'),
  nodeCount: integer('node_count').notNull(),
  edgeCount: integer('edge_count').notNull(),
  nodes: jsonb('nodes').notNull(),
  edges: jsonb('edges').notNull(),
  createdBy: uuid('created_by'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

// Snapshot types: 'scheduled' | 'pre_failover' | 'post_change' | 'manual'
```

| Method | Path | Response |
|--------|------|----------|
| POST | `/api/v1/graph/snapshots` | `{ snapshot }` |
| GET | `/api/v1/graph/snapshots` | `{ snapshots: GraphSnapshot[] }` |
| GET | `/api/v1/graph/snapshots/:id` | `{ snapshot }` |
| GET | `/api/v1/graph/snapshots/:id/diff` | `{ addedNodes, removedNodes, addedEdges, removedEdges }` |

**Testing**:
- `Integration: POST /graph/snapshots -> captures all nodes and edges for tenant`
- `Integration: diff between two snapshots -> correctly identifies added/removed nodes and edges`
- `Integration: snapshot node/edge counts match actual graph state`

---

## Phase 3: DR Plan and Runbook Builder

### Purpose

Implement the DR plan lifecycle (draft, review, approved, active, archived) and the runbook step builder with dependency-ordered sequencing. After this phase, users can create DR plans targeting specific environments, assign assets, define step-by-step runbooks with action parameters, and see the computed failover order derived from the dependency graph. This is the core orchestration data model.

### Tasks

#### 3.1 — DR Plan CRUD and Lifecycle

**What**: Implement DR plan creation, versioning, approval workflow, and lifecycle state management.

**Design**:

```typescript
// packages/shared/src/types/dr-plan.ts
export type PlanType = 'full_failover' | 'partial_failover' | 'failback' | 'test_only';
export type PlanStatus = 'draft' | 'review' | 'approved' | 'active' | 'archived';

export interface DrPlan {
  id: string;
  tenantId: string;
  name: string;
  description: string | null;
  planType: PlanType;
  sourceEnvironmentId: string;
  targetEnvironmentId: string;
  targetRtoSeconds: number;
  targetRpoSeconds: number;
  status: PlanStatus;
  version: number;
  assetIds: string[];
  networkConfig: NetworkConfig;
  complianceConfig: ComplianceConfig;
  computedFailoverOrder: FailoverOrderEntry[] | null;
  computedCriticalPathRto: number | null;
  graphSnapshotId: string | null;
  approvedBy: string | null;
  approvedAt: string | null;
  createdBy: string;
  lastTestedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface NetworkConfig {
  subnetMappings: Array<{ source: string; target: string }>;
  dnsupdates: Array<{ record: string; targetIp: string }>;
  securityGroupMappings: Array<{ source: string; target: string }>;
}

export interface ComplianceConfig {
  frameworks: string[];              // ['iso_22301', 'dora', 'soc2']
  testFrequencyDays: number;         // how often testing is required
  maxRtoSeconds: number;             // compliance-mandated maximum RTO
  requiresApproval: boolean;
}

export interface FailoverOrderEntry {
  nodeId: string;
  name: string;
  order: number;
  estimatedRtoSeconds: number;
}
```

State machine transitions:
```
draft -> review        (submit for review)
review -> approved     (approver grants approval)
review -> draft        (reviewer requests changes)
approved -> active     (plan activated for use)
active -> archived     (plan superseded or retired)
archived -> draft      (plan cloned as new draft version)
```

API endpoints:

| Method | Path | Response |
|--------|------|----------|
| POST | `/api/v1/dr-plans` | `{ plan }` |
| GET | `/api/v1/dr-plans` | `{ plans: DrPlan[], total }` |
| GET | `/api/v1/dr-plans/:id` | `{ plan }` |
| PATCH | `/api/v1/dr-plans/:id` | `{ plan }` |
| POST | `/api/v1/dr-plans/:id/submit-review` | `{ plan }` |
| POST | `/api/v1/dr-plans/:id/approve` | `{ plan }` |
| POST | `/api/v1/dr-plans/:id/activate` | `{ plan }` |
| POST | `/api/v1/dr-plans/:id/archive` | `{ plan }` |
| GET | `/api/v1/dr-plans/:id/failover-order` | `{ order: FailoverOrderEntry[] }` |

On approval, the system:
1. Captures a graph snapshot (snapshot_type = 'plan_approval')
2. Computes topological sort for the plan's assets
3. Computes the critical path RTO
4. Stores results in `computedFailoverOrder` and `computedCriticalPathRto`

**Testing**:
- `Unit: state machine rejects invalid transitions (e.g. draft -> active)`
- `Unit: state machine accepts valid transitions (draft -> review -> approved -> active)`
- `Integration: POST /dr-plans -> 201, plan created with draft status`
- `Integration: POST /dr-plans/:id/approve -> computes failover order from graph`
- `Integration: POST /dr-plans/:id/approve without assets -> 400 (plan must have assets)`
- `Integration: POST /dr-plans/:id/approve by non-approver -> 403 Forbidden`
- `Integration: plan version increments on each approval cycle`
- `Integration: archiving a plan does not delete it, only changes status`

#### 3.2 — Runbook Step Builder

**What**: Implement runbook step creation, ordering, dependency management, and action-type-specific parameter validation.

**Design**:

```typescript
// packages/shared/src/types/runbook-step.ts
export type StepType = 'automated' | 'manual' | 'approval' | 'validation' | 'notification';
export type ActionType = 'failover_vm' | 'configure_dns' | 'run_health_check' | 'run_script' |
  'notify_stakeholders' | 'wait_for_approval' | 'configure_network' | 'start_replication' |
  'stop_replication' | 'validate_service';
export type OnFailure = 'halt' | 'skip' | 'retry' | 'manual_override';

export interface RunbookStep {
  id: string;
  planId: string;
  name: string;
  stepOrder: number;
  stepType: StepType;
  actionType: ActionType;
  targetAssetId: string | null;
  isCritical: boolean;
  onFailure: OnFailure;
  timeoutSeconds: number;
  dependsOn: string[];            // step IDs that must complete first
  actionParams: Record<string, unknown>;
  createdAt: string;
  updatedAt: string;
}

// Action parameter schemas (validated on write)
export interface FailoverVmParams {
  replicationEngine: string;       // 'veeam' | 'zerto' | 'aws_drs' | 'azure_asr'
  targetHost: string;
  powerOn: boolean;
  verifyBoot: boolean;
  bootTimeoutSeconds: number;
}

export interface RunHealthCheckParams {
  checkType: 'http' | 'tcp' | 'script';
  url?: string;
  port?: number;
  script?: string;
  expectedStatus?: number;
  timeoutMs: number;
  retries: number;
}

export interface NotifyStakeholdersParams {
  channels: ('slack' | 'email' | 'teams' | 'webhook')[];
  template: string;               // template name
  recipients: string[];           // team or user identifiers
}
```

| Method | Path | Response |
|--------|------|----------|
| POST | `/api/v1/dr-plans/:planId/steps` | `{ step }` |
| GET | `/api/v1/dr-plans/:planId/steps` | `{ steps: RunbookStep[] }` |
| PATCH | `/api/v1/dr-plans/:planId/steps/:stepId` | `{ step }` |
| DELETE | `/api/v1/dr-plans/:planId/steps/:stepId` | `204` |
| POST | `/api/v1/dr-plans/:planId/steps/reorder` | `{ steps: RunbookStep[] }` |

**Testing**:
- `Unit: failover_vm action params validation -> requires replicationEngine, targetHost`
- `Unit: run_health_check params -> requires checkType and timeoutMs`
- `Unit: step dependency cycle detection -> rejects A depends on B depends on A`
- `Integration: POST /steps -> 201, step created with correct order`
- `Integration: POST /steps with depends_on referencing non-existent step -> 400`
- `Integration: POST /steps on approved plan -> 400 (plan must be in draft or review)`
- `Integration: DELETE /steps/:id -> remaining steps reordered correctly`
- `Integration: POST /steps/reorder -> step_order values updated atomically`
- `Fixture-based: 10-step runbook with dependencies -> correct execution DAG computed`

---

## Phase 4: Failover Execution Engine

### Purpose

Build the failover execution engine that runs DR plans step-by-step, tracks progress in real-time, measures actual RTO/RPO, and provides live status updates via SSE. After this phase, users can initiate test failovers, watch step-by-step progress, and see measured RTO/RPO compared against targets. This is the operational heart of the system.

### Tasks

#### 4.1 — Failover Execution State Machine

**What**: Implement the failover execution lifecycle with step-by-step tracking, retry logic, and RTO/RPO measurement.

**Design**:

```typescript
// packages/shared/src/types/failover.ts
export type ExecutionType = 'test' | 'planned_failover' | 'unplanned_failover' | 'failback';
export type TriggerType = 'manual' | 'scheduled' | 'ai_predicted' | 'alert_triggered';
export type ExecutionStatus = 'pending' | 'running' | 'paused' | 'succeeded' | 'failed' | 'cancelled' | 'rolled_back';
export type StepExecutionStatus = 'pending' | 'running' | 'succeeded' | 'failed' | 'skipped' | 'timed_out';

export interface FailoverExecution {
  id: string;
  tenantId: string;
  planId: string;
  executionType: ExecutionType;
  triggerType: TriggerType;
  status: ExecutionStatus;
  initiatedBy: string | null;
  startedAt: string | null;
  completedAt: string | null;
  actualRtoSeconds: number | null;
  actualRpoSeconds: number | null;
  rtoMet: boolean | null;
  rpoMet: boolean | null;
  stepResults: StepResult[];
  complianceEvidence: ComplianceEvidence;
  failureReason: string | null;
  notes: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface StepResult {
  stepId: string;
  stepName: string;
  status: StepExecutionStatus;
  startedAt: string | null;
  completedAt: string | null;
  durationSeconds: number | null;
  outputLog: string;
  errorMessage: string | null;
  retryCount: number;
}

export interface ComplianceEvidence {
  frameworksTested: string[];
  rtoTarget: number;
  rtoActual: number | null;
  rtoMet: boolean | null;
  rpoTarget: number;
  rpoActual: number | null;
  rpoMet: boolean | null;
  approvedBy: string | null;
  approvedAt: string | null;
  attestationText: string | null;
}
```

```typescript
// apps/api/src/services/failover.service.ts
export interface FailoverService {
  initiate(tenantId: string, planId: string, input: InitiateFailoverInput): Promise<FailoverExecution>;
  pause(tenantId: string, executionId: string): Promise<FailoverExecution>;
  resume(tenantId: string, executionId: string): Promise<FailoverExecution>;
  cancel(tenantId: string, executionId: string): Promise<FailoverExecution>;
  getExecution(tenantId: string, executionId: string): Promise<FailoverExecution>;
  listExecutions(tenantId: string, filters?: ExecutionFilters): Promise<FailoverExecution[]>;
}

export interface InitiateFailoverInput {
  executionType: ExecutionType;
  triggerType: TriggerType;
  notes?: string;
}
```

Execution flow:
1. Validate plan is in `active` status
2. Capture graph snapshot (type = 'pre_failover')
3. Create failover_execution record (status = 'pending')
4. Enqueue failover job in BullMQ
5. Worker picks up job, sets status = 'running', records startedAt
6. Execute steps in dependency order (from computed failover order)
7. For each step: set step status to 'running', execute action, record result
8. On step failure: check onFailure policy (halt/skip/retry/manual_override)
9. On completion: calculate actual RTO (completedAt - startedAt), compare with target
10. Generate compliance evidence snapshot
11. Emit CloudEvents for each state transition

| Method | Path | Response |
|--------|------|----------|
| POST | `/api/v1/failovers` | `{ execution }` |
| GET | `/api/v1/failovers` | `{ executions: FailoverExecution[] }` |
| GET | `/api/v1/failovers/:id` | `{ execution }` |
| POST | `/api/v1/failovers/:id/pause` | `{ execution }` |
| POST | `/api/v1/failovers/:id/resume` | `{ execution }` |
| POST | `/api/v1/failovers/:id/cancel` | `{ execution }` |
| GET | `/api/v1/failovers/:id/stream` | SSE stream of step progress events |

**Testing**:
- `Unit: execution state machine -> pending->running->succeeded valid`
- `Unit: execution state machine -> pending->cancelled valid`
- `Unit: execution state machine -> succeeded->running invalid`
- `Unit: RTO calculation -> completedAt - startedAt in seconds`
- `Unit: RTO met comparison -> actual <= target is true`
- `Integration: POST /failovers -> 201, execution created and job enqueued`
- `Integration: POST /failovers on non-active plan -> 400 Bad Request`
- `Integration: step execution follows dependency order`
- `Integration: step failure with onFailure='halt' -> execution fails, remaining steps skipped`
- `Integration: step failure with onFailure='skip' -> execution continues, step marked skipped`
- `Integration: step failure with onFailure='retry' -> step retried up to maxRetries`
- `Integration: step timeout -> step marked timed_out, onFailure policy applied`
- `Integration: POST /failovers/:id/cancel on running execution -> status becomes cancelled`

#### 4.2 — BullMQ Worker for Failover Step Execution

**What**: Implement the background worker that processes failover jobs and executes individual runbook steps.

**Design**:

```typescript
// apps/worker/src/processors/failover.processor.ts
import { Worker, Job } from 'bullmq';

interface FailoverJobData {
  tenantId: string;
  executionId: string;
  planId: string;
}

export function createFailoverWorker(deps: WorkerDependencies): Worker<FailoverJobData> {
  return new Worker<FailoverJobData>('failover', async (job: Job<FailoverJobData>) => {
    const { tenantId, executionId, planId } = job.data;

    // Load plan and steps in dependency order
    const plan = await deps.drPlanService.getPlan(tenantId, planId);
    const steps = await deps.drPlanService.getStepsInExecutionOrder(tenantId, planId);

    // Execute steps sequentially (respecting dependencies)
    for (const step of steps) {
      // Check if dependencies are satisfied
      const depsReady = await checkDependencies(step.dependsOn, executionId);
      if (!depsReady) {
        await waitForDependencies(step.dependsOn, executionId);
      }

      const result = await executeStep(step, deps);
      await recordStepResult(executionId, step.id, result);

      // Report progress via BullMQ
      await job.updateProgress({
        currentStep: step.name,
        completedSteps: completedCount,
        totalSteps: steps.length,
      });

      if (result.status === 'failed' && step.onFailure === 'halt') {
        throw new Error(`Step "${step.name}" failed: ${result.errorMessage}`);
      }
    }
  }, {
    connection: deps.redis,
    concurrency: 5,       // max concurrent failover executions
    limiter: {
      max: 10,
      duration: 60000,    // rate limit: 10 executions per minute per queue
    },
  });
}
```

Step executor dispatch:

```typescript
// apps/api/src/services/step-executors/index.ts
export interface StepExecutor {
  execute(step: RunbookStep, context: ExecutionContext): Promise<StepResult>;
}

export const stepExecutors: Record<ActionType, StepExecutor> = {
  failover_vm: new FailoverVmExecutor(),
  configure_dns: new ConfigureDnsExecutor(),
  run_health_check: new HealthCheckExecutor(),
  run_script: new ScriptExecutor(),
  notify_stakeholders: new NotificationExecutor(),
  wait_for_approval: new ApprovalExecutor(),
  configure_network: new NetworkConfigExecutor(),
  start_replication: new ReplicationExecutor(),
  stop_replication: new ReplicationExecutor(),
  validate_service: new ValidationExecutor(),
};
```

**Testing**:
- `Unit: failover processor handles empty step list -> execution succeeds immediately`
- `Unit: step executor dispatch -> correct executor called for each action type`
- `Unit: dependency wait logic -> waits only for incomplete dependencies`
- `Integration (mocked executors): 3-step runbook executes in order, all succeed`
- `Integration (mocked executors): step 2 fails with halt -> step 3 never executed`
- `Integration (mocked executors): step timeout -> step marked timed_out after timeoutSeconds`
- `Integration: BullMQ job progress updates emitted for each step`
- `Integration: concurrent failover executions limited by concurrency setting`

#### 4.3 — Server-Sent Events for Real-Time Progress

**What**: Implement SSE endpoint for streaming failover execution progress and health status updates to the frontend.

**Design**:

```typescript
// apps/api/src/plugins/sse.plugin.ts
import fp from 'fastify-plugin';

export default fp(async (fastify) => {
  fastify.get('/api/v1/failovers/:id/stream', {
    schema: {
      params: { type: 'object', properties: { id: { type: 'string', format: 'uuid' } } },
    },
  }, async (request, reply) => {
    const { id } = request.params as { id: string };

    reply.raw.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    });

    // Subscribe to Redis pub/sub channel for this execution
    const channel = `failover:${id}:progress`;
    const subscriber = request.server.redis.duplicate();
    await subscriber.subscribe(channel);

    subscriber.on('message', (ch, message) => {
      reply.raw.write(`event: step-progress\ndata: ${message}\n\n`);
    });

    // Send keepalive every 15 seconds
    const keepalive = setInterval(() => {
      reply.raw.write(': keepalive\n\n');
    }, 15000);

    request.raw.on('close', () => {
      clearInterval(keepalive);
      subscriber.unsubscribe(channel);
      subscriber.quit();
    });
  });
});
```

SSE event types:

```typescript
export type SSEEvent =
  | { event: 'step-started'; data: { stepId: string; stepName: string; order: number } }
  | { event: 'step-completed'; data: { stepId: string; status: StepExecutionStatus; durationSeconds: number } }
  | { event: 'step-failed'; data: { stepId: string; errorMessage: string; onFailure: OnFailure } }
  | { event: 'execution-completed'; data: { status: ExecutionStatus; actualRtoSeconds: number; rtoMet: boolean } }
  | { event: 'execution-failed'; data: { failureReason: string } };
```

**Testing**:
- `Integration: GET /failovers/:id/stream -> receives SSE headers`
- `Integration: failover execution emits step-started event for each step`
- `Integration: failover completion emits execution-completed event`
- `Integration: client disconnect -> subscriber cleaned up, no memory leak`
- `Integration: keepalive comments sent every 15 seconds`

---

## Phase 5: Health Monitoring and RTO/RPO Dashboard

### Purpose

Implement continuous health monitoring of infrastructure assets, replication lag tracking, and an RTO/RPO gap analysis dashboard. After this phase, the system proactively monitors recovery readiness and surfaces SLA gaps before a disaster reveals them — addressing the finding that only 32% of organisations have a readiness dashboard.

### Tasks

#### 5.1 — Health Check Engine

**What**: Implement configurable health check probes that run on a schedule against registered assets, tracking replication lag, service availability, and storage health.

**Design**:

```typescript
// packages/shared/src/types/health-check.ts
export type CheckType = 'replication_lag' | 'service_health' | 'network_latency' | 'storage_health';

export interface HealthCheck {
  id: string;
  tenantId: string;
  assetId: string;
  checkType: CheckType;
  intervalSeconds: number;       // default: 300 (5 minutes)
  isActive: boolean;
  lastCheckAt: string | null;
  lastStatus: HealthStatus | null;
  config: HealthCheckConfig;
  createdAt: string;
  updatedAt: string;
}

export interface HealthCheckConfig {
  // replication_lag: poll replication engine API for current RPO
  replicationEngine?: string;
  maxLagSeconds?: number;        // threshold for 'degraded' status

  // service_health: HTTP health check
  url?: string;
  expectedStatus?: number;
  timeoutMs?: number;

  // network_latency: ping/TCP check
  host?: string;
  port?: number;
  maxLatencyMs?: number;
}

export interface HealthCheckResult {
  id: string;
  healthCheckId: string;
  status: HealthStatus;
  responseTimeMs: number | null;
  details: Record<string, unknown>;
  checkedAt: string;
}
```

```typescript
// apps/worker/src/processors/health-check.processor.ts
// BullMQ repeatable job that runs health checks on schedule

export function createHealthCheckScheduler(deps: WorkerDependencies) {
  // On startup, load all active health checks and create repeatable jobs
  const scheduler = new Worker<HealthCheckJobData>('health-checks', async (job) => {
    const { tenantId, healthCheckId, assetId, checkType, config } = job.data;

    const result = await runHealthCheck(checkType, config, deps);

    // Record result
    await deps.db.insert(healthCheckResults).values({
      healthCheckId,
      status: result.status,
      responseTimeMs: result.responseTimeMs,
      details: result.details,
    });

    // Update asset health status if changed
    await updateAssetHealthIfChanged(tenantId, assetId, result.status, deps);

    // Emit CloudEvent on health status change
    if (result.statusChanged) {
      await deps.eventService.emit({
        tenantId,
        type: 'com.dro.health.status-changed',
        source: '/health-checks',
        subject: assetId,
        data: { previousStatus: result.previousStatus, newStatus: result.status },
      });
    }
  }, { connection: deps.redis });
}
```

| Method | Path | Response |
|--------|------|----------|
| POST | `/api/v1/health-checks` | `{ healthCheck }` |
| GET | `/api/v1/health-checks` | `{ healthChecks: HealthCheck[] }` |
| GET | `/api/v1/health-checks/:id/results` | `{ results: HealthCheckResult[] }` |
| POST | `/api/v1/health-checks/:id/run-now` | `{ result: HealthCheckResult }` |

**Testing**:
- `Unit: HTTP health check with 200 response -> status 'healthy'`
- `Unit: HTTP health check with 500 response -> status 'critical'`
- `Unit: HTTP health check timeout -> status 'critical', responseTimeMs = null`
- `Unit: replication lag below threshold -> status 'healthy'`
- `Unit: replication lag above threshold -> status 'degraded'`
- `Integration: health check creates repeatable BullMQ job on correct interval`
- `Integration: health status change updates asset healthStatus field`
- `Integration: health status change emits CloudEvent`
- `Integration: GET /health-checks/:id/results -> returns chronological results`

#### 5.2 — RTO/RPO Gap Analysis API

**What**: Implement RTO/RPO gap analysis that compares measured recovery performance and current replication lag against declared SLA targets, surfacing gaps per plan and per asset.

**Design**:

```typescript
// apps/api/src/services/rto-rpo.service.ts
export interface RtoRpoGapAnalysis {
  planId: string;
  planName: string;
  targetRtoSeconds: number;
  targetRpoSeconds: number;
  // From last test execution
  lastTestedRtoSeconds: number | null;
  lastTestedRpoSeconds: number | null;
  rtoGapSeconds: number | null;    // actual - target (positive = breach)
  rpoGapSeconds: number | null;
  rtoStatus: 'met' | 'at_risk' | 'breached' | 'untested';
  rpoStatus: 'met' | 'at_risk' | 'breached' | 'untested';
  // From live replication monitoring
  currentMaxReplicationLagSeconds: number | null;
  assetsAtRisk: Array<{
    assetId: string;
    assetName: string;
    currentRpoSeconds: number;
    targetRpoSeconds: number;
    gapSeconds: number;
  }>;
  lastTestedAt: string | null;
  daysSinceLastTest: number | null;
  testOverdue: boolean;            // true if daysSinceLastTest > compliance test frequency
}
```

| Method | Path | Response |
|--------|------|----------|
| GET | `/api/v1/rto-rpo/dashboard` | `{ plans: RtoRpoGapAnalysis[] }` |
| GET | `/api/v1/rto-rpo/plans/:planId` | `{ analysis: RtoRpoGapAnalysis }` |
| GET | `/api/v1/rto-rpo/history?planId=x&from=&to=` | `{ datapoints: RtoRpoDatapoint[] }` |

**Testing**:
- `Unit: rtoStatus = 'breached' when actual > target`
- `Unit: rtoStatus = 'at_risk' when actual > 80% of target`
- `Unit: rtoStatus = 'untested' when no test execution exists`
- `Unit: testOverdue = true when daysSinceLastTest > complianceConfig.testFrequencyDays`
- `Integration: dashboard aggregates gap analysis across all active plans`
- `Integration: assetsAtRisk identifies assets where current replication lag exceeds target RPO`
- `Integration: history endpoint returns time-series data from past failover executions`

#### 5.3 — Alert Rules and Notification Engine

**What**: Implement configurable alert rules that trigger notifications when health checks degrade, replication lag breaches thresholds, or RTO/RPO gaps are detected.

**Design**:

```typescript
// packages/shared/src/types/alert.ts
export type AlertCondition = 'rpo_breach' | 'rto_risk' | 'replication_lag' | 'health_degraded' | 'test_overdue';
export type AlertSeverity = 'info' | 'warning' | 'critical';
export type AlertStatus = 'open' | 'acknowledged' | 'resolved' | 'suppressed';

export interface AlertRule {
  id: string;
  tenantId: string;
  name: string;
  conditionType: AlertCondition;
  thresholdConfig: {
    metric: string;
    operator: 'gt' | 'lt' | 'eq' | 'gte' | 'lte';
    value: number;
    durationSeconds: number;       // condition must persist for this long
  };
  severity: AlertSeverity;
  notificationChannels: Array<{
    type: 'email' | 'slack' | 'teams' | 'webhook';
    target: string;
  }>;
  isActive: boolean;
}
```

| Method | Path | Response |
|--------|------|----------|
| POST | `/api/v1/alert-rules` | `{ alertRule }` |
| GET | `/api/v1/alert-rules` | `{ alertRules: AlertRule[] }` |
| GET | `/api/v1/alerts` | `{ alerts: Alert[] }` |
| POST | `/api/v1/alerts/:id/acknowledge` | `{ alert }` |
| POST | `/api/v1/alerts/:id/resolve` | `{ alert }` |

**Testing**:
- `Unit: RPO breach condition with current lag > threshold -> alert created`
- `Unit: alert deduplication -> same condition does not create duplicate open alerts`
- `Integration: health check degradation triggers alert creation`
- `Integration: alert acknowledgement records user and timestamp`
- `Integration: notification dispatched to configured channels on alert creation`
- `Integration: alert auto-resolves when condition clears`

---

## Phase 6: Compliance Evidence and Audit Trail

### Purpose

Implement automated compliance evidence generation for SOC 2, ISO 22301, DORA, and NIS2 from DR test execution data. After this phase, compliance officers can generate audit-ready evidence packages, track testing schedules against regulatory requirements, and export evidence for auditor review. This addresses the significant gap identified in research: no standard exists for DR evidence packaging, and manual assembly is the norm.

### Tasks

#### 6.1 — Compliance Framework Configuration

**What**: Implement compliance framework registration, requirement tracking, and tenant obligation mapping.

**Design**:

```typescript
// packages/shared/src/constants/compliance-frameworks.ts
export const COMPLIANCE_FRAMEWORKS = {
  iso_22301: {
    code: 'iso_22301',
    name: 'ISO 22301:2019 — Business Continuity Management Systems',
    requirements: [
      { code: '8.4', title: 'Business continuity plans and procedures', testFrequencyDays: 365 },
      { code: '8.5', title: 'Exercise and testing', testFrequencyDays: 365 },
      { code: '9.1', title: 'Monitoring, measurement, analysis and evaluation', testFrequencyDays: 180 },
    ],
  },
  soc2: {
    code: 'soc2',
    name: 'SOC 2 Type II — Availability TSC',
    requirements: [
      { code: 'A1.2', title: 'Recovery procedures tested', testFrequencyDays: 365 },
      { code: 'A1.3', title: 'Recovery plan documented and maintained', testFrequencyDays: 180 },
    ],
  },
  dora: {
    code: 'dora',
    name: 'DORA — EU Digital Operational Resilience Act',
    requirements: [
      { code: 'Art11.6', title: 'DR plan tested at least annually', testFrequencyDays: 365 },
      { code: 'Art11.7', title: 'Recovery within 2 hours', testFrequencyDays: 365 },
      { code: 'Art12.2', title: 'Immutable third backup copy', testFrequencyDays: 365 },
    ],
  },
  nis2: {
    code: 'nis2',
    name: 'NIS2 — EU Directive 2022/2555',
    requirements: [
      { code: 'Art21.2c', title: 'Business continuity and disaster recovery', testFrequencyDays: 365 },
      { code: 'Art21.2d', title: 'Backup management and recovery', testFrequencyDays: 365 },
    ],
  },
  nist_800_34: {
    code: 'nist_800_34',
    name: 'NIST SP 800-34 Rev. 1 — Contingency Planning',
    requirements: [
      { code: 'Step5', title: 'Contingency plan development', testFrequencyDays: 365 },
      { code: 'Step6', title: 'Plan testing, training, and exercises', testFrequencyDays: 365 },
      { code: 'Step7', title: 'Plan maintenance', testFrequencyDays: 180 },
    ],
  },
} as const;
```

| Method | Path | Response |
|--------|------|----------|
| GET | `/api/v1/compliance/frameworks` | `{ frameworks }` |
| POST | `/api/v1/compliance/tenant-obligations` | `{ obligation }` |
| GET | `/api/v1/compliance/tenant-obligations` | `{ obligations }` |
| GET | `/api/v1/compliance/status` | `{ complianceStatus: ComplianceStatusSummary }` |

**Testing**:
- `Unit: built-in frameworks contain correct DORA 2-hour RTO requirement`
- `Integration: tenant can enable/disable compliance frameworks`
- `Integration: compliance status shows 'at_risk' when test is overdue per framework frequency`
- `Integration: compliance status shows 'non_compliant' when no test has ever been run`

#### 6.2 — Automated Evidence Package Generation

**What**: Generate structured compliance evidence packages from DR test execution data, including test logs, timings, approvals, and attestation text.

**Design**:

```typescript
// apps/api/src/services/compliance.service.ts
export interface ComplianceService {
  generateEvidencePackage(
    tenantId: string,
    executionId: string,
    frameworkCode: string
  ): Promise<EvidencePackage>;

  exportEvidencePackage(
    tenantId: string,
    packageId: string,
    format: 'json' | 'pdf' | 'csv'
  ): Promise<Buffer>;

  getComplianceTimeline(
    tenantId: string,
    frameworkCode: string,
    fromDate: string,
    toDate: string
  ): Promise<ComplianceTimelineEntry[]>;
}

export interface EvidencePackage {
  id: string;
  frameworkCode: string;
  frameworkName: string;
  generatedAt: string;
  tenantName: string;
  executionSummary: {
    executionId: string;
    planName: string;
    executionType: string;
    startedAt: string;
    completedAt: string;
    durationSeconds: number;
    status: string;
  };
  rtoRpoEvidence: {
    targetRtoSeconds: number;
    actualRtoSeconds: number;
    rtoMet: boolean;
    targetRpoSeconds: number;
    actualRpoSeconds: number;
    rpoMet: boolean;
  };
  stepEvidence: Array<{
    stepName: string;
    status: string;
    startedAt: string;
    completedAt: string;
    durationSeconds: number;
    outputLog: string;
  }>;
  approvalEvidence: {
    approvedBy: string;
    approvedAt: string;
    approverRole: string;
  } | null;
  attestationText: string;        // Auto-generated summary for auditors
  requirementsMapped: Array<{
    requirementCode: string;
    requirementTitle: string;
    status: 'satisfied' | 'not_satisfied' | 'partially_satisfied';
    evidence: string;
  }>;
}
```

| Method | Path | Response |
|--------|------|----------|
| POST | `/api/v1/compliance/evidence` | `{ evidencePackage }` |
| GET | `/api/v1/compliance/evidence` | `{ packages: EvidencePackage[] }` |
| GET | `/api/v1/compliance/evidence/:id` | `{ package: EvidencePackage }` |
| GET | `/api/v1/compliance/evidence/:id/export?format=pdf` | Binary PDF |

**Testing**:
- `Integration: generate evidence from successful test execution -> all requirements mapped`
- `Integration: DORA evidence includes 2-hour RTO check`
- `Integration: evidence includes step-by-step timing logs`
- `Integration: evidence includes approver information when plan requires approval`
- `Integration: export as JSON produces valid structured document`
- `Integration: compliance timeline shows all test executions for a framework period`
- `Fixture-based: sample execution data -> expected attestation text structure`

---

## Phase 7: Replication Adapter Layer

### Purpose

Implement the vendor-neutral replication adapter interface with concrete adapters for Veeam, Zerto, AWS DRS, and Azure ASR. After this phase, the orchestrator can query replication status, trigger failover operations, and monitor RPO from any supported replication engine through a unified API. This delivers on the "pluggable backends" differentiator from the README.

### Tasks

#### 7.1 — Replication Adapter Interface and Registry

**What**: Define the adapter interface contract and implement an adapter registry that selects the correct adapter based on the replication engine configured for each asset.

**Design**:

```typescript
// apps/api/src/adapters/replication/adapter.interface.ts
export interface ReplicationAdapter {
  readonly engineName: string;

  // Connection
  connect(config: Record<string, unknown>): Promise<void>;
  disconnect(): Promise<void>;
  testConnection(config: Record<string, unknown>): Promise<{ connected: boolean; version?: string }>;

  // Status
  getReplicationStatus(assetId: string): Promise<ReplicationStatus>;
  getReplicationLag(assetId: string): Promise<{ lagSeconds: number; lastSyncAt: string }>;

  // Operations
  startReplication(sourceAssetId: string, targetEnvironmentId: string): Promise<void>;
  stopReplication(assetId: string): Promise<void>;
  triggerFailover(assetId: string, options: FailoverOptions): Promise<FailoverResult>;
  triggerFailback(assetId: string): Promise<FailbackResult>;

  // Point-in-time recovery (for ransomware scenarios)
  listRecoveryPoints(assetId: string, from: string, to: string): Promise<RecoveryPoint[]>;
  recoverToPoint(assetId: string, pointId: string): Promise<RecoveryResult>;
}

export interface ReplicationStatus {
  status: 'active' | 'paused' | 'failed' | 'initialising' | 'not_configured';
  currentRpoSeconds: number | null;
  targetRpoSeconds: number;
  lastSyncAt: string | null;
  journalSizeGb: number | null;
  engineSpecific: Record<string, unknown>;
}

export interface FailoverOptions {
  useMostRecentPoint: boolean;
  recoveryPointId?: string;        // for point-in-time recovery
  powerOnTarget: boolean;
  runPostLaunchScripts: boolean;
}

// apps/api/src/adapters/replication/registry.ts
export class AdapterRegistry {
  private adapters = new Map<string, ReplicationAdapter>();

  register(adapter: ReplicationAdapter): void {
    this.adapters.set(adapter.engineName, adapter);
  }

  get(engineName: string): ReplicationAdapter {
    const adapter = this.adapters.get(engineName);
    if (!adapter) throw new Error(`No adapter registered for engine: ${engineName}`);
    return adapter;
  }
}
```

**Testing**:
- `Unit: registry returns correct adapter for engine name`
- `Unit: registry throws for unregistered engine`
- `Unit: adapter interface type checks compile (tsc --noEmit)`

#### 7.2 — AWS DRS Adapter

**What**: Implement the replication adapter for AWS Elastic Disaster Recovery using the AWS SDK (boto3 equivalent: @aws-sdk/client-drs).

**Design**:

```typescript
// apps/api/src/adapters/replication/aws-drs.adapter.ts
import { DrsClient, DescribeSourceServersCommand, StartRecoveryCommand } from '@aws-sdk/client-drs';

export class AwsDrsAdapter implements ReplicationAdapter {
  readonly engineName = 'aws_drs';
  private client: DrsClient | null = null;

  async connect(config: { region: string; accessKeyId?: string; secretAccessKey?: string; roleArn?: string }) {
    this.client = new DrsClient({
      region: config.region,
      credentials: config.roleArn
        ? /* assume role */ undefined
        : { accessKeyId: config.accessKeyId!, secretAccessKey: config.secretAccessKey! },
    });
  }

  async getReplicationStatus(sourceServerId: string): Promise<ReplicationStatus> {
    const result = await this.client!.send(new DescribeSourceServersCommand({
      filters: { sourceServerIDs: [sourceServerId] },
    }));
    const server = result.items?.[0];
    return {
      status: mapDrsStatus(server?.dataReplicationInfo?.dataReplicationState),
      currentRpoSeconds: server?.dataReplicationInfo?.lagDuration
        ? parseDuration(server.dataReplicationInfo.lagDuration) : null,
      targetRpoSeconds: 0, // continuous replication
      lastSyncAt: server?.dataReplicationInfo?.lastSnapshotDateTime?.toISOString() ?? null,
      journalSizeGb: null,
      engineSpecific: { sourceServerId, lifecycleState: server?.lifeCycle?.state },
    };
  }

  async triggerFailover(sourceServerId: string, options: FailoverOptions): Promise<FailoverResult> {
    const result = await this.client!.send(new StartRecoveryCommand({
      sourceServers: [{ sourceServerID: sourceServerId }],
    }));
    return { jobId: result.job?.jobID, status: 'initiated' };
  }
  // ... remaining methods
}
```

**Testing**:
- `Unit (mocked SDK): getReplicationStatus maps DRS states correctly`
- `Unit (mocked SDK): triggerFailover calls StartRecoveryCommand with correct source server`
- `Unit (mocked SDK): connection test with invalid credentials -> connected: false`
- `Integration (optional, real AWS): list source servers returns valid response`

#### 7.3 — Azure ASR and Veeam/Zerto Adapter Stubs

**What**: Implement the Azure ASR adapter and stub adapters for Veeam VRO and Zerto that define the integration points for future completion.

**Design**:

Azure ASR adapter uses `@azure/arm-recoveryservicesiterecovery`:

```typescript
// apps/api/src/adapters/replication/azure-asr.adapter.ts
export class AzureAsrAdapter implements ReplicationAdapter {
  readonly engineName = 'azure_asr';
  // Uses Azure SDK: SiteRecoveryManagementClient
  // Authentication via DefaultAzureCredential (supports managed identity, service principal)
}
```

Veeam and Zerto adapters implement the interface with REST API calls:

```typescript
// apps/api/src/adapters/replication/veeam.adapter.ts
export class VeeamAdapter implements ReplicationAdapter {
  readonly engineName = 'veeam';
  // REST API calls to Veeam Recovery Orchestrator (HAL-format, OAuth 2.0)
  // Documented at: https://helpcenter.veeam.com/docs/vro/rest/overview.html
}

// apps/api/src/adapters/replication/zerto.adapter.ts
export class ZertoAdapter implements ReplicationAdapter {
  readonly engineName = 'zerto';
  // REST API calls to Zerto ZVM appliance (bearer token auth)
}
```

**Testing**:
- `Unit: Azure ASR adapter maps recovery point types correctly`
- `Unit: Veeam adapter constructs correct HAL-format API requests`
- `Unit: all adapters implement ReplicationAdapter interface (compile-time check)`
- `Integration (mocked HTTP): Veeam adapter OAuth token refresh flow`

---

## Phase 8: ITSM Integrations and Notifications

### Purpose

Implement integrations with ServiceNow, Jira, Slack, Microsoft Teams, and webhook endpoints for approval workflows, incident ticketing, and real-time notifications. After this phase, DR test initiation can require ITSM approval, failover events create incident tickets automatically, and stakeholders receive real-time notifications through their preferred channels.

### Tasks

#### 8.1 — Integration Configuration API

**What**: Implement the integration management API for configuring connections to external ITSM and notification platforms.

**Design**:

```typescript
// packages/shared/src/types/integration.ts
export type IntegrationType = 'servicenow' | 'jira' | 'slack' | 'teams' | 'pagerduty' | 'email' | 'webhook';

export interface Integration {
  id: string;
  tenantId: string;
  integrationType: IntegrationType;
  name: string;
  isActive: boolean;
  config: Record<string, unknown>;   // type-specific, encrypted at rest
  lastSyncAt: string | null;
  createdAt: string;
  updatedAt: string;
}

// Config schemas per type
export interface SlackIntegrationConfig {
  webhookUrl: string;
  channel: string;
  username?: string;
}

export interface ServiceNowIntegrationConfig {
  instanceUrl: string;            // https://xxx.service-now.com
  clientId: string;
  clientSecret: string;           // encrypted
  tableForIncidents: string;      // default: 'incident'
}

export interface JiraIntegrationConfig {
  baseUrl: string;
  email: string;
  apiToken: string;               // encrypted
  projectKey: string;
  issueType: string;              // default: 'Task'
}

export interface WebhookIntegrationConfig {
  url: string;
  method: 'POST' | 'PUT';
  headers: Record<string, string>;
  secret?: string;                // for HMAC signature verification
}
```

| Method | Path | Response |
|--------|------|----------|
| POST | `/api/v1/integrations` | `{ integration }` |
| GET | `/api/v1/integrations` | `{ integrations }` |
| PATCH | `/api/v1/integrations/:id` | `{ integration }` |
| DELETE | `/api/v1/integrations/:id` | `204` |
| POST | `/api/v1/integrations/:id/test` | `{ success: boolean, message: string }` |

**Testing**:
- `Unit: Slack webhook URL validation -> rejects non-Slack URLs`
- `Unit: ServiceNow instance URL validation -> requires HTTPS`
- `Integration: POST /integrations/test with valid Slack webhook -> success: true (mocked)`
- `Integration: sensitive config fields encrypted in database`
- `Integration: PATCH /integrations/:id -> does not expose encrypted secrets in response`

#### 8.2 — Notification Dispatch Service

**What**: Implement the notification dispatch service that sends alerts and status updates through configured integration channels.

**Design**:

```typescript
// apps/api/src/services/notification.service.ts
export interface NotificationService {
  send(tenantId: string, notification: NotificationRequest): Promise<void>;
  sendToChannels(tenantId: string, channels: NotificationChannel[], notification: NotificationRequest): Promise<void>;
}

export interface NotificationRequest {
  type: 'alert' | 'test_result' | 'failover_status' | 'compliance_reminder' | 'approval_request';
  subject: string;
  body: string;
  severity?: AlertSeverity;
  metadata?: Record<string, unknown>;
}

// Notifications dispatched via BullMQ to avoid blocking the API
// apps/worker/src/processors/notification.processor.ts
export class NotificationProcessor {
  private dispatchers: Record<IntegrationType, NotificationDispatcher> = {
    slack: new SlackDispatcher(),
    teams: new TeamsDispatcher(),
    email: new EmailDispatcher(),
    webhook: new WebhookDispatcher(),
    servicenow: new ServiceNowDispatcher(),
    jira: new JiraDispatcher(),
    pagerduty: new PagerDutyDispatcher(),
  };
}
```

**Testing**:
- `Unit (mocked HTTP): Slack dispatcher sends correct payload format`
- `Unit (mocked HTTP): webhook dispatcher includes HMAC signature header when secret configured`
- `Unit (mocked HTTP): ServiceNow dispatcher creates incident record`
- `Integration: notification job enqueued and processed by worker`
- `Integration: failed notification recorded with error message`
- `Integration: notification history queryable by tenant and type`

---

## Phase 9: AI-Native Features

### Purpose

Implement the AI-native differentiators: dependency inference from infrastructure telemetry, failure prediction from health indicators, natural language runbook generation from IaC definitions, and automated post-incident report generation. These features transform the orchestrator from a manual tool into a proactive, intelligent system.

### Tasks

#### 9.1 — AI Dependency Inference from Telemetry

**What**: Implement AI-powered discovery of infrastructure dependencies by analysing network flow data, APM telemetry, and connection patterns to infer which assets depend on which.

**Design**:

```typescript
// apps/api/src/services/ai/dependency-inference.ts
export interface DependencyInferenceService {
  analyseNetworkFlows(
    tenantId: string,
    flowData: NetworkFlowRecord[]
  ): Promise<InferredDependency[]>;

  analyseApmTraces(
    tenantId: string,
    traces: ApmTrace[]
  ): Promise<InferredDependency[]>;

  suggestDependencies(
    tenantId: string
  ): Promise<DependencySuggestion[]>;
}

export interface NetworkFlowRecord {
  sourceIp: string;
  destinationIp: string;
  destinationPort: number;
  protocol: 'tcp' | 'udp';
  bytesTransferred: number;
  flowCount: number;
  firstSeen: string;
  lastSeen: string;
}

export interface InferredDependency {
  upstreamAssetId: string;
  downstreamAssetId: string;
  dependencyType: 'hard' | 'soft' | 'data' | 'network';
  confidenceScore: number;          // 0.0 to 1.0
  evidence: string;                  // human-readable explanation
  networkEvidence: {
    flowCount: number;
    ports: number[];
    protocol: string;
    bytesTransferred: number;
  };
}

export interface DependencySuggestion {
  id: string;
  inferredDependency: InferredDependency;
  status: 'pending' | 'accepted' | 'dismissed';
  suggestedAt: string;
}
```

The inference pipeline:
1. Ingest network flow records (from VPC flow logs, NetFlow, or sFlow)
2. Map source/destination IPs to registered assets
3. Use Claude API to analyse flow patterns and classify dependency types
4. Score confidence based on flow volume, consistency, and port semantics
5. Present suggestions to operators for review before adding to the graph

```typescript
// Claude API prompt for dependency classification
const DEPENDENCY_CLASSIFICATION_PROMPT = `
You are analysing network flow data between infrastructure assets to infer
service dependencies. Given the following flow summary between two assets,
classify the dependency:

Source: {sourceName} ({sourceType}, {sourcePlatform})
Destination: {destName} ({destType}, {destPlatform})
Port: {port}, Protocol: {protocol}
Flow count: {flowCount} over {timePeriod}
Bytes transferred: {bytes}

Classify the dependency type as one of:
- "hard": destination must be available for source to function (e.g. app -> database)
- "soft": source degrades but continues if destination is unavailable (e.g. app -> cache)
- "data": data flows from source to destination but no runtime dependency (e.g. ETL -> warehouse)
- "network": network infrastructure dependency (e.g. app -> load balancer)

Provide a confidence score (0.0 - 1.0) and a one-sentence explanation.
`;
```

| Method | Path | Response |
|--------|------|----------|
| POST | `/api/v1/ai/dependency-inference/analyse` | `{ suggestions: DependencySuggestion[] }` |
| GET | `/api/v1/ai/dependency-inference/suggestions` | `{ suggestions }` |
| POST | `/api/v1/ai/dependency-inference/suggestions/:id/accept` | `{ edge: GraphEdge }` |
| POST | `/api/v1/ai/dependency-inference/suggestions/:id/dismiss` | `204` |

**Testing**:
- `Unit: IP-to-asset mapping resolves registered assets`
- `Unit: flows to port 5432 classified as hard dependency (database)`
- `Unit: flows to port 6379 classified as soft dependency (cache)`
- `Unit: confidence score higher for consistent, high-volume flows`
- `Integration (mocked Claude API): flow data -> structured dependency suggestions returned`
- `Integration: accepting suggestion creates graph edge with discovery_method='ai_inferred'`
- `Integration: dismissed suggestions do not appear in subsequent queries`
- `Fixture-based: sample VPC flow logs -> expected dependency graph`

#### 9.2 — Natural Language Runbook Generation from IaC

**What**: Parse Terraform state files, CloudFormation templates, and ARM templates to automatically generate DR runbook steps using Claude API.

**Design**:

```typescript
// apps/api/src/adapters/iac/parser.interface.ts
export interface IaCParser {
  readonly format: string;
  parse(content: string): Promise<IaCTopology>;
}

export interface IaCTopology {
  resources: IaCResource[];
  dependencies: IaCDependency[];
  outputs: Record<string, unknown>;
}

export interface IaCResource {
  logicalId: string;
  resourceType: string;           // 'aws_instance', 'azurerm_virtual_machine', etc.
  provider: string;
  properties: Record<string, unknown>;
  region?: string;
}

// apps/api/src/services/ai/runbook-generator.ts
export interface RunbookGeneratorService {
  generateFromIaC(
    tenantId: string,
    iacContent: string,
    iacFormat: 'terraform_state' | 'cloudformation' | 'arm_template',
    targetEnvironment: Environment
  ): Promise<GeneratedRunbook>;
}

export interface GeneratedRunbook {
  planName: string;
  planDescription: string;
  steps: GeneratedStep[];
  estimatedTotalRtoSeconds: number;
  warnings: string[];
}

export interface GeneratedStep {
  name: string;
  stepType: StepType;
  actionType: ActionType;
  targetResourceId: string;
  dependsOn: string[];
  actionParams: Record<string, unknown>;
  estimatedDurationSeconds: number;
  rationale: string;               // AI explanation of why this step is needed
}
```

| Method | Path | Response |
|--------|------|----------|
| POST | `/api/v1/ai/runbook-generator/from-iac` | `{ runbook: GeneratedRunbook }` |
| POST | `/api/v1/ai/runbook-generator/apply` | `{ plan: DrPlan }` (creates plan from generated runbook) |

**Testing**:
- `Unit: Terraform state parser extracts aws_instance resources`
- `Unit: Terraform state parser infers dependencies from references`
- `Unit: CloudFormation parser extracts DependsOn relationships`
- `Integration (mocked Claude API): Terraform state with RDS + EC2 -> runbook with DB-first ordering`
- `Integration: generated runbook can be applied to create a valid DR plan`
- `Fixture-based: sample Terraform state -> expected step sequence`

#### 9.3 — Failure Prediction Engine

**What**: Analyse health check trends, replication lag patterns, and system telemetry to predict impending failures and recommend pre-emptive actions.

**Design**:

```typescript
// apps/api/src/services/ai/failure-prediction.ts
export interface FailurePredictionService {
  analyse(tenantId: string): Promise<FailurePrediction[]>;
  getActivePredictions(tenantId: string): Promise<FailurePrediction[]>;
}

export interface FailurePrediction {
  id: string;
  tenantId: string;
  targetAssetId: string;
  predictionType: 'failure_prediction';
  confidenceScore: number;
  severity: AlertSeverity;
  title: string;
  description: string;
  predictedFailureWindow: string;   // e.g. '2h', '24h'
  indicators: string[];             // what telemetry triggered the prediction
  recommendedAction: string;
  modelVersion: string;
  status: 'pending' | 'accepted' | 'dismissed' | 'acted_on';
  createdAt: string;
}
```

The prediction pipeline:
1. BullMQ scheduled job runs analysis every 15 minutes
2. Loads recent health check results (last 24 hours) for all assets
3. Identifies trend patterns (increasing latency, growing replication lag, degrading storage metrics)
4. Sends patterns to Claude API for risk assessment
5. Creates ai_insights records for predictions above confidence threshold (0.7)
6. Triggers alerts for high-severity predictions

**Testing**:
- `Unit: monotonically increasing replication lag over 6 hours -> prediction generated`
- `Unit: stable healthy metrics -> no prediction generated`
- `Unit: predictions below 0.7 confidence filtered out`
- `Integration (mocked Claude API): degrading health trend -> failure prediction with recommended action`
- `Integration: prediction triggers alert via notification service`

#### 9.4 — Post-Incident Report Generation

**What**: Automatically generate post-incident reports from recovery event logs, including root cause analysis, timeline reconstruction, and regulatory attestation text.

**Design**:

```typescript
// apps/api/src/services/ai/report-generator.ts
export interface ReportGeneratorService {
  generatePostIncidentReport(
    tenantId: string,
    executionId: string
  ): Promise<PostIncidentReport>;
}

export interface PostIncidentReport {
  id: string;
  executionId: string;
  generatedAt: string;
  executiveSummary: string;         // 2-3 paragraph board-level summary
  timeline: TimelineEntry[];
  rootCauseAnalysis: string;
  impactAssessment: {
    affectedAssets: number;
    criticalAssetsAffected: number;
    estimatedDowntimeMinutes: number;
    serviceGroupsAffected: string[];
  };
  recoveryPerformance: {
    targetRtoSeconds: number;
    actualRtoSeconds: number;
    rtoMet: boolean;
    targetRpoSeconds: number;
    actualRpoSeconds: number;
    rpoMet: boolean;
    stepsSucceeded: number;
    stepsFailed: number;
    stepsSkipped: number;
  };
  lessonsLearned: string[];
  recommendations: string[];
  regulatoryAttestation: Record<string, string>; // framework -> attestation text
  rawEventLog: CloudEvent[];
}

export interface TimelineEntry {
  timestamp: string;
  event: string;
  actor: string;                    // user or 'system'
  details: string;
}
```

| Method | Path | Response |
|--------|------|----------|
| POST | `/api/v1/ai/reports/post-incident` | `{ report: PostIncidentReport }` |
| GET | `/api/v1/ai/reports` | `{ reports: PostIncidentReport[] }` |
| GET | `/api/v1/ai/reports/:id` | `{ report }` |
| GET | `/api/v1/ai/reports/:id/export?format=pdf` | Binary PDF |

**Testing**:
- `Integration (mocked Claude API): successful execution -> report with positive attestation`
- `Integration (mocked Claude API): failed execution -> report with root cause and recommendations`
- `Integration: timeline correctly ordered from event log`
- `Integration: regulatory attestation generated for each enabled compliance framework`
- `Fixture-based: sample execution events -> expected report structure`

---

## Phase 10: Web Dashboard (Frontend)

### Purpose

Build the React-based web dashboard that visualises the dependency graph, provides real-time failover monitoring, and surfaces RTO/RPO gap analysis. After this phase, operators have a modern, real-time interface for managing DR plans, monitoring recovery readiness, and executing failovers — addressing the competitor criticism that incumbent UIs are "perceived as dated."

### Tasks

#### 10.1 — Application Shell and Authentication UI

**What**: Implement the frontend application shell with routing, authentication flows (login, SSO redirect), tenant context, and navigation layout.

**Design**:

```typescript
// apps/web/src/App.tsx
// React Router v6 with authenticated routes
const router = createBrowserRouter([
  { path: '/login', element: <LoginPage /> },
  { path: '/auth/callback', element: <SSOCallbackPage /> },
  {
    path: '/',
    element: <AuthenticatedLayout />,
    children: [
      { index: true, element: <DashboardPage /> },
      { path: 'assets', element: <AssetsPage /> },
      { path: 'graph', element: <DependencyGraphPage /> },
      { path: 'plans', element: <PlansPage /> },
      { path: 'plans/:id', element: <PlanDetailPage /> },
      { path: 'failovers', element: <FailoversPage /> },
      { path: 'failovers/:id', element: <FailoverDetailPage /> },
      { path: 'compliance', element: <CompliancePage /> },
      { path: 'alerts', element: <AlertsPage /> },
      { path: 'settings', element: <SettingsPage /> },
    ],
  },
]);
```

Navigation structure:
- Dashboard (overview with readiness score)
- Assets (inventory with filtering)
- Dependency Graph (interactive DAG visualisation)
- DR Plans (list, detail, runbook builder)
- Failovers (execution history, live monitoring)
- Compliance (framework status, evidence packages)
- Alerts (active alerts, alert rules)
- Settings (integrations, users, tenant config)

**Testing**:
- `E2E: unauthenticated user visiting / -> redirected to /login`
- `E2E: successful login -> redirected to dashboard`
- `E2E: navigation links render correctly for all routes`
- `Unit: AuthenticatedLayout renders sidebar with correct menu items`

#### 10.2 — Dependency Graph Visualisation

**What**: Implement an interactive dependency graph visualisation using React Flow, with node colouring by health status, edge styling by dependency type, and blast radius highlighting.

**Design**:

```typescript
// apps/web/src/components/graph/DependencyGraph.tsx
import ReactFlow, { Node, Edge, Controls, MiniMap } from 'reactflow';

interface GraphNodeData {
  name: string;
  nodeType: string;
  healthStatus: HealthStatus;
  tier: AssetTier;
  rtoSeconds: number | null;
  rpoSeconds: number | null;
}

// Custom node component with health status indicator
const AssetNode = ({ data }: { data: GraphNodeData }) => (
  <div className={cn('rounded-lg border-2 p-3', healthStatusStyles[data.healthStatus])}>
    <div className="flex items-center gap-2">
      <NodeTypeIcon type={data.nodeType} />
      <span className="font-medium">{data.name}</span>
    </div>
    <div className="text-xs text-gray-500 mt-1">
      RTO: {formatDuration(data.rtoSeconds)} | RPO: {formatDuration(data.rpoSeconds)}
    </div>
    <HealthBadge status={data.healthStatus} />
  </div>
);

// Edge styles by dependency type
const edgeStyles: Record<string, CSSProperties> = {
  depends_on: { stroke: '#3b82f6', strokeWidth: 2 },
  replicates_to: { stroke: '#10b981', strokeWidth: 2, strokeDasharray: '5,5' },
  connects_to: { stroke: '#6b7280', strokeWidth: 1 },
};
```

Features:
- Auto-layout using dagre for dependency DAG
- Click node to see blast radius (affected nodes highlighted)
- Right-click node for context menu (view details, add dependency, run health check)
- Filter by node type, health status, tier
- Legend showing node types and edge types

**Testing**:
- `Unit: GraphNode renders with correct health status colour`
- `Unit: edge styling matches dependency type`
- `E2E: graph page loads with nodes from API`
- `E2E: clicking a node highlights blast radius`
- `E2E: adding a dependency creates edge via API and updates graph`

#### 10.3 — Failover Monitoring Dashboard

**What**: Implement real-time failover monitoring with step-by-step progress, RTO countdown, and live log streaming via SSE.

**Design**:

```typescript
// apps/web/src/components/failover/FailoverMonitor.tsx
// Connects to SSE endpoint and renders live progress

const FailoverMonitor = ({ executionId }: { executionId: string }) => {
  const [steps, setSteps] = useState<StepProgress[]>([]);
  const [rtoElapsed, setRtoElapsed] = useState(0);
  const [status, setStatus] = useState<ExecutionStatus>('pending');

  useEffect(() => {
    const eventSource = new EventSource(`/api/v1/failovers/${executionId}/stream`);
    eventSource.addEventListener('step-started', (e) => { /* update step */ });
    eventSource.addEventListener('step-completed', (e) => { /* update step */ });
    eventSource.addEventListener('execution-completed', (e) => { /* final status */ });
    return () => eventSource.close();
  }, [executionId]);

  return (
    <div>
      <RtoCountdown elapsed={rtoElapsed} target={targetRto} />
      <StepProgressList steps={steps} />
      <LiveLogStream executionId={executionId} />
    </div>
  );
};
```

Components:
- RTO countdown timer (green/yellow/red based on progress vs target)
- Step progress list with status icons (pending/running/succeeded/failed/skipped)
- Live log output for the currently executing step
- Pause/Resume/Cancel buttons

**Testing**:
- `E2E: failover monitor receives SSE events and updates step status`
- `E2E: RTO countdown changes colour when approaching target`
- `E2E: pause button sends pause request and updates UI`
- `Unit: StepProgressList renders correct icons for each status`

#### 10.4 — RTO/RPO Dashboard and Compliance Overview

**What**: Implement the readiness dashboard showing RTO/RPO gap analysis across all plans and the compliance status overview.

**Design**:

Dashboard components:
- Readiness score card (percentage of plans meeting RTO/RPO targets)
- Gap analysis table per plan (target vs actual, last tested)
- Replication lag chart (time-series, last 24 hours)
- Compliance status matrix (framework x requirement, colour-coded)
- Overdue test warnings
- Recent alert summary

**Testing**:
- `E2E: dashboard loads gap analysis data from API`
- `E2E: compliance matrix shows correct status colours`
- `E2E: clicking overdue test warning navigates to plan detail`
- `Unit: readiness score calculation -> (plans meeting targets / total active plans) * 100`

---

## Phase 11: IaC Integration and Drift Detection

### Purpose

Implement infrastructure-as-code parsing to auto-discover assets and dependencies from Terraform state, and detect configuration drift between the live infrastructure graph and the last approved DR plan snapshot. This addresses the gap identified in research: no current DR tool natively parses Terraform state to auto-generate DR dependency graphs.

### Tasks

#### 11.1 — Terraform State Parser

**What**: Parse Terraform state files to extract resources, infer dependencies, and create graph nodes/edges automatically.

**Design**:

```typescript
// apps/api/src/adapters/iac/terraform.parser.ts
export class TerraformParser implements IaCParser {
  readonly format = 'terraform_state';

  async parse(stateJson: string): Promise<IaCTopology> {
    const state = JSON.parse(stateJson);
    const resources: IaCResource[] = [];
    const dependencies: IaCDependency[] = [];

    for (const resource of state.resources) {
      for (const instance of resource.instances) {
        resources.push({
          logicalId: `${resource.type}.${resource.name}`,
          resourceType: resource.type,
          provider: resource.provider,
          properties: instance.attributes,
          region: instance.attributes?.region || instance.attributes?.location,
        });

        // Infer dependencies from references in attributes
        if (instance.dependencies) {
          for (const dep of instance.dependencies) {
            dependencies.push({
              sourceId: `${resource.type}.${resource.name}`,
              targetId: dep,
              type: 'depends_on',
            });
          }
        }
      }
    }

    return { resources, dependencies, outputs: state.outputs || {} };
  }
}

// Resource type to asset type mapping
const TERRAFORM_ASSET_TYPE_MAP: Record<string, { assetType: AssetType; platform: AssetPlatform }> = {
  'aws_instance': { assetType: 'vm', platform: 'ec2' },
  'aws_db_instance': { assetType: 'database', platform: 'rds' },
  'aws_rds_cluster': { assetType: 'database', platform: 'aurora' },
  'aws_lb': { assetType: 'load_balancer', platform: 'ec2' },
  'azurerm_virtual_machine': { assetType: 'vm', platform: 'azure_vm' },
  'azurerm_linux_virtual_machine': { assetType: 'vm', platform: 'azure_vm' },
  'google_compute_instance': { assetType: 'vm', platform: 'gke_pod' },
  // ... additional mappings
};
```

| Method | Path | Response |
|--------|------|----------|
| POST | `/api/v1/iac/import` | `{ import: IaCImport }` (multipart file upload) |
| GET | `/api/v1/iac/imports` | `{ imports: IaCImport[] }` |
| GET | `/api/v1/iac/imports/:id` | `{ import, discoveredAssets, discoveredDependencies }` |
| POST | `/api/v1/iac/imports/:id/apply` | `{ createdNodes, createdEdges }` |

**Testing**:
- `Fixture-based: sample Terraform state with 5 resources -> 5 IaCResource objects`
- `Fixture-based: aws_instance referencing aws_db_instance -> dependency inferred`
- `Unit: resource type mapping -> aws_instance maps to (vm, ec2)`
- `Unit: unknown resource type -> logged as warning, skipped`
- `Integration: import and apply -> graph nodes and edges created`
- `Integration: re-import same state -> change detection via content hash`

#### 11.2 — Configuration Drift Detection

**What**: Compare the current live graph against the graph snapshot from the last approved DR plan to detect configuration drift.

**Design**:

```typescript
// apps/api/src/services/drift-detection.service.ts
export interface DriftDetectionService {
  detectDrift(tenantId: string, planId: string): Promise<DriftReport>;
  scheduleDriftDetection(tenantId: string, planId: string, intervalHours: number): Promise<void>;
}

export interface DriftReport {
  planId: string;
  planName: string;
  snapshotAt: string;              // when the approved snapshot was taken
  analysedAt: string;
  hasDrift: boolean;
  addedNodes: GraphNode[];
  removedNodes: GraphNode[];
  modifiedNodes: Array<{ node: GraphNode; changes: PropertyChange[] }>;
  addedEdges: GraphEdge[];
  removedEdges: GraphEdge[];
  riskAssessment: 'low' | 'medium' | 'high' | 'critical';
  recommendation: string;
}

export interface PropertyChange {
  field: string;
  previousValue: unknown;
  currentValue: unknown;
}
```

| Method | Path | Response |
|--------|------|----------|
| GET | `/api/v1/dr-plans/:planId/drift` | `{ driftReport: DriftReport }` |
| POST | `/api/v1/dr-plans/:planId/drift/schedule` | `{ scheduled: true }` |

**Testing**:
- `Unit: no changes between snapshot and live graph -> hasDrift: false`
- `Unit: new node added since snapshot -> addedNodes contains it`
- `Unit: dependency removed since snapshot -> removedEdges contains it`
- `Integration: drift detection runs against approved plan snapshot`
- `Integration: scheduled drift detection creates repeatable BullMQ job`
- `Integration: high-risk drift triggers alert notification`

---

## Phase 12: Mobile-Responsive UI and Advanced Features

### Purpose

Polish the application with mobile-responsive layouts for field operators, chaos engineering integration hooks, and the AI chatbot interface for natural language DR status queries. This phase delivers the final differentiating features identified in the research: mobile UI for field execution and natural language interaction.

### Tasks

#### 12.1 — Mobile-Responsive Layout

**What**: Adapt the web dashboard for mobile devices, focusing on failover monitoring and step execution for field operators.

**Design**:

Key mobile-optimised views:
- Failover progress view (simplified step list with large tap targets)
- Manual step completion (tap to mark step done with notes)
- Alert acknowledgement (swipe to acknowledge)
- Asset health overview (card-based layout)

Tailwind CSS responsive breakpoints:
```
sm: 640px  — phone landscape
md: 768px  — tablet portrait
lg: 1024px — tablet landscape / small laptop
xl: 1280px — desktop
```

**Testing**:
- `E2E: failover monitor renders correctly at 375px width (iPhone SE)`
- `E2E: manual step completion button is large enough for touch (min 44px)`
- `E2E: navigation collapses to hamburger menu below md breakpoint`
- `E2E: graph visualisation switches to list view on mobile`

#### 12.2 — Chaos Engineering Integration Hooks

**What**: Implement integration with LitmusChaos and Chaos Mesh to gate DR plan readiness on chaos experiment resilience scores.

**Design**:

```typescript
// apps/api/src/adapters/chaos/chaos.interface.ts
export interface ChaosAdapter {
  readonly platform: string;

  listExperiments(namespace?: string): Promise<ChaosExperiment[]>;
  getExperimentResult(experimentId: string): Promise<ChaosResult>;
  triggerExperiment(experimentId: string): Promise<{ runId: string }>;
}

export interface ChaosExperiment {
  id: string;
  name: string;
  type: string;              // 'pod-kill', 'network-partition', 'cpu-stress', etc.
  targetResources: string[];
  schedule?: string;          // cron expression
  lastRunStatus?: string;
  resilienceScore?: number;   // 0-100
}

// DR plan readiness gating
export interface ReadinessGate {
  planId: string;
  gateType: 'chaos_resilience';
  minResilienceScore: number; // e.g. 80
  experimentIds: string[];
  lastEvaluation: {
    score: number;
    met: boolean;
    evaluatedAt: string;
  } | null;
}
```

| Method | Path | Response |
|--------|------|----------|
| GET | `/api/v1/chaos/experiments` | `{ experiments }` |
| POST | `/api/v1/chaos/experiments/:id/trigger` | `{ runId }` |
| POST | `/api/v1/dr-plans/:planId/readiness-gates` | `{ gate }` |
| GET | `/api/v1/dr-plans/:planId/readiness-gates` | `{ gates }` |

**Testing**:
- `Unit (mocked K8s API): LitmusChaos adapter lists experiments`
- `Integration: readiness gate blocks plan activation when resilience score below threshold`
- `Integration: readiness gate passes when resilience score above threshold`

#### 12.3 — AI Chatbot Interface

**What**: Implement a natural language chat interface for querying DR status, plan details, and recovery guidance using Claude API.

**Design**:

```typescript
// apps/api/src/services/ai/chatbot.ts
export interface ChatbotService {
  chat(tenantId: string, userId: string, message: string): Promise<ChatResponse>;
  getChatHistory(tenantId: string, userId: string, limit?: number): Promise<ChatMessage[]>;
}

export interface ChatResponse {
  message: string;
  sources: Array<{ type: string; id: string; name: string }>;  // data sources referenced
  suggestedActions: Array<{ label: string; action: string; params: Record<string, string> }>;
}
```

Supported queries:
- "What is the current recovery readiness for our production environment?"
- "Show me all assets with RPO breach"
- "When was the last DR test for the payment processing plan?"
- "What would be affected if the primary database goes down?"

The chatbot uses Claude API with tool use to query the DRO API endpoints internally.

**Testing**:
- `Integration (mocked Claude API): "show RPO breaches" -> queries assets with RPO breach`
- `Integration (mocked Claude API): "blast radius for db-primary" -> returns affected assets`
- `Integration: chat history persisted and retrievable`
- `Unit: suggested actions are valid API actions`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Asset Inventory & Graph      ─── requires Phase 1
    │
Phase 3: DR Plans & Runbooks          ─── requires Phase 2
    │
Phase 4: Failover Execution Engine    ─── requires Phase 3
    │
    ├── Phase 5: Health Monitoring     ─── requires Phase 2 (can start after Phase 2)
    │       │
    │       └── Phase 7: Replication Adapters  ─── requires Phase 5
    │
    ├── Phase 6: Compliance & Audit    ─── requires Phase 4
    │
    ├── Phase 8: ITSM Integrations     ─── requires Phase 4 (can parallel with Phase 6)
    │
    └── Phase 9: AI-Native Features    ─── requires Phases 2, 4 (can parallel with 6, 7, 8)
         │
Phase 10: Web Dashboard               ─── requires Phases 2, 3, 4, 5 (can start components after Phase 2)
    │
Phase 11: IaC & Drift Detection       ─── requires Phases 2, 3 (can parallel with Phases 5-10)
    │
Phase 12: Mobile & Advanced           ─── requires Phases 9, 10
```

### Parallelism Opportunities

- **Phases 5, 6, 7, 8** can be developed concurrently after Phase 4
- **Phase 9** can begin after Phases 2 and 4 are complete, running in parallel with Phases 6, 7, 8
- **Phase 10** (frontend) can begin component development after Phase 2, with full integration after Phase 5
- **Phase 11** (IaC integration) can run in parallel with Phases 5-10 after Phase 3 is complete

---

## Definition of Done (per phase)

1. All tasks implemented as specified in the Design section.
2. All unit tests pass (`pnpm test:unit`).
3. All integration tests pass (`pnpm test:integration`) — Testcontainers for DB/Redis.
4. ESLint passes with zero errors (`pnpm lint`).
5. Prettier formatting applied (`pnpm format:check`).
6. TypeScript strict mode compilation succeeds (`tsc --noEmit`).
7. Docker build succeeds (`docker build .`).
8. Docker Compose stack starts and passes smoke test.
9. New database migrations created and tested (up and down).
10. New API endpoints appear in auto-generated OpenAPI spec.
11. New configuration options documented in `.env.example`.
12. CloudEvents emitted for all new state transitions.
13. Event log records created for all mutating operations.
14. Row-Level Security enforced on all new tenant-scoped tables.

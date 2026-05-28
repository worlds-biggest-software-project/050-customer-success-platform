# Customer Success Platform -- Development Plan

> Project: 050-customer-success-platform
> Generated: 2026-05-25
> Status: Comprehensive phased plan with technology decisions, task breakdowns, and testing specifications

---

## Table of Contents

1. [Technology Decisions](#technology-decisions)
2. [Project Structure](#project-structure)
3. [Phase Dependency Graph](#phase-dependency-graph)
4. [Phase 1: Foundation & Data Layer](#phase-1-foundation--data-layer)
5. [Phase 2: Integration Engine](#phase-2-integration-engine)
6. [Phase 3: Health Scoring Engine](#phase-3-health-scoring-engine)
7. [Phase 4: Playbook & CTA System](#phase-4-playbook--cta-system)
8. [Phase 5: Portfolio Dashboard & API](#phase-5-portfolio-dashboard--api)
9. [Phase 6: AI Health Narratives](#phase-6-ai-health-narratives)
10. [Phase 7: Churn Prediction ML Pipeline](#phase-7-churn-prediction-ml-pipeline)
11. [Phase 8: Journey Orchestration](#phase-8-journey-orchestration)
12. [Phase 9: QBR Builder Agent](#phase-9-qbr-builder-agent)
13. [Phase 10: Expansion Signal Detection](#phase-10-expansion-signal-detection)
14. [Phase 11: Customer-Facing Portal](#phase-11-customer-facing-portal)
15. [Phase 12: Enterprise Hardening](#phase-12-enterprise-hardening)
16. [Definition of Done](#definition-of-done)

---

## Technology Decisions

### Data Model Selection: Hybrid Relational + JSONB (Suggestion 3) with selective Event Sourcing

**Rationale:** After evaluating all four data model proposals, the Hybrid Relational + JSONB model (Suggestion 3) is selected as the primary architecture, augmented with event sourcing from Suggestion 2 for the health score and CTA domains only.

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Primary data model** | Hybrid Relational + JSONB (Suggestion 3) | Fewest tables (~14), fastest MVP delivery, per-tenant flexibility via JSONB `custom_fields`/`traits` without schema migrations. Mirrors Vitally's proven architecture pattern. |
| **Event sourcing** | Selective -- health scores and CTAs only | Full event sourcing (Suggestion 2) adds complexity disproportionate to MVP needs. However, health score history and CTA audit trails are core domain requirements. We adopt an append-only `health_score_events` table and CTA state-change log without committing to full CQRS across all entities. |
| **Graph layer** | Deferred to Phase 10+ | Graph-relational hybrid (Suggestion 4) adds stakeholder influence and churn contagion capabilities, but these are differentiating features for enterprise -- not MVP requirements. Account hierarchies use `parent_id` FK initially; graph overlay added when enterprise accounts demand it. |
| **Full normalization** | Rejected for MVP | Suggestion 1's 23+ table schema is self-documenting but imposes high migration burden and slows iteration. JSONB columns provide equivalent expressiveness with fewer tables. |

### Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Language** | TypeScript (Node.js 22 LTS) | Unified frontend/backend language; largest ecosystem for SaaS integrations; strong typing via TypeScript reduces JSONB-related bugs |
| **API Framework** | Fastify 5 | 2-3x faster than Express; built-in JSON Schema validation (validates JSONB payloads at API boundary); OpenAPI 3.1 spec auto-generation via `@fastify/swagger` |
| **Database** | PostgreSQL 17 | JSONB with GIN indexes, partitioning, Row-Level Security for multi-tenancy, `gen_random_uuid()` for UUIDs. All four data model suggestions are PostgreSQL-native. |
| **ORM / Query Builder** | Drizzle ORM | Type-safe SQL queries; JSONB operator support; lightweight migration system; avoids Prisma's generated client overhead |
| **Cache** | Redis 7 (Valkey) | Health score caching, rate limiting, job queue backing store, real-time event pub/sub for WebSocket notifications |
| **Job Queue** | BullMQ | Redis-backed; handles playbook step execution, integration sync scheduling, health score recalculation, webhook delivery retries |
| **Frontend** | Next.js 15 (App Router) | React Server Components for dashboard performance; server actions for mutations; deployed on Vercel or self-hosted |
| **UI Components** | shadcn/ui + Tailwind CSS 4 | Accessible, composable components; consistent with modern SaaS UX expectations; no vendor lock-in |
| **Charts** | Recharts 3 | Health score trend lines, portfolio distribution, ARR-at-risk visualization |
| **Real-time** | WebSocket (Socket.io 4) | Real-time health score updates, CTA notifications, churn alerts pushed to CSM dashboard |
| **Auth** | Better Auth (or Clerk) | SSO (OIDC/SAML) for enterprise; email/password for self-serve; multi-tenant session management |
| **AI/LLM** | Anthropic Claude API (via SDK) | Health narrative generation, QBR content drafting, context-aware playbook decisions. Claude chosen for long-context reasoning over account history. Prompt caching for repeated scorecard analysis patterns. |
| **ML Pipeline** | Python 3.12 + scikit-learn + XGBoost | Churn prediction models; feature engineering from product usage events; model training and evaluation. Separate microservice communicating via REST/gRPC. |
| **Object Storage** | S3-compatible (AWS S3, MinIO, R2) | ML model artifacts, QBR deck exports, attachment storage |
| **Webhook Format** | CloudEvents v1.0 (CNCF) | Portable event envelope for outbound webhooks; compatible with AWS EventBridge, Kafka, Knative |
| **API Spec** | OpenAPI 3.1 | Auto-generated from Fastify route schemas; enables SDK generation for integrators |
| **Event Tracking Ingest** | Segment Spec compatible | Accept Segment Track/Identify/Group calls; normalize Amplitude and Mixpanel events to common schema |
| **Testing** | Vitest (unit/integration) + Playwright (E2E) | Vitest for fast TypeScript testing; Playwright for dashboard E2E; pytest for ML pipeline |
| **CI/CD** | GitHub Actions | Lint, test, build, deploy pipeline; preview deployments for PRs |
| **Infrastructure** | Docker Compose (dev), Kubernetes (prod) | Local development parity; production on any K8s provider |

### Standards Compliance

| Standard | Implementation |
|----------|---------------|
| OAuth 2.0 (RFC 6749) | All CRM/product analytics integrations use OAuth 2.0 authorization code flow |
| OpenID Connect | SSO for enterprise tenants via OIDC providers (Okta, Azure AD, Google Workspace) |
| CloudEvents v1.0 | Outbound webhook payloads wrapped in CloudEvents envelope |
| Segment Spec | Event ingestion API accepts Segment-format Track, Identify, Group calls |
| OData v4 (ISO/IEC 20802-1) | Reporting API endpoints expose health score and account data via OData for BI tool compatibility |
| JSON Schema Draft 2020-12 | All JSONB payloads validated against registered JSON Schemas |
| RFC 7807 | API error responses follow Problem Details format |
| GDPR (EU 2016/679) | Data processing agreements, right-to-erasure support, consent tracking |
| SOC 2 Type II | Audit logging, access controls, encryption at rest (Phase 12) |
| OWASP API Security Top 10 | Tenant isolation via RLS, input validation, rate limiting, BOLA prevention |

---

## Project Structure

```
customer-success-platform/
|-- apps/
|   |-- web/                          # Next.js 15 frontend (dashboard)
|   |   |-- src/
|   |   |   |-- app/                  # App Router pages
|   |   |   |   |-- (auth)/           # Login, SSO callback
|   |   |   |   |-- (dashboard)/      # Authenticated layout
|   |   |   |   |   |-- accounts/     # Account list, detail, health
|   |   |   |   |   |-- portfolio/    # Portfolio dashboard
|   |   |   |   |   |-- cockpit/      # CSM action queue
|   |   |   |   |   |-- playbooks/    # Playbook management
|   |   |   |   |   |-- integrations/ # Integration setup
|   |   |   |   |   |-- settings/     # Tenant settings
|   |   |   |-- components/           # Shared UI components
|   |   |   |-- lib/                  # Client utilities
|   |   |-- next.config.ts
|   |   |-- package.json
|   |-- api/                          # Fastify API server
|   |   |-- src/
|   |   |   |-- routes/               # Route handlers by domain
|   |   |   |   |-- accounts/
|   |   |   |   |-- health-scores/
|   |   |   |   |-- ctas/
|   |   |   |   |-- playbooks/
|   |   |   |   |-- integrations/
|   |   |   |   |-- events/           # Segment-compatible ingest
|   |   |   |   |-- webhooks/
|   |   |   |-- services/             # Business logic layer
|   |   |   |   |-- scoring/          # Health score calculation
|   |   |   |   |-- playbook-engine/  # Playbook trigger/execution
|   |   |   |   |-- integration-sync/ # CRM and analytics sync
|   |   |   |   |-- ai/              # LLM narrative generation
|   |   |   |-- db/                   # Drizzle schema, migrations
|   |   |   |-- jobs/                 # BullMQ job processors
|   |   |   |-- middleware/           # Auth, tenant isolation, rate limit
|   |   |   |-- plugins/             # Fastify plugins
|   |   |-- package.json
|   |-- ml/                           # Python ML microservice
|   |   |-- src/
|   |   |   |-- models/              # Churn prediction, expansion detection
|   |   |   |-- features/            # Feature engineering
|   |   |   |-- api/                 # FastAPI endpoints
|   |   |   |-- training/            # Model training scripts
|   |   |-- requirements.txt
|   |   |-- Dockerfile
|-- packages/
|   |-- shared/                       # Shared TypeScript types, constants
|   |   |-- src/
|   |   |   |-- types/               # Domain type definitions
|   |   |   |-- constants/           # Health bands, lifecycle stages, etc.
|   |   |   |-- schemas/             # JSON Schema definitions
|   |-- db/                           # Database package (Drizzle schema)
|   |   |-- src/
|   |   |   |-- schema/              # Table definitions
|   |   |   |-- migrations/          # SQL migration files
|   |   |   |-- seeds/               # Seed data for development
|-- docker/
|   |-- docker-compose.yml            # PostgreSQL, Redis, MinIO
|   |-- docker-compose.prod.yml
|-- .github/
|   |-- workflows/
|   |   |-- ci.yml                   # Lint, test, build
|   |   |-- deploy.yml               # Production deployment
|-- turbo.json                        # Turborepo config
|-- package.json                      # Root workspace
|-- tsconfig.base.json
```

**Monorepo tooling:** Turborepo for build orchestration across `apps/web`, `apps/api`, and `packages/*`. pnpm workspaces for dependency management.

---

## Phase Dependency Graph

```
Phase 1: Foundation & Data Layer
    |
    v
Phase 2: Integration Engine -------+
    |                               |
    v                               v
Phase 3: Health Scoring Engine     Phase 5: Portfolio Dashboard & API
    |                               |
    +------+------+-----------------+
           |      |
           v      v
Phase 4: Playbook & CTA System
           |
     +-----+-----+
     |            |
     v            v
Phase 6: AI      Phase 8: Journey
Narratives       Orchestration
     |            |
     v            v
Phase 7: Churn   Phase 9: QBR
Prediction ML    Builder Agent
     |            |
     +-----+------+
           |
           v
Phase 10: Expansion Signal Detection
           |
           v
Phase 11: Customer-Facing Portal
           |
           v
Phase 12: Enterprise Hardening
```

**Legend:**
- Vertical arrows: strict dependency (must complete before starting)
- Phases 2 and 5 can proceed in parallel after Phase 1
- Phases 6 and 8 can proceed in parallel after Phase 4
- Phases 7 and 9 can proceed in parallel after Phases 6 and 8 respectively

---

## Phase 1: Foundation & Data Layer

**Goal:** Establish the monorepo, database schema, authentication, tenant isolation, and core CRUD operations for accounts and contacts.

**Duration:** 3-4 weeks
**Dependencies:** None (starting phase)

### Task 1.1: Monorepo Scaffold and Tooling

**What:** Initialize the Turborepo monorepo with pnpm workspaces, configure TypeScript, ESLint, Prettier, and create the Docker Compose development environment (PostgreSQL 17, Redis 7).

**Design:**

```typescript
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["^build"] },
    "lint": {},
    "db:migrate": { "cache": false }
  }
}

// package.json (root)
{
  "name": "customer-success-platform",
  "private": true,
  "packageManager": "pnpm@9.15.0",
  "workspaces": ["apps/*", "packages/*"]
}
```

```yaml
# docker/docker-compose.yml
services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: csp_dev
      POSTGRES_USER: csp
      POSTGRES_PASSWORD: csp_dev_password
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports: ["9000:9000", "9001:9001"]
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
volumes:
  pgdata:
```

**Testing:**
- `pnpm install` completes without errors across all workspaces
- `pnpm turbo build` succeeds for all packages
- `docker compose up -d` starts PostgreSQL, Redis, and MinIO; all health checks pass
- `pnpm turbo lint` passes with zero warnings
- TypeScript strict mode enabled; `pnpm turbo typecheck` passes

---

### Task 1.2: Database Schema and Migrations

**What:** Implement the Hybrid Relational + JSONB schema (Suggestion 3) using Drizzle ORM. Create the core tables: `tenants`, `users`, `accounts`, `contacts`, `scorecards`, `integrations`. Configure PostgreSQL Row-Level Security (RLS) for multi-tenant isolation.

**Design:**

```typescript
// packages/db/src/schema/tenants.ts
import { pgTable, uuid, varchar, jsonb, timestamp, boolean } from 'drizzle-orm/pg-core';

export const tenants = pgTable('tenants', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  plan: varchar('plan', { length: 50 }).notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/src/schema/accounts.ts
export const accounts = pgTable('accounts', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id, { onDelete: 'cascade' }),
  externalId: varchar('external_id', { length: 255 }),
  name: varchar('name', { length: 500 }).notNull(),
  domain: varchar('domain', { length: 255 }),
  industry: varchar('industry', { length: 100 }),
  tier: varchar('tier', { length: 50 }).default('standard'),
  lifecycleStage: varchar('lifecycle_stage', { length: 50 }).default('onboarding'),
  arr: numeric('arr', { precision: 12, scale: 2 }),
  mrr: numeric('mrr', { precision: 12, scale: 2 }),
  contractStart: date('contract_start'),
  contractEnd: date('contract_end'),
  renewalDate: date('renewal_date'),
  countryCode: char('country_code', { length: 2 }),
  csmId: uuid('csm_id').references(() => users.id),
  parentId: uuid('parent_id').references(() => accounts.id),
  // Denormalized health score (updated by scoring engine)
  healthScore: numeric('health_score', { precision: 5, scale: 2 }),
  healthBand: varchar('health_band', { length: 10 }),
  healthTrend: varchar('health_trend', { length: 10 }),
  healthNarrative: text('health_narrative'),
  healthScoredAt: timestamp('health_scored_at', { withTimezone: true }),
  // Denormalized churn prediction
  churnProbability: numeric('churn_probability', { precision: 5, scale: 4 }),
  churnRiskBand: varchar('churn_risk_band', { length: 20 }),
  // CRM sync metadata
  crmSource: varchar('crm_source', { length: 50 }),
  crmUrl: text('crm_url'),
  crmData: jsonb('crm_data').notNull().default({}),
  // Tenant-specific custom fields
  customFields: jsonb('custom_fields').notNull().default({}),
  tags: text('tags').array().notNull().default([]),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

```sql
-- RLS policy for tenant isolation (applied via migration)
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_accounts ON accounts
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Repeat for all tenant-scoped tables
```

**Testing:**
- `pnpm db:migrate` applies all migrations successfully to a clean database
- `pnpm db:migrate` is idempotent (running twice produces no errors)
- RLS test: set `app.tenant_id` to tenant A, insert account for tenant B -- query returns zero rows for tenant B's account
- All indexes created: verify with `\di` in psql
- GIN indexes on `custom_fields` and `crm_data` JSONB columns exist and are functional
- `EXPLAIN ANALYZE` on `SELECT * FROM accounts WHERE custom_fields @> '{"region": "EMEA"}'` shows GIN index scan
- Drizzle schema types match the SQL DDL exactly (column types, nullability, defaults)

---

### Task 1.3: Authentication and Tenant Context

**What:** Implement user authentication (email/password + OAuth/SSO), tenant creation flow, and the middleware that sets `app.tenant_id` on every database connection for RLS enforcement.

**Design:**

```typescript
// apps/api/src/middleware/tenant-context.ts
import { FastifyRequest, FastifyReply } from 'fastify';

export async function tenantContextMiddleware(
  request: FastifyRequest,
  reply: FastifyReply
) {
  const tenantId = request.user?.tenantId;
  if (!tenantId) {
    return reply.code(401).send({ error: 'Tenant context required' });
  }
  // Set RLS context for this request's database connection
  await request.server.db.execute(
    sql`SELECT set_config('app.tenant_id', ${tenantId}, true)`
  );
}

// apps/api/src/routes/auth/register.ts
export async function registerRoutes(fastify: FastifyInstance) {
  fastify.post('/auth/register', {
    schema: {
      body: {
        type: 'object',
        required: ['email', 'password', 'name', 'tenantName'],
        properties: {
          email: { type: 'string', format: 'email' },
          password: { type: 'string', minLength: 12 },
          name: { type: 'string', minLength: 1 },
          tenantName: { type: 'string', minLength: 1 },
        },
      },
    },
    handler: async (request, reply) => {
      // Create tenant, then create admin user within that tenant
      const { tenant, user } = await authService.registerTenant(request.body);
      const token = await authService.issueToken(user);
      return reply.code(201).send({ tenant, user, token });
    },
  });
}
```

**Testing:**
- Register new tenant: POST `/auth/register` with valid data returns 201, creates tenant + user in database
- Register with duplicate email: returns 409 Conflict
- Register with weak password (<12 chars): returns 400 with validation error
- Login: POST `/auth/login` with valid credentials returns JWT with `tenantId` claim
- Login with wrong password: returns 401
- Authenticated request: GET `/api/accounts` with valid JWT returns 200
- Unauthenticated request: GET `/api/accounts` without JWT returns 401
- Cross-tenant isolation: User from tenant A cannot see accounts from tenant B (RLS enforced at DB level)
- JWT expiration: expired token returns 401
- Tenant slug uniqueness: creating two tenants with same slug returns 409

---

### Task 1.4: Account and Contact CRUD API

**What:** Implement REST API endpoints for account and contact management. Full CRUD with pagination, filtering, sorting, and search.

**Design:**

```typescript
// apps/api/src/routes/accounts/index.ts
export async function accountRoutes(fastify: FastifyInstance) {
  // List accounts with pagination, filtering, sorting
  fastify.get('/accounts', {
    schema: {
      querystring: {
        type: 'object',
        properties: {
          page: { type: 'integer', minimum: 1, default: 1 },
          pageSize: { type: 'integer', minimum: 1, maximum: 100, default: 25 },
          sort: { type: 'string', enum: ['name', 'arr', 'healthScore', 'renewalDate', 'createdAt'] },
          order: { type: 'string', enum: ['asc', 'desc'], default: 'asc' },
          lifecycleStage: { type: 'string' },
          healthBand: { type: 'string', enum: ['green', 'yellow', 'red'] },
          csmId: { type: 'string', format: 'uuid' },
          search: { type: 'string' },
          tags: { type: 'array', items: { type: 'string' } },
        },
      },
      response: {
        200: {
          type: 'object',
          properties: {
            data: { type: 'array', items: { $ref: 'Account' } },
            pagination: { $ref: 'Pagination' },
          },
        },
      },
    },
    handler: accountController.list,
  });

  // Get single account with contacts and latest health score
  fastify.get('/accounts/:id', { handler: accountController.get });

  // Create account
  fastify.post('/accounts', { handler: accountController.create });

  // Update account
  fastify.patch('/accounts/:id', { handler: accountController.update });

  // Delete account (soft delete via lifecycle_stage = 'archived')
  fastify.delete('/accounts/:id', { handler: accountController.archive });
}
```

**Testing:**
- Create account: POST `/api/accounts` with valid body returns 201 with UUID `id`
- Create account with missing `name`: returns 400 validation error
- Get account: GET `/api/accounts/:id` returns account with contacts array
- Get nonexistent account: returns 404
- List accounts: GET `/api/accounts` returns paginated response with `data` and `pagination`
- List accounts with `healthBand=red` filter: returns only red-band accounts
- List accounts with `search=acme`: returns accounts matching name search
- List accounts sorted by `arr desc`: returns accounts in descending ARR order
- Update account custom fields: PATCH `/api/accounts/:id` with `{ "customFields": { "region": "EMEA" } }` merges into existing custom fields
- Archive account: DELETE `/api/accounts/:id` sets `lifecycleStage` to `archived`
- Contact CRUD: full parallel test suite for contacts nested under accounts
- Pagination: request page 2 with pageSize 10 returns correct offset
- Tags filter: `tags=enterprise,at-risk` returns accounts with both tags

---

### Task 1.5: Shared Types and Constants Package

**What:** Create the `packages/shared` package with TypeScript type definitions, constants (lifecycle stages, health bands, CTA types, playbook categories), and JSON Schema definitions used by both frontend and API.

**Design:**

```typescript
// packages/shared/src/constants/health.ts
export const HEALTH_BANDS = ['green', 'yellow', 'red'] as const;
export type HealthBand = typeof HEALTH_BANDS[number];

export const HEALTH_TRENDS = ['improving', 'stable', 'declining'] as const;
export type HealthTrend = typeof HEALTH_TRENDS[number];

export const LIFECYCLE_STAGES = [
  'onboarding', 'adopting', 'growing', 'renewing', 'churned', 'archived'
] as const;
export type LifecycleStage = typeof LIFECYCLE_STAGES[number];

export const CONTACT_ROLES = [
  'champion', 'decision_maker', 'end_user', 'executive_sponsor', 'detractor'
] as const;
export type ContactRole = typeof CONTACT_ROLES[number];

export const CTA_TYPES = ['risk', 'renewal', 'expansion', 'onboarding', 'custom'] as const;
export type CtaType = typeof CTA_TYPES[number];

export const PLAYBOOK_CATEGORIES = [
  'onboarding', 'risk', 'renewal', 'expansion', 'escalation'
] as const;
export type PlaybookCategory = typeof PLAYBOOK_CATEGORIES[number];

// packages/shared/src/types/account.ts
export interface Account {
  id: string;
  tenantId: string;
  externalId: string | null;
  name: string;
  domain: string | null;
  industry: string | null;
  tier: 'enterprise' | 'mid-market' | 'smb' | 'startup' | 'standard';
  lifecycleStage: LifecycleStage;
  arr: number | null;
  mrr: number | null;
  renewalDate: string | null;
  csmId: string | null;
  healthScore: number | null;
  healthBand: HealthBand | null;
  healthTrend: HealthTrend | null;
  healthNarrative: string | null;
  customFields: Record<string, unknown>;
  tags: string[];
  createdAt: string;
  updatedAt: string;
}
```

**Testing:**
- Types compile without errors in both `apps/api` and `apps/web`
- Constants are importable from `@csp/shared` workspace package
- JSON Schema validation: `ajv.validate(accountSchema, validPayload)` returns true
- JSON Schema validation: `ajv.validate(accountSchema, invalidPayload)` returns false with correct error path
- All enum constants used in API route schemas match the shared constants (no drift)

---

## Phase 2: Integration Engine

**Goal:** Build the integration framework for CRM (Salesforce, HubSpot) and product analytics (Segment-compatible event ingestion) data sources. Bidirectional sync for CRM; inbound-only for product events.

**Duration:** 4-5 weeks
**Dependencies:** Phase 1

### Task 2.1: Integration Framework and OAuth Flow

**What:** Build the generic integration lifecycle: connect, configure, sync, disconnect. Implement OAuth 2.0 authorization code flow for Salesforce and HubSpot. Store credentials encrypted in the `integrations` table.

**Design:**

```typescript
// apps/api/src/services/integration-sync/base-provider.ts
export abstract class IntegrationProvider {
  abstract readonly provider: string;

  abstract getAuthorizationUrl(tenantId: string, redirectUri: string): string;
  abstract exchangeCodeForTokens(code: string, redirectUri: string): Promise<OAuthTokens>;
  abstract refreshAccessToken(refreshToken: string): Promise<OAuthTokens>;
  abstract testConnection(credentials: Record<string, unknown>): Promise<boolean>;

  // Sync methods
  abstract syncAccountsInbound(integration: Integration): Promise<SyncResult>;
  abstract syncAccountsOutbound(integration: Integration, accounts: Account[]): Promise<SyncResult>;
  abstract syncContactsInbound(integration: Integration): Promise<SyncResult>;
}

// apps/api/src/services/integration-sync/salesforce-provider.ts
export class SalesforceProvider extends IntegrationProvider {
  readonly provider = 'salesforce';

  getAuthorizationUrl(tenantId: string, redirectUri: string): string {
    const params = new URLSearchParams({
      response_type: 'code',
      client_id: env.SALESFORCE_CLIENT_ID,
      redirect_uri: redirectUri,
      state: tenantId,  // CSRF protection
      scope: 'api refresh_token',
    });
    return `https://login.salesforce.com/services/oauth2/authorize?${params}`;
  }

  async syncAccountsInbound(integration: Integration): Promise<SyncResult> {
    const sf = new SalesforceClient(integration.credentials);
    const sfAccounts = await sf.query(
      `SELECT Id, Name, Industry, AnnualRevenue, Website, OwnerId
       FROM Account WHERE Type = 'Customer' AND LastModifiedDate > ${integration.lastSyncAt}`
    );
    // Map Salesforce fields to account schema using integration.config.fieldMapping
    // Upsert accounts by externalId
    // Return { synced: number, failed: number, errors: [] }
  }
}
```

```typescript
// apps/api/src/routes/integrations/oauth-callback.ts
fastify.get('/integrations/oauth/callback', {
  handler: async (request, reply) => {
    const { code, state: tenantId } = request.query;
    const provider = getProvider(request.query.provider);
    const tokens = await provider.exchangeCodeForTokens(code, redirectUri);
    await integrationService.saveCredentials(tenantId, provider.provider, tokens);
    // Redirect back to integration settings page
    return reply.redirect(`/settings/integrations?connected=${provider.provider}`);
  },
});
```

**Testing:**
- OAuth flow: initiate Salesforce connection returns redirect URL with correct `client_id` and `scope`
- Token exchange: mock Salesforce token endpoint; verify `access_token` and `refresh_token` stored (encrypted)
- Token refresh: expired token triggers automatic refresh before sync
- Connection test: `testConnection()` with valid credentials returns true
- Connection test: `testConnection()` with invalid credentials returns false
- Integration status transitions: `active` -> `paused` -> `active` -> `disconnected`
- Credentials encryption: raw `access_token` is not readable from database without decryption key

---

### Task 2.2: Salesforce Bidirectional Sync

**What:** Implement full bidirectional sync between Salesforce and the CS platform. Inbound: pull accounts, contacts, opportunities. Outbound: push health scores, health bands, and CTA flags back to Salesforce custom fields.

**Design:**

```typescript
// apps/api/src/services/integration-sync/salesforce-provider.ts
async syncAccountsOutbound(integration: Integration, accounts: Account[]): Promise<SyncResult> {
  const sf = new SalesforceClient(integration.credentials);
  const fieldMapping = integration.config.outboundFieldMapping;
  // Default outbound mapping:
  // health_score -> CSP_Health_Score__c
  // health_band -> CSP_Health_Band__c
  // churn_risk_band -> CSP_Churn_Risk__c
  // renewal_date -> CSP_Renewal_Date__c

  const batch = accounts.map(account => ({
    Id: account.externalId,
    [fieldMapping.healthScore]: account.healthScore,
    [fieldMapping.healthBand]: account.healthBand,
    [fieldMapping.churnRisk]: account.churnRiskBand,
  }));

  const result = await sf.sobject('Account').update(batch);
  return { synced: result.successes, failed: result.failures, errors: result.errors };
}

// apps/api/src/jobs/sync-salesforce.ts
// BullMQ job processor: runs every 15 minutes (configurable per tenant)
export async function processSalesforceSync(job: Job<SyncJobData>) {
  const integration = await integrationService.get(job.data.integrationId);
  const provider = new SalesforceProvider();

  // Inbound sync
  const inboundResult = await provider.syncAccountsInbound(integration);
  const contactResult = await provider.syncContactsInbound(integration);

  // Outbound sync (push health scores to Salesforce)
  const accountsWithScores = await accountService.getAccountsWithHealthScores(integration.tenantId);
  const outboundResult = await provider.syncAccountsOutbound(integration, accountsWithScores);

  // Log sync results
  await syncLogService.record(integration.id, 'bidirectional', {
    inbound: inboundResult,
    contacts: contactResult,
    outbound: outboundResult,
  });

  // Update last_sync_at
  await integrationService.updateSyncTimestamp(integration.id);
}
```

**Testing:**
- Inbound sync: mock Salesforce SOQL response; verify accounts created/updated in database with correct field mapping
- Inbound sync deduplication: syncing same Salesforce account twice does not create duplicates (upsert by `externalId`)
- Outbound sync: verify Salesforce API receives correct custom field values for health score and health band
- Outbound sync partial failure: 8 of 10 accounts succeed; 2 fail -- verify sync log records both success and failure counts
- Field mapping: custom field mapping in `integration.config` correctly maps `Industry` -> `industry`, `AnnualRevenue` -> `arr`
- Incremental sync: second sync only pulls records modified since `lastSyncAt`
- Sync scheduling: BullMQ job runs every 15 minutes; verify job is enqueued on integration activation
- Rate limiting: Salesforce API rate limit (100 calls/min) is respected; backoff on 429 response
- Sync log: each sync run creates a `sync_log` entry with direction, record counts, and timing

---

### Task 2.3: HubSpot Bidirectional Sync

**What:** Implement the same bidirectional sync pattern for HubSpot CRM. Inbound: pull companies, contacts, deals. Outbound: push health scores to custom properties.

**Design:**

```typescript
// apps/api/src/services/integration-sync/hubspot-provider.ts
export class HubSpotProvider extends IntegrationProvider {
  readonly provider = 'hubspot';

  async syncAccountsInbound(integration: Integration): Promise<SyncResult> {
    const hubspot = new HubSpotClient(integration.credentials);
    const companies = await hubspot.crm.companies.getAll({
      properties: ['name', 'domain', 'industry', 'annualrevenue', 'lifecyclestage'],
      after: integration.lastSyncAt,
    });
    // Map HubSpot company properties to account schema
    // Upsert by externalId (HubSpot company ID)
  }

  async syncAccountsOutbound(integration: Integration, accounts: Account[]): Promise<SyncResult> {
    const hubspot = new HubSpotClient(integration.credentials);
    // Create custom properties in HubSpot if they don't exist:
    // csp_health_score (number), csp_health_band (enumeration), csp_churn_risk (enumeration)
    const batch = accounts.map(account => ({
      id: account.externalId,
      properties: {
        csp_health_score: String(account.healthScore),
        csp_health_band: account.healthBand,
        csp_churn_risk: account.churnRiskBand,
      },
    }));
    return hubspot.crm.companies.batchApi.update({ inputs: batch });
  }
}
```

**Testing:**
- Same test matrix as Salesforce sync (Task 2.2) adapted for HubSpot API shapes
- HubSpot custom property creation: verify custom properties are created on first outbound sync
- HubSpot OAuth: verify OAuth 2.0 flow (HubSpot requires OAuth, not API keys)
- HubSpot pagination: verify all companies are pulled when count exceeds single page (100 per page)
- HubSpot association sync: contacts correctly associated with companies

---

### Task 2.4: Segment-Compatible Event Ingestion API

**What:** Build a Segment Spec-compatible HTTP API that accepts Track, Identify, and Group calls from product analytics sources. Normalize events from Segment, Amplitude, and Mixpanel into a common schema. Store in the partitioned `events` table.

**Design:**

```typescript
// apps/api/src/routes/events/ingest.ts
fastify.post('/events/track', {
  schema: {
    body: {
      type: 'object',
      required: ['userId', 'event'],
      properties: {
        userId: { type: 'string' },
        anonymousId: { type: 'string' },
        event: { type: 'string' },
        properties: { type: 'object' },
        timestamp: { type: 'string', format: 'date-time' },
        context: { type: 'object' },
      },
    },
  },
  handler: async (request, reply) => {
    // Authenticate via write key in Authorization header (Basic auth)
    const writeKey = extractWriteKey(request);
    const integration = await integrationService.getByWriteKey(writeKey);

    // Resolve userId to contact and account
    const contact = await contactService.findByExternalId(integration.tenantId, request.body.userId);

    // Insert event
    await eventService.ingest({
      tenantId: integration.tenantId,
      accountId: contact?.accountId,
      contactId: contact?.id,
      eventType: 'track',
      eventName: request.body.event,
      source: 'segment',
      properties: request.body.properties ?? {},
      context: request.body.context ?? {},
      sourceEventId: request.body.messageId,
      occurredAt: request.body.timestamp ?? new Date(),
    });

    return reply.code(200).send({ success: true });
  },
});

// Batch endpoint for high-throughput ingestion
fastify.post('/events/batch', {
  handler: async (request, reply) => {
    const { batch } = request.body;  // Array of Track/Identify/Group calls
    await eventService.ingestBatch(batch.map(normalizeSegmentEvent));
    return reply.code(200).send({ success: true });
  },
});
```

**Testing:**
- Single Track event: POST `/events/track` with valid payload returns 200 and event appears in `events` table
- Batch ingestion: POST `/events/batch` with 50 events returns 200; all 50 events in database
- Deduplication: sending same event (same `messageId`) twice results in one row
- Write key auth: invalid write key returns 401
- Missing required fields: omitting `event` field returns 400 with validation error
- User resolution: `userId` correctly mapped to `contact_id` and `account_id`
- Unresolved user: event with unknown `userId` is still stored (with null `contact_id`)
- Partition verification: events with `occurred_at` in May 2026 are in the `events_2026_05` partition
- High throughput: batch of 2000 events ingested within 5 seconds (benchmark test)
- Amplitude format: `/events/amplitude` endpoint accepts Amplitude HTTP API format and normalizes to common schema
- Mixpanel format: `/events/mixpanel` endpoint accepts Mixpanel Import API format

---

## Phase 3: Health Scoring Engine

**Goal:** Build the configurable health scoring system with weighted multi-dimensional scoring, real-time recalculation on incoming signals, and health score history tracking.

**Duration:** 3-4 weeks
**Dependencies:** Phase 1, Phase 2 (needs events and CRM data)

### Task 3.1: Scorecard Configuration API

**What:** Build CRUD API for scorecard management. Each tenant can create multiple scorecards with different measures and weights. One scorecard is marked as default.

**Design:**

```typescript
// apps/api/src/services/scoring/scorecard-service.ts
export class ScorecardService {
  async create(tenantId: string, input: CreateScorecardInput): Promise<Scorecard> {
    // Validate: weights must sum to 1.0
    const totalWeight = input.measures.reduce((sum, m) => sum + m.weight, 0);
    if (Math.abs(totalWeight - 1.0) > 0.001) {
      throw new ValidationError('Measure weights must sum to 1.0');
    }
    // Validate: each measure has valid scoring method
    for (const measure of input.measures) {
      if (!['rule', 'ml_model', 'manual'].includes(measure.method)) {
        throw new ValidationError(`Invalid scoring method: ${measure.method}`);
      }
    }
    return db.insert(scorecards).values({
      tenantId,
      name: input.name,
      measures: input.measures,
      isDefault: input.isDefault ?? false,
    });
  }
}

// Example scorecard measures JSONB:
// [
//   { "name": "usage", "weight": 0.30, "method": "rule",
//     "thresholds": { "green": 80, "yellow": 50, "red": 0 },
//     "config": { "metric": "dau_as_pct_of_licensed_seats", "lookback_days": 14 } },
//   { "name": "engagement", "weight": 0.25, "method": "rule",
//     "thresholds": { "green": 75, "yellow": 40, "red": 0 },
//     "config": { "metrics": ["logins_per_week", "features_used", "api_calls"] } },
//   { "name": "support", "weight": 0.20, "method": "rule",
//     "thresholds": { "green": 85, "yellow": 60, "red": 0 },
//     "config": { "metrics": ["open_ticket_count", "avg_resolution_hours", "escalation_count"] } },
//   { "name": "nps", "weight": 0.15, "method": "rule",
//     "thresholds": { "green": 70, "yellow": 40, "red": 0 },
//     "config": { "metric": "latest_nps_score", "decay_days": 180 } },
//   { "name": "financial", "weight": 0.10, "method": "rule",
//     "thresholds": { "green": 90, "yellow": 70, "red": 0 },
//     "config": { "metrics": ["arr_growth_rate", "invoice_on_time_pct"] } }
// ]
```

**Testing:**
- Create scorecard with 5 measures summing to 1.0: returns 201
- Create scorecard with weights summing to 0.85: returns 400 validation error
- Create scorecard with duplicate name in same tenant: returns 409
- Set scorecard as default: previous default is unset
- Get scorecard: returns full measures array with weights and thresholds
- Update measure weight: PATCH scorecard updates the weight and re-validates sum
- Delete scorecard: only allowed if no health scores reference it; otherwise returns 409

---

### Task 3.2: Health Score Calculation Engine

**What:** Build the scoring engine that calculates a composite health score per account based on the scorecard's measures, weights, and thresholds. Each measure score is computed from the relevant data source (events, CRM, support tickets, NPS). The composite score is the weighted sum. Scores are stored as snapshots in `health_scores` with the full measure breakdown in the `measure_scores` JSONB column.

**Design:**

```typescript
// apps/api/src/services/scoring/scoring-engine.ts
export class ScoringEngine {
  async calculateHealthScore(accountId: string, scorecardId: string): Promise<HealthScore> {
    const scorecard = await scorecardService.get(scorecardId);
    const account = await accountService.get(accountId);

    const measureScores: Record<string, MeasureScore> = {};
    let compositeScore = 0;

    for (const measure of scorecard.measures) {
      const calculator = this.getMeasureCalculator(measure.name);
      const result = await calculator.calculate(account, measure.config);

      const score = this.normalizeToScore(result.rawValue, measure);
      const band = this.classifyBand(score, measure.thresholds);

      measureScores[measure.name] = {
        score,
        band,
        rawValue: result.rawValue,
        signals: result.signals,
      };

      compositeScore += score * measure.weight;
    }

    const compositeBand = this.classifyBand(compositeScore, scorecard.defaultThresholds);
    const previousScore = await healthScoreService.getLatest(accountId);
    const trend = this.calculateTrend(compositeScore, previousScore?.compositeScore);

    // Insert health score snapshot
    const healthScore = await db.insert(healthScores).values({
      tenantId: account.tenantId,
      accountId,
      scorecardId,
      compositeScore,
      scoreBand: compositeBand,
      trend,
      measureScores,
      topSignals: this.extractTopSignals(measureScores, 3),
      scoredAt: new Date(),
    });

    // Update denormalized fields on account
    await accountService.updateHealthScore(accountId, {
      healthScore: compositeScore,
      healthBand: compositeBand,
      healthTrend: trend,
      healthScoredAt: new Date(),
    });

    return healthScore;
  }

  private getMeasureCalculator(measureName: string): MeasureCalculator {
    const calculators: Record<string, MeasureCalculator> = {
      usage: new UsageMeasureCalculator(this.eventService),
      engagement: new EngagementMeasureCalculator(this.eventService),
      support: new SupportMeasureCalculator(this.ticketService),
      nps: new NpsMeasureCalculator(this.surveyService),
      financial: new FinancialMeasureCalculator(this.accountService),
    };
    return calculators[measureName] ?? new ManualMeasureCalculator();
  }
}

// apps/api/src/services/scoring/calculators/usage-calculator.ts
export class UsageMeasureCalculator implements MeasureCalculator {
  async calculate(account: Account, config: MeasureConfig): Promise<MeasureResult> {
    const lookbackDays = config.lookback_days ?? 14;
    const events = await this.eventService.getEventCounts(account.id, lookbackDays);

    // DAU as percentage of licensed seats
    const dauPct = (events.uniqueUsers / (account.customFields.licensedSeats ?? 1)) * 100;

    // Compare to previous period
    const prevEvents = await this.eventService.getEventCounts(account.id, lookbackDays, lookbackDays);
    const prevDauPct = (prevEvents.uniqueUsers / (account.customFields.licensedSeats ?? 1)) * 100;
    const dauChange = dauPct - prevDauPct;

    const signals: Signal[] = [];
    if (dauChange < -20) {
      signals.push({
        signal: `dau_dropped_${Math.abs(Math.round(dauChange))}_pct`,
        impact: dauChange * 0.5,
        description: `DAU dropped ${Math.abs(Math.round(dauChange))}% over ${lookbackDays} days`,
      });
    }

    return { rawValue: dauPct, signals };
  }
}
```

**Testing:**
- Score calculation with all green measures: composite score >= 80, band = "green"
- Score calculation with mixed measures: usage red (30), engagement yellow (55), support green (90), nps green (85), financial green (95) -- composite = 30*0.30 + 55*0.25 + 90*0.20 + 85*0.15 + 95*0.10 = 9.0 + 13.75 + 18.0 + 12.75 + 9.5 = 63.0, band = "yellow"
- Score trend: previous score 80, current score 63 -- trend = "declining"
- Score trend: previous score 60, current score 63 -- trend = "improving"
- Score trend: previous score 63, current score 63 -- trend = "stable"
- Health score snapshot stored in `health_scores` table with correct `measure_scores` JSONB
- Denormalized fields on `accounts` table updated after scoring
- Top signals: 3 most impactful signals extracted from all measures
- No events for account: usage and engagement scores default to 0 (red band)
- NPS decay: NPS response older than `decay_days` results in lower NPS score
- Financial measure: account with ARR growth > 10% scores higher than flat ARR

---

### Task 3.3: Real-Time Score Recalculation on Events

**What:** Trigger health score recalculation when significant events arrive (usage drops, support tickets opened, NPS responses received). Use BullMQ to debounce recalculations -- batch incoming events and recalculate at most once per 5 minutes per account.

**Design:**

```typescript
// apps/api/src/jobs/score-recalculation.ts
const DEBOUNCE_MS = 5 * 60 * 1000; // 5 minutes

export async function enqueueScoreRecalculation(accountId: string) {
  await scoreRecalcQueue.add(
    'recalculate',
    { accountId },
    {
      jobId: `score-${accountId}`,  // Deduplicates: same job ID replaces pending job
      delay: DEBOUNCE_MS,
      removeOnComplete: true,
    }
  );
}

// Called from event ingestion pipeline
export async function onEventIngested(event: Event) {
  if (isHealthSignalEvent(event)) {
    await enqueueScoreRecalculation(event.accountId);
  }
}

function isHealthSignalEvent(event: Event): boolean {
  const healthSignalEvents = [
    'login', 'feature_used', 'session_ended', 'report_exported',
    'support_ticket_created', 'support_ticket_resolved',
    'nps_response_submitted',
  ];
  return healthSignalEvents.includes(event.eventName);
}
```

**Testing:**
- Event ingested triggers recalculation: ingest a `feature_used` event, verify BullMQ job enqueued
- Debounce: ingest 10 events for same account within 5 minutes, verify only 1 recalculation job executes
- Non-health events ignored: ingest a `page_viewed` event, verify no recalculation job enqueued
- Score changes persisted: after recalculation, `health_scores` table has new row and `accounts` table updated
- Band change detection: score drops from green (82) to yellow (58), verify band change recorded
- WebSocket notification: on band change, connected CSM receives real-time alert (Phase 5 integration)

---

### Task 3.4: Health Score History API

**What:** Build API endpoints to retrieve health score history for an account (time series), health score distribution across the portfolio, and score trend comparisons.

**Design:**

```typescript
// apps/api/src/routes/health-scores/index.ts
fastify.get('/accounts/:accountId/health-scores', {
  schema: {
    querystring: {
      type: 'object',
      properties: {
        from: { type: 'string', format: 'date' },
        to: { type: 'string', format: 'date' },
        interval: { type: 'string', enum: ['daily', 'weekly', 'monthly'] },
      },
    },
  },
  handler: async (request, reply) => {
    const scores = await healthScoreService.getHistory(
      request.params.accountId,
      request.query.from,
      request.query.to,
      request.query.interval,
    );
    return { data: scores };
  },
});

// Portfolio health distribution
fastify.get('/portfolio/health-distribution', {
  handler: async (request, reply) => {
    const distribution = await healthScoreService.getPortfolioDistribution(
      request.user.tenantId,
    );
    // Returns: { green: { count: 45, arr: 2400000 }, yellow: { count: 12, arr: 890000 }, red: { count: 3, arr: 320000 } }
    return { data: distribution };
  },
});
```

**Testing:**
- Health score history: GET `/accounts/:id/health-scores?from=2026-01-01&to=2026-05-25` returns array of score snapshots
- Interval aggregation: `interval=weekly` returns one score per week (latest in each week)
- Portfolio distribution: returns correct count and ARR per health band
- Empty history: new account with no scores returns empty array
- Date filtering: scores outside the `from`/`to` range are excluded

---

## Phase 4: Playbook & CTA System

**Goal:** Build the playbook template system, automatic CTA generation based on health events, CSM cockpit for action management, and playbook execution engine.

**Duration:** 3-4 weeks
**Dependencies:** Phase 3 (needs health score events to trigger playbooks)

### Task 4.1: Playbook Template CRUD

**What:** Build CRUD API for playbook templates. Each playbook has a trigger condition (health threshold crossing, lifecycle change, renewal approaching) and an ordered list of steps (create CTA, send email, send Slack message, wait, escalate).

**Design:**

```typescript
// apps/api/src/routes/playbooks/index.ts
fastify.post('/playbooks', {
  schema: {
    body: {
      type: 'object',
      required: ['name', 'category', 'triggerConfig', 'steps'],
      properties: {
        name: { type: 'string' },
        category: { type: 'string', enum: ['onboarding', 'risk', 'renewal', 'expansion', 'escalation'] },
        triggerConfig: {
          type: 'object',
          required: ['type'],
          properties: {
            type: { type: 'string', enum: ['health_threshold', 'lifecycle_change', 'scheduled', 'manual'] },
            measure: { type: 'string' },
            condition: { type: 'string', enum: ['drops_below', 'rises_above', 'band_changes_to'] },
            threshold: { type: 'number' },
            from: { type: 'string' },
            to: { type: 'string' },
            daysBefore: { type: 'integer' },
          },
        },
        steps: {
          type: 'array',
          items: {
            type: 'object',
            required: ['order', 'action'],
            properties: {
              order: { type: 'integer' },
              action: { type: 'string', enum: ['create_cta', 'send_email', 'send_slack', 'wait', 'escalation', 'ai_decision'] },
              config: { type: 'object' },
            },
          },
        },
      },
    },
  },
  handler: playbookController.create,
});
```

**Testing:**
- Create playbook: POST with valid body returns 201
- Create playbook with empty steps: returns 400
- Create playbook with duplicate step orders: returns 400
- List playbooks by category: GET `/playbooks?category=risk` returns only risk playbooks
- Update playbook steps: PATCH replaces entire steps array
- Deactivate playbook: PATCH `{ "isActive": false }` prevents trigger

---

### Task 4.2: Playbook Trigger Engine

**What:** Build the event-driven trigger engine that evaluates playbook trigger conditions when health scores change, lifecycle stages transition, or scheduled events fire. When a trigger matches, the engine creates a CTA and begins executing playbook steps.

**Design:**

```typescript
// apps/api/src/services/playbook-engine/trigger-engine.ts
export class PlaybookTriggerEngine {
  async evaluateTriggersForAccount(accountId: string, event: TriggerEvent): Promise<void> {
    const tenantId = event.tenantId;
    const playbooks = await playbookService.getActivePlaybooks(tenantId);

    for (const playbook of playbooks) {
      if (this.matchesTrigger(playbook.triggerConfig, event)) {
        // Check if this playbook is already running for this account
        const existingCta = await ctaService.findActiveByPlaybook(accountId, playbook.id);
        if (existingCta) continue;  // Don't re-trigger

        // Create CTA from playbook
        const cta = await ctaService.create({
          tenantId,
          accountId,
          playbookId: playbook.id,
          assignedTo: event.csmId,
          ctaType: playbook.category === 'risk' ? 'risk' : playbook.category,
          priority: this.determinePriority(event),
          title: this.renderTemplate(playbook.steps[0]?.config?.titleTemplate, event),
          context: { triggerEvent: event },
          tasks: this.buildTasksFromSteps(playbook.steps),
        });

        // Enqueue first playbook step execution
        await playbookExecutionQueue.add('execute-step', {
          ctaId: cta.id,
          playbookId: playbook.id,
          stepIndex: 0,
        });
      }
    }
  }

  private matchesTrigger(triggerConfig: TriggerConfig, event: TriggerEvent): boolean {
    switch (triggerConfig.type) {
      case 'health_threshold':
        return event.type === 'health_score_changed'
          && event.measure === triggerConfig.measure
          && this.evaluateCondition(event.newScore, triggerConfig.condition, triggerConfig.threshold);
      case 'lifecycle_change':
        return event.type === 'lifecycle_changed'
          && event.from === triggerConfig.from
          && event.to === triggerConfig.to;
      case 'scheduled':
        return event.type === 'renewal_approaching'
          && event.daysUntilRenewal <= triggerConfig.daysBefore;
      default:
        return false;
    }
  }
}
```

**Testing:**
- Health threshold trigger: usage score drops below 40, risk playbook triggers, CTA created with priority "high"
- Lifecycle trigger: account transitions from "onboarding" to "adopting", onboarding completion playbook triggers
- Scheduled trigger: account with renewal in 89 days, 90-day renewal playbook triggers
- No double-trigger: same playbook does not create second CTA if one is already active for this account
- Inactive playbook: deactivated playbook does not trigger even when conditions match
- Priority determination: red band account gets "critical" priority CTA; yellow gets "high"
- Template rendering: `"Usage decline -- {{account.name}}"` renders to `"Usage decline -- Acme Corp"`

---

### Task 4.3: CTA Management and CSM Cockpit API

**What:** Build the CTA CRUD and the CSM cockpit API that surfaces prioritized actions. CSMs see their CTAs sorted by priority and due date, with account context (health band, ARR, renewal date).

**Design:**

```typescript
// apps/api/src/routes/ctas/cockpit.ts
fastify.get('/cockpit', {
  schema: {
    querystring: {
      type: 'object',
      properties: {
        status: { type: 'string', enum: ['open', 'in_progress', 'snoozed'] },
        ctaType: { type: 'string' },
        sort: { type: 'string', default: 'priority_then_due' },
      },
    },
  },
  handler: async (request, reply) => {
    const ctas = await ctaService.getCsmCockpit(request.user.id, request.query);
    // Returns CTAs with denormalized account info:
    // { id, title, priority, status, dueDate, account: { name, arr, healthBand, renewalDate }, tasks: [...] }
    return { data: ctas };
  },
});

// Update CTA status
fastify.patch('/ctas/:id', {
  handler: async (request, reply) => {
    const cta = await ctaService.update(request.params.id, request.body);
    // If status changed, log to timeline
    if (request.body.status) {
      await timelineService.addEntry(cta.accountId, {
        entryType: 'system',
        subject: `CTA ${request.body.status}: ${cta.title}`,
        metadata: { ctaId: cta.id, oldStatus: cta.status, newStatus: request.body.status },
      });
    }
    return { data: cta };
  },
});

// Snooze CTA
fastify.post('/ctas/:id/snooze', {
  schema: { body: { type: 'object', required: ['until'], properties: { until: { type: 'string', format: 'date-time' } } } },
  handler: ctaController.snooze,
});
```

**Testing:**
- CSM cockpit: returns CTAs assigned to current user, sorted by priority (critical > high > medium > low), then by due date ascending
- Cockpit shows only open/in_progress CTAs by default (excludes closed)
- CTA with denormalized account: response includes `account.name`, `account.arr`, `account.healthBand`
- Update CTA status: PATCH `{ "status": "in_progress" }` updates status and creates timeline entry
- Close CTA: PATCH `{ "status": "closed_success" }` sets `closedAt` timestamp
- Snooze CTA: POST `/ctas/:id/snooze` with `until: "2026-06-01"` sets status to "snoozed" and `snoozedUntil` field
- Snoozed CTA re-opens: BullMQ scheduled job re-opens snoozed CTAs when `snoozedUntil` passes
- Task completion: PATCH `/ctas/:id/tasks/:taskId` with `{ "status": "completed" }` updates the JSONB tasks array
- All tasks completed: CTA automatically transitions to "closed_success" when all tasks are done

---

## Phase 5: Portfolio Dashboard & API

**Goal:** Build the frontend dashboard with portfolio overview, account detail views, health score trends, CSM cockpit, and real-time notifications.

**Duration:** 4-5 weeks
**Dependencies:** Phase 1, Phase 2 (data available), Phase 3 (health scores), Phase 4 (CTAs)

### Task 5.1: Dashboard Layout and Navigation

**What:** Build the authenticated dashboard shell with sidebar navigation, tenant context display, and responsive layout using Next.js 15 App Router and shadcn/ui.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/layout.tsx
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex h-screen">
      <Sidebar>
        <SidebarHeader>
          <TenantLogo />
          <TenantName />
        </SidebarHeader>
        <SidebarNav>
          <NavItem href="/portfolio" icon={LayoutDashboard}>Portfolio</NavItem>
          <NavItem href="/accounts" icon={Building2}>Accounts</NavItem>
          <NavItem href="/cockpit" icon={Target}>Cockpit</NavItem>
          <NavItem href="/playbooks" icon={BookOpen}>Playbooks</NavItem>
          <NavItem href="/integrations" icon={Plug}>Integrations</NavItem>
          <NavItem href="/settings" icon={Settings}>Settings</NavItem>
        </SidebarNav>
        <SidebarFooter>
          <UserMenu />
        </SidebarFooter>
      </Sidebar>
      <main className="flex-1 overflow-y-auto bg-muted/30">
        <TopBar />
        <div className="p-6">{children}</div>
      </main>
    </div>
  );
}
```

**Testing:**
- Dashboard renders without hydration errors
- Sidebar navigation: clicking each nav item loads the correct page
- Responsive: sidebar collapses to icon-only on screens < 1024px
- Tenant name displays correctly in sidebar header
- User menu shows user name, role, and logout option
- Unauthenticated access redirects to login page

---

### Task 5.2: Portfolio Dashboard Page

**What:** Build the portfolio overview page showing health distribution (donut chart), at-risk ARR, upcoming renewals, open CTAs by type, and NRR trend line.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/portfolio/page.tsx
export default async function PortfolioPage() {
  const [distribution, renewals, ctaSummary, arrTrend] = await Promise.all([
    api.getHealthDistribution(),
    api.getUpcomingRenewals({ days: 90 }),
    api.getCtaSummary(),
    api.getArrTrend({ months: 12 }),
  ]);

  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-semibold">Portfolio Health</h1>
      <div className="grid grid-cols-4 gap-4">
        <MetricCard title="Total ARR" value={formatCurrency(distribution.totalArr)} />
        <MetricCard title="At-Risk ARR" value={formatCurrency(distribution.red.arr)} variant="destructive" />
        <MetricCard title="Renewals (90d)" value={renewals.count} />
        <MetricCard title="Open CTAs" value={ctaSummary.open} />
      </div>
      <div className="grid grid-cols-2 gap-6">
        <Card>
          <CardHeader><CardTitle>Health Distribution</CardTitle></CardHeader>
          <CardContent>
            <HealthDonutChart data={distribution} />
          </CardContent>
        </Card>
        <Card>
          <CardHeader><CardTitle>ARR Trend</CardTitle></CardHeader>
          <CardContent>
            <ArrTrendChart data={arrTrend} />
          </CardContent>
        </Card>
      </div>
      <Card>
        <CardHeader><CardTitle>At-Risk Accounts</CardTitle></CardHeader>
        <CardContent>
          <AtRiskAccountsTable accounts={distribution.redAccounts} />
        </CardContent>
      </Card>
    </div>
  );
}
```

**Testing:**
- Portfolio page loads within 2 seconds with 500 accounts
- Health distribution donut shows green/yellow/red segments with correct percentages
- At-risk ARR metric shows sum of ARR for red-band accounts
- Renewals count matches accounts with `renewalDate` within 90 days
- ARR trend chart displays 12 months of data with correct values
- At-risk accounts table shows red-band accounts sorted by ARR descending
- Empty state: new tenant with no accounts shows appropriate empty state message

---

### Task 5.3: Account Detail Page

**What:** Build the account detail page with Customer 360 view: health score history chart, current measure breakdown, timeline, contacts, active CTAs, and key metrics.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/accounts/[id]/page.tsx
export default async function AccountDetailPage({ params }: { params: { id: string } }) {
  const [account, healthHistory, contacts, ctas, timeline] = await Promise.all([
    api.getAccount(params.id),
    api.getHealthScoreHistory(params.id, { months: 6 }),
    api.getContacts(params.id),
    api.getAccountCtas(params.id, { status: 'open' }),
    api.getTimeline(params.id, { limit: 20 }),
  ]);

  return (
    <div className="space-y-6">
      <AccountHeader account={account} />
      <div className="grid grid-cols-3 gap-4">
        <MetricCard title="Health Score" value={account.healthScore} band={account.healthBand} />
        <MetricCard title="ARR" value={formatCurrency(account.arr)} />
        <MetricCard title="Renewal" value={formatDate(account.renewalDate)} />
      </div>
      {account.healthNarrative && (
        <AiNarrativeCard narrative={account.healthNarrative} />
      )}
      <div className="grid grid-cols-2 gap-6">
        <HealthScoreChart history={healthHistory} />
        <MeasureBreakdownCard scores={healthHistory[0]?.measureScores} />
      </div>
      <Tabs defaultValue="timeline">
        <TabsList>
          <TabsTrigger value="timeline">Timeline</TabsTrigger>
          <TabsTrigger value="contacts">Contacts ({contacts.length})</TabsTrigger>
          <TabsTrigger value="ctas">CTAs ({ctas.length})</TabsTrigger>
        </TabsList>
        <TabsContent value="timeline"><TimelineView entries={timeline} /></TabsContent>
        <TabsContent value="contacts"><ContactsTable contacts={contacts} /></TabsContent>
        <TabsContent value="ctas"><CtaList ctas={ctas} /></TabsContent>
      </Tabs>
    </div>
  );
}
```

**Testing:**
- Account detail loads with all sections populated
- Health score chart renders 6 months of data as a line chart
- Measure breakdown shows each measure's score, band, and weight
- Timeline shows entries in reverse chronological order
- Contacts tab shows all contacts with role badges
- CTAs tab shows open CTAs with priority badges
- AI narrative card renders when `healthNarrative` is non-null
- 404 page shown for nonexistent account ID
- Health score chart handles missing data points (gaps in history)

---

### Task 5.4: CSM Cockpit Page

**What:** Build the CSM cockpit page showing the prioritized action queue. Each CTA card shows the account context, health band, priority, due date, and task progress.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/cockpit/page.tsx
export default async function CockpitPage() {
  const ctas = await api.getCockpit({ status: 'open,in_progress' });

  return (
    <div className="space-y-4">
      <div className="flex justify-between items-center">
        <h1 className="text-2xl font-semibold">My Actions</h1>
        <CockpitFilters />
      </div>
      <div className="space-y-3">
        {ctas.map(cta => (
          <CtaCard key={cta.id} cta={cta}>
            <CtaCardHeader>
              <PriorityBadge priority={cta.priority} />
              <span className="font-medium">{cta.title}</span>
              <HealthBandBadge band={cta.account.healthBand} />
            </CtaCardHeader>
            <CtaCardContent>
              <div className="flex items-center gap-4 text-sm text-muted-foreground">
                <span>{cta.account.name}</span>
                <span>{formatCurrency(cta.account.arr)}</span>
                <span>Due: {formatDate(cta.dueDate)}</span>
              </div>
              <TaskProgressBar completed={cta.completedTasks} total={cta.totalTasks} />
            </CtaCardContent>
            <CtaCardActions>
              <Button variant="outline" size="sm" onClick={() => snooze(cta.id)}>Snooze</Button>
              <Button variant="outline" size="sm" onClick={() => complete(cta.id)}>Complete</Button>
            </CtaCardActions>
          </CtaCard>
        ))}
      </div>
    </div>
  );
}
```

**Testing:**
- Cockpit shows CTAs sorted by priority then due date
- Critical CTAs have red accent; high CTAs have orange
- Snooze button opens date picker; selecting date snoozes CTA and removes from list
- Complete button marks CTA as closed_success and removes from list
- Task progress bar shows correct fraction (e.g., "2 of 5 tasks")
- Clicking CTA title navigates to account detail page
- Empty cockpit shows "No actions pending" message
- Real-time: new CTA created by playbook appears in cockpit without page refresh (WebSocket)

---

### Task 5.5: Real-Time Notifications via WebSocket

**What:** Implement WebSocket connections for real-time dashboard updates. Push health band changes, new CTAs, and churn alerts to connected CSM clients.

**Design:**

```typescript
// apps/api/src/plugins/websocket.ts
export async function websocketPlugin(fastify: FastifyInstance) {
  fastify.register(socketIo, {
    cors: { origin: env.FRONTEND_URL },
  });

  fastify.io.use(async (socket, next) => {
    const token = socket.handshake.auth.token;
    const user = await verifyToken(token);
    if (!user) return next(new Error('Unauthorized'));
    socket.data.user = user;
    socket.join(`tenant:${user.tenantId}`);
    socket.join(`user:${user.id}`);
    next();
  });
}

// apps/api/src/services/notification-service.ts
export class NotificationService {
  async notifyHealthBandChange(accountId: string, oldBand: string, newBand: string) {
    const account = await accountService.get(accountId);
    fastify.io.to(`tenant:${account.tenantId}`).emit('health:band-changed', {
      accountId,
      accountName: account.name,
      oldBand,
      newBand,
      healthScore: account.healthScore,
    });
  }

  async notifyNewCta(cta: Cta) {
    fastify.io.to(`user:${cta.assignedTo}`).emit('cta:created', {
      ctaId: cta.id,
      title: cta.title,
      priority: cta.priority,
      accountName: cta.accountName,
    });
  }
}
```

**Testing:**
- WebSocket connection established with valid JWT
- WebSocket connection rejected with invalid JWT
- Health band change event received by all users in the tenant
- New CTA event received only by the assigned CSM
- Client reconnection: disconnected client reconnects and re-joins rooms
- Multiple tabs: user with 2 tabs receives event on both
- Event payload matches expected schema

---

## Phase 6: AI Health Narratives

**Goal:** Generate plain-language explanations of health score changes using Claude API. Each account gets a narrative explaining why the score changed and what the most important signals are.

**Duration:** 2-3 weeks
**Dependencies:** Phase 3 (health scores with measure breakdowns)

### Task 6.1: Narrative Generation Service

**What:** Build the LLM-powered service that generates health score narratives. Takes the current and previous health score snapshots, measure breakdowns, and top signals, then generates a 2-3 sentence explanation.

**Design:**

```typescript
// apps/api/src/services/ai/narrative-service.ts
import Anthropic from '@anthropic-ai/sdk';

export class NarrativeService {
  private client: Anthropic;

  constructor() {
    this.client = new Anthropic();
  }

  async generateHealthNarrative(
    account: Account,
    currentScore: HealthScore,
    previousScore: HealthScore | null,
  ): Promise<string> {
    const systemPrompt = `You are a customer success analyst. Generate a concise 2-3 sentence health score narrative for a CSM. Focus on: (1) what changed, (2) which signals drove the change, (3) what the CSM should investigate. Use specific numbers. Do not use jargon. Write in present tense.`;

    const userPrompt = `Account: ${account.name} (${account.tier}, ARR: $${account.arr})
Lifecycle: ${account.lifecycleStage}
Renewal: ${account.renewalDate ? `in ${daysUntil(account.renewalDate)} days` : 'N/A'}

Current Health Score: ${currentScore.compositeScore}/100 (${currentScore.scoreBand})
${previousScore ? `Previous Score: ${previousScore.compositeScore}/100 (${previousScore.scoreBand})` : 'First score for this account.'}
Trend: ${currentScore.trend}

Measure Breakdown:
${Object.entries(currentScore.measureScores).map(([name, m]) =>
  `- ${name}: ${m.score}/100 (${m.band}) [raw: ${m.rawValue}]`
).join('\n')}

Top Signals:
${currentScore.topSignals.map(s =>
  `- ${s.description} (impact: ${s.impact > 0 ? '+' : ''}${s.impact})`
).join('\n')}`;

    const response = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 300,
      system: systemPrompt,
      messages: [{ role: 'user', content: userPrompt }],
    });

    return response.content[0].type === 'text' ? response.content[0].text : '';
  }
}
```

**Testing:**
- Narrative generated for declining score mentions the specific measures that dropped
- Narrative generated for improving score mentions the measures that improved
- Narrative includes specific numbers (e.g., "DAU dropped 40%")
- Narrative is 2-3 sentences long (50-150 words)
- Narrative does not hallucinate data not present in the input
- Narrative stored in `health_scores.ai_narrative` and `accounts.health_narrative`
- API error handling: if Claude API is unavailable, health score still saved with null narrative
- Rate limiting: narrative generation respects Claude API rate limits with exponential backoff
- Cost tracking: log token usage per narrative for cost monitoring

---

### Task 6.2: Narrative Display in Dashboard

**What:** Display the AI-generated narrative prominently on the account detail page, in the CSM cockpit CTA cards, and in Slack/email notifications.

**Design:**

```typescript
// apps/web/src/components/ai-narrative-card.tsx
export function AiNarrativeCard({ narrative, scoredAt }: { narrative: string; scoredAt: string }) {
  return (
    <Card className="border-l-4 border-l-primary">
      <CardHeader className="flex flex-row items-center gap-2 pb-2">
        <Sparkles className="h-4 w-4 text-primary" />
        <CardTitle className="text-sm font-medium">AI Health Summary</CardTitle>
        <span className="text-xs text-muted-foreground ml-auto">
          Updated {formatRelativeTime(scoredAt)}
        </span>
      </CardHeader>
      <CardContent>
        <p className="text-sm leading-relaxed">{narrative}</p>
      </CardContent>
    </Card>
  );
}
```

**Testing:**
- Narrative card renders on account detail page when narrative is available
- Narrative card hidden when narrative is null
- Relative time display: "Updated 2 hours ago" format
- Narrative text is not truncated (full 2-3 sentences displayed)
- Narrative card appears in CTA cards in cockpit view
- Slack notification includes narrative text

---

## Phase 7: Churn Prediction ML Pipeline

**Goal:** Build the machine learning pipeline for predictive churn scoring. Train models on historical product usage, support, and engagement data to predict churn probability with explainable feature importance.

**Duration:** 4-5 weeks
**Dependencies:** Phase 2 (event data), Phase 3 (health scores), Phase 6 (narrative integration)

### Task 7.1: Feature Engineering Pipeline

**What:** Build the Python feature engineering pipeline that extracts predictive features from raw events, CRM data, and support ticket data. Features include usage trends, engagement decay rates, support burden, NPS trajectories, and stakeholder changes.

**Design:**

```python
# apps/ml/src/features/feature_engine.py
import pandas as pd
from datetime import datetime, timedelta

class FeatureEngine:
    """Extracts churn-predictive features for a single account."""

    def extract_features(self, account_id: str, as_of_date: datetime) -> dict:
        features = {}

        # Usage features (14-day and 30-day windows)
        usage_14d = self.get_usage_metrics(account_id, as_of_date, days=14)
        usage_30d = self.get_usage_metrics(account_id, as_of_date, days=30)
        features['dau_14d'] = usage_14d['daily_active_users']
        features['dau_30d'] = usage_30d['daily_active_users']
        features['dau_trend_14d'] = usage_14d['dau_trend']  # slope of DAU over 14 days
        features['feature_breadth_14d'] = usage_14d['unique_features_used']
        features['session_duration_avg_14d'] = usage_14d['avg_session_minutes']
        features['login_frequency_14d'] = usage_14d['logins_per_user_per_week']

        # Engagement decay: ratio of recent activity to historical baseline
        baseline = self.get_usage_metrics(account_id, as_of_date - timedelta(days=30), days=30)
        features['engagement_decay_ratio'] = (
            usage_14d['total_events'] / max(baseline['total_events'], 1)
        )

        # Support features
        support = self.get_support_metrics(account_id, as_of_date, days=30)
        features['open_tickets_count'] = support['open_tickets']
        features['p1_tickets_30d'] = support['p1_tickets']
        features['avg_resolution_hours'] = support['avg_resolution_hours']
        features['escalation_count_30d'] = support['escalations']

        # NPS features
        nps = self.get_nps_metrics(account_id, as_of_date)
        features['latest_nps_score'] = nps['latest_score']
        features['nps_trend'] = nps['trend']  # positive, flat, negative
        features['days_since_nps'] = nps['days_since_response']

        # Contract features
        contract = self.get_contract_features(account_id, as_of_date)
        features['days_until_renewal'] = contract['days_until_renewal']
        features['contract_tenure_months'] = contract['tenure_months']
        features['arr'] = contract['arr']
        features['arr_growth_rate'] = contract['arr_growth_rate']

        # Stakeholder features
        stakeholder = self.get_stakeholder_features(account_id, as_of_date)
        features['contact_count'] = stakeholder['total_contacts']
        features['champion_active'] = stakeholder['has_active_champion']
        features['exec_sponsor_active'] = stakeholder['has_active_exec_sponsor']
        features['contacts_departed_90d'] = stakeholder['contacts_departed']

        return features
```

**Testing:**
- Feature extraction produces all expected features for a fully-populated account
- Missing data handling: account with no events returns `dau_14d=0`, not NaN
- Engagement decay ratio: account with steady usage returns ratio close to 1.0
- Engagement decay ratio: account with 50% drop returns ratio close to 0.5
- Feature extraction runs within 500ms per account (benchmark with 1000 events)
- Feature schema is stable: adding new features does not break existing model training
- NPS trend calculation: 3 consecutive declining NPS scores returns `trend="negative"`

---

### Task 7.2: Model Training and Evaluation

**What:** Train an XGBoost classification model for binary churn prediction. Evaluate using precision, recall, F1, and AUC-ROC. Implement k-fold cross-validation and feature importance extraction.

**Design:**

```python
# apps/ml/src/models/churn_model.py
import xgboost as xgb
from sklearn.model_selection import StratifiedKFold
from sklearn.metrics import precision_score, recall_score, f1_score, roc_auc_score
import shap

class ChurnPredictionModel:
    def train(self, features_df: pd.DataFrame, labels: pd.Series) -> dict:
        # Stratified 5-fold cross-validation
        kfold = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
        metrics = {'precision': [], 'recall': [], 'f1': [], 'auc_roc': []}

        for train_idx, val_idx in kfold.split(features_df, labels):
            X_train, X_val = features_df.iloc[train_idx], features_df.iloc[val_idx]
            y_train, y_val = labels.iloc[train_idx], labels.iloc[val_idx]

            model = xgb.XGBClassifier(
                n_estimators=200,
                max_depth=6,
                learning_rate=0.1,
                scale_pos_weight=len(y_train[y_train == 0]) / len(y_train[y_train == 1]),
                eval_metric='auprc',
                random_state=42,
            )
            model.fit(X_train, y_train, eval_set=[(X_val, y_val)], verbose=False)

            y_pred = model.predict(X_val)
            y_prob = model.predict_proba(X_val)[:, 1]

            metrics['precision'].append(precision_score(y_val, y_pred))
            metrics['recall'].append(recall_score(y_val, y_pred))
            metrics['f1'].append(f1_score(y_val, y_pred))
            metrics['auc_roc'].append(roc_auc_score(y_val, y_prob))

        # Train final model on full dataset
        final_model = xgb.XGBClassifier(...)
        final_model.fit(features_df, labels)

        # Extract SHAP feature importance
        explainer = shap.TreeExplainer(final_model)
        shap_values = explainer.shap_values(features_df)

        return {
            'model': final_model,
            'metrics': {k: {'mean': np.mean(v), 'std': np.std(v)} for k, v in metrics.items()},
            'feature_importance': dict(zip(features_df.columns, np.abs(shap_values).mean(axis=0))),
        }

    def predict(self, model: xgb.XGBClassifier, features: dict) -> dict:
        features_df = pd.DataFrame([features])
        probability = model.predict_proba(features_df)[0][1]
        risk_band = 'high' if probability > 0.7 else 'medium' if probability > 0.3 else 'low'

        # Per-prediction SHAP values for explainability
        explainer = shap.TreeExplainer(model)
        shap_values = explainer.shap_values(features_df)[0]
        top_features = sorted(
            zip(features_df.columns, shap_values),
            key=lambda x: abs(x[1]), reverse=True
        )[:5]

        return {
            'churn_probability': float(probability),
            'risk_band': risk_band,
            'top_features': [
                {'feature': f, 'importance': float(abs(v)), 'direction': 'positive' if v > 0 else 'negative'}
                for f, v in top_features
            ],
        }
```

**Testing:**
- Model achieves precision >= 0.85 on cross-validation (target: 0.94 per ZapScale benchmark)
- Model achieves AUC-ROC >= 0.90
- Feature importance: `dau_trend_14d`, `engagement_decay_ratio`, and `open_tickets_count` are in top 5 features
- Prediction output includes `churn_probability`, `risk_band`, and `top_features` with correct schema
- Model handles missing features gracefully (XGBoost native missing value handling)
- Class imbalance: `scale_pos_weight` correctly balances churned vs. retained classes
- Model artifact serialized to S3/MinIO and loadable for inference
- Training reproducibility: same seed produces same metrics within 0.01 tolerance

---

### Task 7.3: Prediction API and Storage

**What:** Build the FastAPI prediction service and integrate with the main API. Predictions are stored in the database and surfaced in the dashboard.

**Design:**

```python
# apps/ml/src/api/predict.py
from fastapi import FastAPI, HTTPException
app = FastAPI()

@app.post("/predict/churn/{account_id}")
async def predict_churn(account_id: str):
    features = feature_engine.extract_features(account_id, datetime.utcnow())
    prediction = churn_model.predict(active_model, features)

    # Store prediction
    await db.execute(
        "INSERT INTO churn_predictions (tenant_id, account_id, model_id, churn_probability, risk_band, top_features) "
        "VALUES ($1, $2, $3, $4, $5, $6)",
        [tenant_id, account_id, active_model_id,
         prediction['churn_probability'], prediction['risk_band'],
         json.dumps(prediction['top_features'])]
    )

    # Update denormalized fields on account
    await db.execute(
        "UPDATE accounts SET churn_probability = $1, churn_risk_band = $2 WHERE id = $3",
        [prediction['churn_probability'], prediction['risk_band'], account_id]
    )

    return prediction
```

**Testing:**
- Prediction API returns valid response within 500ms
- Prediction stored in `churn_predictions` table
- Account denormalized fields updated after prediction
- Prediction with high probability (>0.7) triggers churn risk playbook (integration with Phase 4)
- Model version tracked: prediction references correct `model_id`
- Batch prediction: `/predict/churn/batch` processes all accounts for a tenant

---

## Phase 8: Journey Orchestration

**Goal:** Build the multi-step, multi-channel journey orchestration engine that extends playbooks with email sequences, in-app messaging, and conditional branching.

**Duration:** 3-4 weeks
**Dependencies:** Phase 4 (playbook system)

### Task 8.1: Email Template System

**What:** Build the email template engine with variable interpolation, tenant branding, and delivery via a transactional email provider (SendGrid, Postmark, or AWS SES).

**Design:**

```typescript
// apps/api/src/services/journey/email-service.ts
export class EmailService {
  async sendTemplatedEmail(
    template: EmailTemplate,
    recipient: Contact,
    variables: Record<string, string>,
    tenant: Tenant,
  ): Promise<void> {
    const html = this.renderTemplate(template.htmlBody, {
      ...variables,
      contact_name: recipient.name,
      account_name: variables.accountName,
      csm_name: variables.csmName,
      company_logo: tenant.settings.branding?.logoUrl,
    });

    await this.emailProvider.send({
      from: `${variables.csmName} <${variables.csmEmail}>`,
      to: recipient.email,
      subject: this.renderTemplate(template.subject, variables),
      html,
      trackOpens: true,
      trackClicks: true,
    });

    // Log to timeline
    await timelineService.addEntry(variables.accountId, {
      entryType: 'email',
      subject: `Email sent: ${template.name}`,
      body: html,
      metadata: { templateId: template.id, recipientEmail: recipient.email },
    });
  }
}
```

**Testing:**
- Email sent with correct recipient, subject, and body
- Variable interpolation: `{{contact_name}}` replaced with actual contact name
- Missing variable: template renders with empty string (not `{{variable}}` literal)
- Tenant branding: company logo appears in email HTML
- Email logged to account timeline
- Send failure: transactional email API error is caught and logged without crashing

---

### Task 8.2: Multi-Channel Step Execution

**What:** Extend the playbook step executor to handle all channel types: email, Slack message, in-app notification, wait (delay), and escalation (reassign CTA to manager).

**Design:**

```typescript
// apps/api/src/services/playbook-engine/step-executor.ts
export class StepExecutor {
  async executeStep(cta: Cta, step: PlaybookStep): Promise<void> {
    switch (step.action) {
      case 'send_email':
        const template = await emailTemplateService.get(step.config.templateId);
        const contact = await contactService.getPrimary(cta.accountId);
        await emailService.sendTemplatedEmail(template, contact, this.buildVariables(cta));
        break;

      case 'send_slack':
        await slackService.sendMessage(cta.tenantId, step.config.channel, {
          text: this.renderTemplate(step.config.template, this.buildVariables(cta)),
          blocks: this.buildSlackBlocks(cta, step),
        });
        break;

      case 'wait':
        // Schedule next step after delay
        await playbookExecutionQueue.add('execute-step', {
          ctaId: cta.id,
          stepIndex: step.order,
        }, { delay: step.config.durationDays * 24 * 60 * 60 * 1000 });
        return;  // Do not proceed to next step immediately

      case 'escalation':
        const manager = await userService.getManager(cta.assignedTo);
        await ctaService.reassign(cta.id, manager.id);
        await notificationService.notifyEscalation(cta, manager);
        break;

      case 'create_cta':
        await ctaService.create({ ...step.config.ctaTemplate, accountId: cta.accountId });
        break;
    }

    // Mark task as completed in CTA
    await ctaService.completeTask(cta.id, `step-${step.order}`);

    // Enqueue next step if exists
    const nextStep = await playbookService.getNextStep(cta.playbookId, step.order);
    if (nextStep) {
      await playbookExecutionQueue.add('execute-step', {
        ctaId: cta.id,
        stepIndex: nextStep.order,
      }, { delay: nextStep.delayMinutes * 60 * 1000 });
    }
  }
}
```

**Testing:**
- Email step: email sent to primary contact; task marked completed
- Slack step: message sent to configured channel with correct formatting
- Wait step: next step scheduled with correct delay (3 days = 259200000ms)
- Escalation step: CTA reassigned to manager; manager receives notification
- Step sequence: 5-step playbook executes steps in order with correct delays
- Step failure: failed email delivery does not block subsequent steps; error logged
- Playbook completion: all steps executed, CTA transitions to "closed_success"

---

### Task 8.3: Conditional Journey Branching

**What:** Add conditional logic to playbook steps. After a wait step, evaluate account health to decide the next branch (e.g., if health improved, skip escalation; if health declined further, escalate immediately).

**Design:**

```typescript
// apps/api/src/services/playbook-engine/step-executor.ts
case 'condition':
  const conditionMet = await this.evaluateCondition(cta.accountId, step.config.condition);
  const nextStepOrder = conditionMet ? step.config.thenStep : step.config.elseStep;
  await playbookExecutionQueue.add('execute-step', {
    ctaId: cta.id,
    stepIndex: nextStepOrder,
  });
  break;

// Example condition config:
// { "type": "health_band_is", "band": "red", "thenStep": 4, "elseStep": 6 }
// { "type": "health_improved_since_cta", "thenStep": 5, "elseStep": 3 }
```

**Testing:**
- Condition evaluates true: execution jumps to `thenStep`
- Condition evaluates false: execution jumps to `elseStep`
- Health improved condition: account health improved since CTA creation, condition returns true
- Health worsened condition: account health declined, escalation branch triggered
- Nested conditions: condition step leading to another condition step works correctly

---

## Phase 9: QBR Builder Agent

**Goal:** Build an AI agent that automatically drafts Quarterly Business Review decks from live CRM, product usage, and support data.

**Duration:** 3-4 weeks
**Dependencies:** Phase 2 (integrations), Phase 3 (health scores), Phase 6 (AI infrastructure)

### Task 9.1: QBR Data Assembly Service

**What:** Build the service that gathers all data needed for a QBR: account health history, product usage trends, support ticket summary, NPS results, contract details, and key milestones. Package into a structured context document for LLM consumption.

**Design:**

```typescript
// apps/api/src/services/ai/qbr-data-service.ts
export class QbrDataService {
  async assembleQbrContext(accountId: string, quarterStart: Date, quarterEnd: Date): Promise<QbrContext> {
    const [account, healthHistory, usageMetrics, supportSummary, npsSummary, milestones, contacts] =
      await Promise.all([
        accountService.getWithDetails(accountId),
        healthScoreService.getHistory(accountId, quarterStart, quarterEnd),
        eventService.getUsageSummary(accountId, quarterStart, quarterEnd),
        supportService.getQuarterSummary(accountId, quarterStart, quarterEnd),
        surveyService.getNpsSummary(accountId, quarterStart, quarterEnd),
        timelineService.getMilestones(accountId, quarterStart, quarterEnd),
        contactService.getByAccount(accountId),
      ]);

    return {
      account,
      quarter: { start: quarterStart, end: quarterEnd },
      healthHistory,
      usageMetrics: {
        dauTrend: usageMetrics.dauTrend,
        featureAdoption: usageMetrics.featureAdoption,
        topFeatures: usageMetrics.topFeatures,
        newFeaturesAdopted: usageMetrics.newFeaturesAdopted,
      },
      supportSummary: {
        ticketsOpened: supportSummary.opened,
        ticketsResolved: supportSummary.resolved,
        avgResolutionHours: supportSummary.avgResolution,
        escalations: supportSummary.escalations,
      },
      npsSummary,
      milestones,
      stakeholders: contacts.filter(c => ['champion', 'executive_sponsor', 'decision_maker'].includes(c.role)),
    };
  }
}
```

**Testing:**
- QBR context assembles all required data for a quarter
- Missing data handled: account with no NPS responses returns empty NPS summary
- Date range filtering: only data from the specified quarter is included
- Stakeholder filter: only champion, executive_sponsor, and decision_maker contacts included
- Assembly completes within 3 seconds for accounts with 10,000+ events

---

### Task 9.2: QBR Content Generation via LLM

**What:** Use Claude API to generate QBR slide content from the assembled context. Output structured JSON that maps to QBR slide sections: Executive Summary, Health Overview, Usage & Adoption, Support Performance, Action Items, Next Quarter Goals.

**Design:**

```typescript
// apps/api/src/services/ai/qbr-agent.ts
export class QbrAgent {
  async generateQbrContent(context: QbrContext): Promise<QbrContent> {
    const systemPrompt = `You are a customer success analyst generating a Quarterly Business Review. 
Output a JSON object with the following sections, each containing 2-4 bullet points with specific data:
- executiveSummary: High-level account health and relationship summary
- healthOverview: Health score trends with specific numbers and drivers
- usageAdoption: Product usage trends, feature adoption highlights, areas of concern
- supportPerformance: Support ticket trends, resolution times, notable issues
- actionItems: 3-5 specific actions for next quarter
- nextQuarterGoals: 2-3 measurable goals for the customer relationship

Use the provided data. Do not invent numbers. Reference specific metrics.`;

    const response = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2000,
      system: systemPrompt,
      messages: [{
        role: 'user',
        content: `Generate QBR content for ${context.account.name}.

${JSON.stringify(context, null, 2)}`,
      }],
    });

    return JSON.parse(response.content[0].text);
  }
}
```

**Testing:**
- QBR content generated with all 6 sections
- Executive summary mentions account name, ARR, and health trend
- Usage section references specific DAU numbers from context
- Support section references specific ticket counts from context
- Action items are specific and actionable (not generic platitudes)
- No hallucinated data: all numbers in output can be traced to input context
- JSON output is valid and matches expected schema
- Generation completes within 15 seconds

---

### Task 9.3: QBR Export (PDF / Google Slides)

**What:** Render the generated QBR content into a downloadable PDF or Google Slides presentation using a slide template.

**Design:**

```typescript
// apps/api/src/services/ai/qbr-export.ts
export class QbrExportService {
  async exportToPdf(qbrContent: QbrContent, tenant: Tenant): Promise<Buffer> {
    const html = this.renderQbrTemplate(qbrContent, {
      logo: tenant.settings.branding?.logoUrl,
      primaryColor: tenant.settings.branding?.primaryColor,
      companyName: tenant.name,
    });
    // Use Puppeteer to render HTML to PDF
    const browser = await puppeteer.launch({ headless: true });
    const page = await browser.newPage();
    await page.setContent(html);
    const pdf = await page.pdf({ format: 'A4', landscape: true, printBackground: true });
    await browser.close();
    return pdf;
  }
}

// API endpoint
fastify.post('/accounts/:id/qbr/generate', {
  handler: async (request, reply) => {
    const context = await qbrDataService.assembleQbrContext(
      request.params.id,
      request.body.quarterStart,
      request.body.quarterEnd,
    );
    const content = await qbrAgent.generateQbrContent(context);
    const pdf = await qbrExportService.exportToPdf(content, request.user.tenant);
    // Store in S3
    const url = await storageService.upload(`qbr/${request.params.id}/${Date.now()}.pdf`, pdf);
    return { data: { content, downloadUrl: url } };
  },
});
```

**Testing:**
- PDF generated with correct branding (logo, colors)
- PDF contains all 6 QBR sections
- PDF download URL is accessible and returns valid PDF
- PDF renders correctly in landscape A4 format
- QBR stored in object storage with correct path
- Generate endpoint returns within 30 seconds
- Concurrent QBR generation: 3 QBRs generated simultaneously without interference

---

## Phase 10: Expansion Signal Detection

**Goal:** Build AI-powered analysis of product usage patterns, stakeholder changes, and support queries to surface accounts with high expansion propensity.

**Duration:** 3-4 weeks
**Dependencies:** Phase 7 (ML infrastructure), Phase 2 (event data)

### Task 10.1: Expansion Feature Engineering

**What:** Build feature extraction specifically for expansion signals: new user role additions, increased usage beyond licensed capacity, support questions about premium features, new department adoption, and usage ceiling patterns.

**Design:**

```python
# apps/ml/src/features/expansion_features.py
class ExpansionFeatureEngine:
    def extract_features(self, account_id: str, as_of_date: datetime) -> dict:
        features = {}

        # License utilization: are they outgrowing their plan?
        features['seat_utilization_pct'] = self.get_seat_utilization(account_id)
        features['seats_added_30d'] = self.get_new_seats(account_id, days=30)

        # Feature ceiling: are they hitting limits?
        features['api_calls_vs_limit_pct'] = self.get_api_usage_vs_limit(account_id)
        features['storage_used_vs_limit_pct'] = self.get_storage_vs_limit(account_id)

        # Premium feature exploration
        features['premium_feature_page_views'] = self.get_premium_page_views(account_id, days=30)
        features['pricing_page_visits'] = self.get_pricing_page_visits(account_id, days=30)

        # Support signals
        features['feature_request_count_90d'] = self.get_feature_requests(account_id, days=90)
        features['upgrade_inquiry_count'] = self.get_upgrade_inquiries(account_id, days=90)

        # Organizational growth
        features['new_departments_30d'] = self.get_new_departments(account_id, days=30)
        features['new_user_roles_30d'] = self.get_new_user_roles(account_id, days=30)

        # Engagement intensity
        features['power_users_pct'] = self.get_power_user_percentage(account_id)
        features['usage_growth_rate_30d'] = self.get_usage_growth_rate(account_id, days=30)

        return features
```

**Testing:**
- Seat utilization > 90% detected as expansion signal
- Premium feature page views correctly counted from event data
- Feature request count aggregated from support ticket tags
- New department detection based on user metadata changes
- All features produce numeric values (no NaN, no null)

---

### Task 10.2: Expansion Scoring and Alerting

**What:** Build the expansion scoring model and integrate with the CTA system. High-expansion-propensity accounts trigger expansion CTAs for CSMs.

**Design:**

```typescript
// apps/api/src/services/scoring/expansion-scorer.ts
export class ExpansionScorer {
  async scoreExpansionPropensity(accountId: string): Promise<ExpansionScore> {
    const features = await mlService.extractExpansionFeatures(accountId);
    const prediction = await mlService.predictExpansion(features);

    if (prediction.expansionProbability > 0.6) {
      // Create expansion CTA
      await ctaService.create({
        accountId,
        ctaType: 'expansion',
        priority: prediction.expansionProbability > 0.8 ? 'high' : 'medium',
        title: `Expansion opportunity: ${prediction.topSignals[0].description}`,
        context: {
          expansionProbability: prediction.expansionProbability,
          topSignals: prediction.topSignals,
          suggestedActions: prediction.suggestedActions,
        },
      });
    }

    return prediction;
  }
}
```

**Testing:**
- Account with 95% seat utilization and premium page views scores high expansion probability
- High-probability account triggers expansion CTA
- CTA title includes the top expansion signal
- CTA context includes all top signals and suggested actions
- Low-probability accounts (<0.6) do not trigger CTAs
- Expansion scores stored alongside churn predictions for the account

---

## Phase 11: Customer-Facing Portal

**Goal:** Build customer-visible mutual success plan pages with milestones, action items, shared resources, and onboarding progress tracking.

**Duration:** 3-4 weeks
**Dependencies:** Phase 4 (CTAs), Phase 5 (dashboard)

### Task 11.1: Success Plan Data Model and API

**What:** Add success plan entities to the database: plans with milestones, shared action items, and resource links. Each plan is associated with an account and has both CSM-visible and customer-visible fields.

**Design:**

```typescript
// packages/db/src/schema/success-plans.ts
export const successPlans = pgTable('success_plans', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  accountId: uuid('account_id').notNull().references(() => accounts.id),
  title: varchar('title', { length: 500 }).notNull(),
  status: varchar('status', { length: 50 }).notNull().default('active'),
  milestones: jsonb('milestones').notNull().default([]),
  // milestones: [{ id, title, dueDate, status, isCustomerVisible }]
  sharedResources: jsonb('shared_resources').notNull().default([]),
  // sharedResources: [{ id, title, url, type: 'document'|'video'|'link' }]
  accessToken: varchar('access_token', { length: 255 }).notNull().unique(),
  // Unique token for customer access (no login required)
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing:**
- Create success plan: POST returns 201 with unique `accessToken`
- Add milestone: PATCH milestones array with new milestone
- Complete milestone: update milestone status to "completed"
- Access token: GET `/success-plans/token/:accessToken` returns customer-visible plan data
- Invalid token: returns 404
- Customer-visible filter: response excludes milestones with `isCustomerVisible: false`

---

### Task 11.2: Customer-Facing Portal UI

**What:** Build a minimal, branded customer-facing page accessible via unique URL (no login required). Shows mutual success plan with milestones, progress bar, shared resources, and action items.

**Design:**

```typescript
// apps/web/src/app/success-plan/[token]/page.tsx
export default async function SuccessPlanPage({ params }: { params: { token: string } }) {
  const plan = await api.getSuccessPlanByToken(params.token);
  if (!plan) return notFound();

  const visibleMilestones = plan.milestones.filter(m => m.isCustomerVisible);
  const completedCount = visibleMilestones.filter(m => m.status === 'completed').length;

  return (
    <div className="max-w-3xl mx-auto py-12 px-6">
      <div className="flex items-center gap-3 mb-8">
        {plan.tenantLogo && <img src={plan.tenantLogo} className="h-8" />}
        <h1 className="text-2xl font-semibold">{plan.title}</h1>
      </div>
      <ProgressBar value={completedCount} max={visibleMilestones.length} />
      <MilestoneTimeline milestones={visibleMilestones} />
      <SharedResources resources={plan.sharedResources} />
    </div>
  );
}
```

**Testing:**
- Customer portal loads without authentication
- Progress bar shows correct completion percentage
- Milestones displayed in chronological order
- Completed milestones have check mark visual indicator
- Shared resources render as clickable links
- Page renders with tenant branding (logo, colors)
- Internal milestones (isCustomerVisible: false) are not shown
- Invalid token shows 404 page

---

## Phase 12: Enterprise Hardening

**Goal:** Production-ready hardening including SSO/SAML, audit logging, GDPR compliance, rate limiting, OData reporting API, and performance optimization.

**Duration:** 4-5 weeks
**Dependencies:** All previous phases

### Task 12.1: SSO/SAML Authentication

**What:** Implement SAML 2.0 and OIDC SSO for enterprise tenants. Support Okta, Azure AD, and Google Workspace as identity providers.

**Design:**

```typescript
// apps/api/src/routes/auth/sso.ts
fastify.get('/auth/sso/:tenantSlug', {
  handler: async (request, reply) => {
    const tenant = await tenantService.getBySlug(request.params.tenantSlug);
    const ssoConfig = tenant.settings.sso;
    if (!ssoConfig?.enabled) return reply.code(404).send({ error: 'SSO not configured' });

    if (ssoConfig.protocol === 'saml') {
      const samlRequest = buildSamlAuthnRequest(ssoConfig);
      return reply.redirect(ssoConfig.idpSsoUrl + '?SAMLRequest=' + encodeSamlRequest(samlRequest));
    } else {
      // OIDC
      return reply.redirect(buildOidcAuthUrl(ssoConfig));
    }
  },
});
```

**Testing:**
- SAML login: redirect to IdP, receive assertion, create/update user, establish session
- OIDC login: redirect to IdP, exchange code for tokens, create/update user
- JIT provisioning: first-time SSO user automatically created with "csm" role
- SSO-only enforcement: tenant with SSO enabled rejects email/password login
- IdP-initiated SSO: SAML assertion posted directly to callback URL works correctly
- Multiple IdPs: different tenants use different identity providers

---

### Task 12.2: Audit Logging

**What:** Implement comprehensive audit logging for all state-changing operations. Every API mutation records who did what, when, to which resource.

**Design:**

```typescript
// apps/api/src/middleware/audit-log.ts
export async function auditLogMiddleware(request: FastifyRequest, reply: FastifyReply, payload: unknown) {
  if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(request.method)) {
    await auditService.log({
      tenantId: request.user?.tenantId,
      userId: request.user?.id,
      action: `${request.method} ${request.routeOptions.url}`,
      resourceType: extractResourceType(request.routeOptions.url),
      resourceId: request.params?.id,
      requestBody: sanitizeForAudit(request.body),
      responseStatus: reply.statusCode,
      ipAddress: request.ip,
      userAgent: request.headers['user-agent'],
      timestamp: new Date(),
    });
  }
}
```

**Testing:**
- Every POST/PUT/PATCH/DELETE request creates an audit log entry
- GET requests do not create audit log entries
- Audit log includes user ID, tenant ID, action, and timestamp
- Sensitive fields (passwords, tokens) are redacted from `requestBody`
- Audit log queryable by tenant, user, resource type, and date range
- Audit log entries are immutable (no UPDATE or DELETE)

---

### Task 12.3: GDPR Compliance

**What:** Implement right-to-erasure (delete all PII for an account/contact), data export (download all data for an account), and consent tracking.

**Design:**

```typescript
// apps/api/src/services/gdpr/erasure-service.ts
export class ErasureService {
  async eraseAccountPii(accountId: string): Promise<ErasureReport> {
    // 1. Delete all contacts for this account
    const contactsDeleted = await db.delete(contacts).where(eq(contacts.accountId, accountId));

    // 2. Anonymize events (remove PII from properties JSONB)
    await db.update(events)
      .set({ properties: sql`properties - 'email' - 'name' - 'ip'`, contactId: null })
      .where(eq(events.accountId, accountId));

    // 3. Anonymize timeline entries
    await db.update(timelineEntries)
      .set({ body: '[REDACTED]', contactId: null })
      .where(eq(timelineEntries.accountId, accountId));

    // 4. Delete NPS responses
    await db.delete(surveyResponses).where(eq(surveyResponses.accountId, accountId));

    // 5. Anonymize account
    await db.update(accounts)
      .set({ name: '[REDACTED]', domain: null, crmData: {}, customFields: {} })
      .where(eq(accounts.id, accountId));

    // 6. Log erasure to audit trail
    await auditService.log({ action: 'gdpr_erasure', resourceType: 'account', resourceId: accountId });

    return { accountId, contactsDeleted, eventsAnonymized: true, completedAt: new Date() };
  }
}
```

**Testing:**
- Erasure: after erasure, account name is "[REDACTED]" and all PII fields are null/empty
- Erasure: all contacts for the account are deleted
- Erasure: event properties no longer contain email, name, or IP
- Erasure: survey responses deleted
- Erasure: audit log records the erasure action
- Data export: `/accounts/:id/export` returns a ZIP with all account data in JSON format
- Data export includes: account, contacts, health scores, events, timeline, survey responses

---

### Task 12.4: OData Reporting API

**What:** Implement OData v4-compatible reporting endpoints for health scores, accounts, and portfolio metrics. Enables native integration with Power BI, Tableau, and Excel.

**Design:**

```typescript
// apps/api/src/routes/odata/index.ts
import { ODataServer } from 'odata-v4-server';

@odata.type(Account)
class AccountsController extends ODataController {
  @odata.GET
  async getAccounts(@odata.query query: ODataQuery): Promise<Account[]> {
    // Translate OData $filter, $select, $orderby, $top, $skip to SQL
    const sqlQuery = translateODataToSql(query, 'accounts');
    return db.execute(sqlQuery);
  }
}

// Registered at: /odata/v4/accounts
// Supports: $filter, $select, $orderby, $top, $skip, $count, $expand
// Example: /odata/v4/accounts?$filter=healthBand eq 'red' and arr gt 50000&$orderby=arr desc&$top=10
```

**Testing:**
- `$filter=healthBand eq 'red'`: returns only red-band accounts
- `$orderby=arr desc`: accounts sorted by ARR descending
- `$top=10&$skip=20`: returns 10 accounts starting from offset 20
- `$select=name,arr,healthScore`: response contains only selected fields
- `$count=true`: response includes `@odata.count` with total record count
- Power BI: OData endpoint is consumable from Power BI Desktop (manual verification)
- Excel: OData endpoint is consumable from Excel Get Data (manual verification)

---

### Task 12.5: Rate Limiting and API Security

**What:** Implement per-tenant rate limiting, API key management for external API access, request signing for webhooks, and OWASP API Security Top 10 mitigations.

**Design:**

```typescript
// apps/api/src/middleware/rate-limit.ts
import { RateLimiterRedis } from 'rate-limiter-flexible';

const rateLimiter = new RateLimiterRedis({
  storeClient: redis,
  keyPrefix: 'rl',
  points: 1000,       // 1000 requests
  duration: 60,        // per 60 seconds
  blockDuration: 60,   // block for 60 seconds if exceeded
});

export async function rateLimitMiddleware(request: FastifyRequest, reply: FastifyReply) {
  const key = `${request.user?.tenantId}:${request.ip}`;
  try {
    await rateLimiter.consume(key);
  } catch {
    return reply.code(429).send({
      type: 'https://httpstatuses.com/429',
      title: 'Too Many Requests',
      status: 429,
      detail: 'Rate limit exceeded. Please retry after 60 seconds.',
    });
  }
}
```

**Testing:**
- 1001st request within 60 seconds returns 429
- Rate limit headers present: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- Different tenants have independent rate limits
- API key: create API key for tenant; use API key to authenticate requests
- API key rotation: revoke old key, issue new key; old key returns 401
- Webhook signing: outbound webhooks include `X-Signature` header with HMAC-SHA256
- BOLA prevention: user from tenant A requesting `/accounts/:id` where ID belongs to tenant B returns 404 (not 403)
- Input validation: malformed UUID in path parameter returns 400 (not 500)

---

### Task 12.6: Performance Optimization

**What:** Optimize database query performance, implement caching for frequently accessed data, and add connection pooling. Target: portfolio dashboard loads in <1 second for tenants with 1000+ accounts.

**Design:**

```typescript
// apps/api/src/services/cache-service.ts
export class CacheService {
  // Cache portfolio distribution for 5 minutes
  async getPortfolioDistribution(tenantId: string): Promise<PortfolioDistribution> {
    const cacheKey = `portfolio:${tenantId}`;
    const cached = await redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    const distribution = await healthScoreService.computeDistribution(tenantId);
    await redis.setex(cacheKey, 300, JSON.stringify(distribution));
    return distribution;
  }

  // Invalidate on health score change
  async onHealthScoreChanged(tenantId: string): Promise<void> {
    await redis.del(`portfolio:${tenantId}`);
  }
}
```

**Testing:**
- Portfolio dashboard API response time < 500ms with 1000 accounts (cached)
- Portfolio dashboard API response time < 2000ms with 1000 accounts (cold cache)
- Cache invalidation: health score change invalidates portfolio cache
- Account list API with pagination: < 200ms for page of 25 accounts
- Health score history API: < 300ms for 6 months of daily scores
- Database connection pooling: 50 concurrent requests handled without connection exhaustion
- Query plans: no sequential scans on large tables (verify with `EXPLAIN ANALYZE`)

---

## Definition of Done

Each phase is considered "done" when ALL of the following criteria are met:

### Code Quality
- [ ] All code passes TypeScript strict mode compilation with zero errors
- [ ] All code passes ESLint with zero warnings (project-configured rules)
- [ ] All new code has accompanying unit tests with >= 80% line coverage
- [ ] All API endpoints have integration tests covering success and error paths
- [ ] No `any` types in TypeScript (except justified and documented exceptions)

### Testing
- [ ] All unit tests pass (`pnpm turbo test`)
- [ ] All integration tests pass against a real PostgreSQL + Redis instance
- [ ] E2E tests pass for all new dashboard pages (Playwright)
- [ ] Performance benchmarks meet stated targets (documented in test output)
- [ ] Security tests: BOLA, injection, and authentication bypass tests pass

### Database
- [ ] All migrations are forward-only and idempotent
- [ ] RLS policies verified for every new table
- [ ] No N+1 query patterns (verified via query logging in integration tests)
- [ ] Indexes exist for all WHERE, JOIN, and ORDER BY columns used in production queries
- [ ] EXPLAIN ANALYZE for critical queries shows index usage (no seq scans on >1000 row tables)

### API
- [ ] OpenAPI 3.1 spec auto-generated and validates against all route schemas
- [ ] All endpoints return RFC 7807 error responses for 4xx/5xx
- [ ] Rate limiting active on all endpoints
- [ ] Tenant isolation verified (cross-tenant access returns 404)

### Documentation
- [ ] API endpoints documented in auto-generated OpenAPI spec
- [ ] Architecture decisions recorded in the codebase (ADR format)
- [ ] Environment setup instructions in repository root

### Deployment
- [ ] Docker Compose development environment starts with single command
- [ ] CI pipeline (GitHub Actions) passes: lint, typecheck, test, build
- [ ] Database migrations run successfully in CI against a clean database
- [ ] No secrets committed to repository (verified by CI secret scanning)

### Phase-Specific Acceptance
- Each phase has its own acceptance criteria documented in the task testing sections above
- A phase is not "done" until every task's testing checklist is fully satisfied

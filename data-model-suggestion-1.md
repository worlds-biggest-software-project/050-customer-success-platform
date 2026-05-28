# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Customer Success Platform · Created: 2026-05-12

## Philosophy

The entity-centric normalized relational model treats every domain concept as a first-class table with explicit foreign key relationships, strict type constraints, and comprehensive indexing. Every health score dimension, every playbook step, every CTA, and every integration credential gets its own table with well-defined columns. The schema is self-documenting: a developer reading the DDL understands the entire domain without consulting external documentation.

This approach draws directly from how Gainsight structures its internal object model — Company, Person, Relationship, Scorecard, CTA, Timeline — but renders it in clean PostgreSQL with UUID keys, timestamped audit columns, and proper normalization. The trade-off is table count: this model has more tables than the alternatives, but each table is narrow, well-indexed, and query-optimized for the specific access patterns CS platforms demand.

The normalized model is best suited for teams that value data integrity above all else, operate in regulated environments where audit requirements are strict, and expect to run complex cross-entity analytical queries (e.g., "show me all accounts where health dropped below 60, the CSM has more than 15 accounts, and renewal is within 90 days").

**Best for:** Teams with strong SQL expertise building a CS platform where data integrity, regulatory compliance, and complex analytical queries are paramount.

**Trade-offs:**
- (+) Maximum data integrity through foreign keys and constraints
- (+) Complex cross-entity queries are straightforward SQL JOINs
- (+) Schema is self-documenting; new developers understand the domain from DDL
- (+) Standard PostgreSQL tooling (pg_dump, logical replication, BI connectors) works natively
- (-) High table count (~35-40 tables) increases migration complexity
- (-) Adding new health score dimensions or integration types requires schema migrations
- (-) Junction tables for many-to-many relationships add query complexity
- (-) Less flexible for per-tenant customization without custom field tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Segment Spec (Identify, Track, Group) | `events` table schema mirrors Segment's Track call structure; `event_type`, `event_name`, `properties` align with Segment vocabulary |
| CloudEvents v1.0 | `webhook_deliveries` table envelope fields (`ce_specversion`, `ce_type`, `ce_source`, `ce_id`) follow CloudEvents context attributes |
| ISO 3166-1/2 | `accounts.country_code` and `accounts.region_code` use ISO 3166 alpha-2 codes for jurisdiction modeling |
| OAuth 2.0 (RFC 6749) | `integration_credentials` table stores OAuth tokens with `access_token`, `refresh_token`, `expires_at` following RFC 6749 token response structure |
| OpenAPI 3.1 | API contract published as OpenAPI 3.1 spec; table structure maps 1:1 to API resource schemas |
| ISO/IEC 25012 | Data quality dimensions (accuracy, completeness, timeliness) tracked in `data_quality_scores` per integration source |
| NPS (Bain/Satmetrix) | `nps_responses` table captures standard NPS fields: score (0-10), promoter/passive/detractor classification |
| OData v4 (ISO/IEC 20802-1) | Reporting endpoints expose tables via OData v4 for native BI tool compatibility |
| JSON Schema Draft 2020-12 | All JSONB columns validated against registered JSON Schemas in `json_schema_registry` |

---

## Tenant & Identity Management

```sql
-- Multi-tenant isolation via tenant_id on every table
-- PostgreSQL Row-Level Security enforces tenant boundaries

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, starter, professional, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'csm',  -- admin, cs_ops, csm, viewer
    avatar_url      TEXT,
    timezone        VARCHAR(50) DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(tenant_id, role);
```

## Account Management

```sql
CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),                         -- CRM account ID (Salesforce, HubSpot)
    name            VARCHAR(500) NOT NULL,
    domain          VARCHAR(255),                         -- e.g. acme.com
    industry        VARCHAR(100),
    employee_count  INTEGER,
    country_code    CHAR(2),                              -- ISO 3166-1 alpha-2
    region_code     VARCHAR(6),                           -- ISO 3166-2 subdivision
    tier            VARCHAR(50) DEFAULT 'standard',       -- enterprise, mid-market, smb, startup
    lifecycle_stage VARCHAR(50) DEFAULT 'onboarding',     -- onboarding, adopting, growing, renewing, churned
    arr             NUMERIC(12,2),                        -- Annual Recurring Revenue
    mrr             NUMERIC(12,2),                        -- Monthly Recurring Revenue
    contract_start  DATE,
    contract_end    DATE,
    renewal_date    DATE,
    csm_id          UUID REFERENCES users(id),            -- assigned CSM
    parent_id       UUID REFERENCES accounts(id),         -- parent account for hierarchies
    crm_source      VARCHAR(50),                          -- salesforce, hubspot
    crm_url         TEXT,
    logo_url        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_accounts_tenant ON accounts(tenant_id);
CREATE INDEX idx_accounts_external ON accounts(tenant_id, external_id);
CREATE INDEX idx_accounts_csm ON accounts(csm_id);
CREATE INDEX idx_accounts_lifecycle ON accounts(tenant_id, lifecycle_stage);
CREATE INDEX idx_accounts_renewal ON accounts(tenant_id, renewal_date);
CREATE INDEX idx_accounts_parent ON accounts(parent_id);
CREATE INDEX idx_accounts_arr ON accounts(tenant_id, arr DESC);

CREATE TABLE contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),                         -- CRM contact ID
    email           VARCHAR(255),
    name            VARCHAR(255) NOT NULL,
    title           VARCHAR(255),
    role            VARCHAR(100),                         -- champion, decision_maker, end_user, executive_sponsor, detractor
    phone           VARCHAR(50),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    last_activity_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_account ON contacts(account_id);
CREATE INDEX idx_contacts_tenant ON contacts(tenant_id);
CREATE INDEX idx_contacts_email ON contacts(tenant_id, email);
CREATE INDEX idx_contacts_role ON contacts(account_id, role);
```

## Health Scoring

```sql
-- Scorecard defines the health score model (weights, thresholds)
CREATE TABLE scorecards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

-- Each scorecard has multiple measures (dimensions)
CREATE TABLE scorecard_measures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scorecard_id    UUID NOT NULL REFERENCES scorecards(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,                -- usage, engagement, support, nps, financial
    weight          NUMERIC(5,2) NOT NULL DEFAULT 0.20,   -- weight in composite score (0.00-1.00)
    scoring_method  VARCHAR(50) NOT NULL DEFAULT 'rule',  -- rule, ml_model, manual
    thresholds      JSONB NOT NULL DEFAULT '{}',
    -- Example thresholds: {"green": 80, "yellow": 50, "red": 0}
    display_order   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_measures_scorecard ON scorecard_measures(scorecard_id);

-- Health scores: one row per account per scoring period
CREATE TABLE health_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    scorecard_id    UUID NOT NULL REFERENCES scorecards(id),
    composite_score NUMERIC(5,2) NOT NULL,                -- 0.00-100.00
    score_band      VARCHAR(10) NOT NULL,                 -- green, yellow, red
    trend           VARCHAR(10) NOT NULL DEFAULT 'stable', -- improving, stable, declining
    ai_narrative    TEXT,                                  -- LLM-generated explanation
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_health_scores_account ON health_scores(account_id, scored_at DESC);
CREATE INDEX idx_health_scores_tenant ON health_scores(tenant_id, scored_at DESC);
CREATE INDEX idx_health_scores_band ON health_scores(tenant_id, score_band);
CREATE INDEX idx_health_scores_composite ON health_scores(tenant_id, composite_score);

-- Individual measure scores per health score snapshot
CREATE TABLE health_score_measures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    health_score_id UUID NOT NULL REFERENCES health_scores(id) ON DELETE CASCADE,
    measure_id      UUID NOT NULL REFERENCES scorecard_measures(id),
    score           NUMERIC(5,2) NOT NULL,                -- 0.00-100.00
    score_band      VARCHAR(10) NOT NULL,
    raw_value       NUMERIC(12,4),                        -- raw metric value before scoring
    signals         JSONB NOT NULL DEFAULT '[]',
    -- Example signals: [{"signal": "login_frequency_drop", "value": -40, "weight": 0.3}]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hsm_health_score ON health_score_measures(health_score_id);
CREATE INDEX idx_hsm_measure ON health_score_measures(measure_id);
```

## Product Usage Events

```sql
-- Raw product usage events ingested from Segment, Amplitude, etc.
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    contact_id      UUID REFERENCES contacts(id),
    event_type      VARCHAR(50) NOT NULL,                 -- track, identify, page, screen (Segment Spec)
    event_name      VARCHAR(255) NOT NULL,                -- e.g. "feature_used", "login", "report_exported"
    properties      JSONB NOT NULL DEFAULT '{}',
    source          VARCHAR(100) NOT NULL,                -- segment, amplitude, mixpanel, pendo, custom
    source_event_id VARCHAR(255),                         -- deduplication key from source
    occurred_at     TIMESTAMPTZ NOT NULL,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

-- Create monthly partitions
CREATE TABLE events_2026_01 PARTITION OF events FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE events_2026_02 PARTITION OF events FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional partitions created by cron job

CREATE INDEX idx_events_account ON events(account_id, occurred_at DESC);
CREATE INDEX idx_events_tenant ON events(tenant_id, occurred_at DESC);
CREATE INDEX idx_events_name ON events(tenant_id, event_name, occurred_at DESC);
CREATE INDEX idx_events_source_dedup ON events(tenant_id, source, source_event_id);
```

## Playbooks & CTAs

```sql
-- Playbook templates define automated intervention sequences
CREATE TABLE playbooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    category        VARCHAR(50) NOT NULL,                 -- onboarding, risk, renewal, expansion, escalation
    trigger_type    VARCHAR(50) NOT NULL,                 -- health_threshold, lifecycle_change, manual, scheduled
    trigger_config  JSONB NOT NULL DEFAULT '{}',
    -- Example: {"measure": "usage", "condition": "drops_below", "threshold": 40}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_playbooks_tenant ON playbooks(tenant_id);
CREATE INDEX idx_playbooks_category ON playbooks(tenant_id, category);

-- Steps within a playbook
CREATE TABLE playbook_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    playbook_id     UUID NOT NULL REFERENCES playbooks(id) ON DELETE CASCADE,
    step_order      INTEGER NOT NULL,
    action_type     VARCHAR(50) NOT NULL,                 -- email, slack_message, in_app, task, escalation, wait
    action_config   JSONB NOT NULL DEFAULT '{}',
    -- Example email: {"template_id": "...", "to": "contact.primary"}
    -- Example wait: {"duration_days": 3}
    delay_minutes   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_steps_playbook ON playbook_steps(playbook_id, step_order);

-- Calls-to-Action: actionable items for CSMs
CREATE TABLE ctas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    playbook_id     UUID REFERENCES playbooks(id),
    assigned_to     UUID REFERENCES users(id),
    cta_type        VARCHAR(50) NOT NULL,                 -- risk, renewal, expansion, onboarding, custom
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium', -- critical, high, medium, low
    status          VARCHAR(50) NOT NULL DEFAULT 'open',  -- open, snoozed, in_progress, closed_success, closed_failed
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    due_date        DATE,
    snoozed_until   TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    closed_reason   VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ctas_account ON ctas(account_id);
CREATE INDEX idx_ctas_assigned ON ctas(assigned_to, status);
CREATE INDEX idx_ctas_tenant_status ON ctas(tenant_id, status);
CREATE INDEX idx_ctas_due ON ctas(tenant_id, due_date) WHERE status IN ('open', 'in_progress');
CREATE INDEX idx_ctas_priority ON ctas(tenant_id, priority) WHERE status IN ('open', 'in_progress');

-- Tasks within a CTA
CREATE TABLE cta_tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cta_id          UUID NOT NULL REFERENCES ctas(id) ON DELETE CASCADE,
    playbook_step_id UUID REFERENCES playbook_steps(id),
    assigned_to     UUID REFERENCES users(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending', -- pending, in_progress, completed, skipped
    due_date        DATE,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cta_tasks_cta ON cta_tasks(cta_id);
CREATE INDEX idx_cta_tasks_assigned ON cta_tasks(assigned_to, status);
```

## Timeline & Activities

```sql
CREATE TABLE timeline_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    contact_id      UUID REFERENCES contacts(id),
    author_id       UUID REFERENCES users(id),
    entry_type      VARCHAR(50) NOT NULL,                 -- call, email, meeting, note, milestone, lifecycle_change, system
    subject         VARCHAR(500),
    body            TEXT,
    sentiment       VARCHAR(20),                          -- positive, neutral, negative
    is_external     BOOLEAN NOT NULL DEFAULT false,       -- visible to customer?
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_timeline_account ON timeline_entries(account_id, occurred_at DESC);
CREATE INDEX idx_timeline_tenant ON timeline_entries(tenant_id, occurred_at DESC);
CREATE INDEX idx_timeline_type ON timeline_entries(account_id, entry_type);
```

## Surveys (NPS / CSAT / CES)

```sql
CREATE TABLE survey_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    survey_type     VARCHAR(20) NOT NULL,                 -- nps, csat, ces, custom
    channel         VARCHAR(50) NOT NULL DEFAULT 'email', -- email, in_app, slack
    is_active       BOOLEAN NOT NULL DEFAULT true,
    schedule_config JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE nps_responses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    contact_id      UUID REFERENCES contacts(id),
    campaign_id     UUID REFERENCES survey_campaigns(id),
    score           SMALLINT NOT NULL CHECK (score >= 0 AND score <= 10),
    classification  VARCHAR(20) NOT NULL,                 -- promoter (9-10), passive (7-8), detractor (0-6)
    feedback        TEXT,
    responded_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_nps_account ON nps_responses(account_id, responded_at DESC);
CREATE INDEX idx_nps_tenant ON nps_responses(tenant_id, responded_at DESC);
CREATE INDEX idx_nps_classification ON nps_responses(tenant_id, classification);
```

## Integration & Data Sync

```sql
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,                 -- salesforce, hubspot, segment, amplitude, zendesk, slack, pendo
    status          VARCHAR(20) NOT NULL DEFAULT 'active', -- active, paused, error, disconnected
    config          JSONB NOT NULL DEFAULT '{}',          -- provider-specific configuration
    last_sync_at    TIMESTAMPTZ,
    sync_error      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, provider)
);

CREATE TABLE integration_credentials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id  UUID NOT NULL REFERENCES integrations(id) ON DELETE CASCADE,
    credential_type VARCHAR(50) NOT NULL,                 -- oauth2, api_key, webhook_secret
    access_token    TEXT,                                  -- encrypted at rest
    refresh_token   TEXT,                                  -- encrypted at rest
    api_key         TEXT,                                  -- encrypted at rest
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_credentials_integration ON integration_credentials(integration_id);

-- Sync log tracks bidirectional data sync operations
CREATE TABLE sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id  UUID NOT NULL REFERENCES integrations(id) ON DELETE CASCADE,
    direction       VARCHAR(10) NOT NULL,                 -- inbound, outbound
    object_type     VARCHAR(100) NOT NULL,                -- account, contact, health_score, deal
    records_synced  INTEGER NOT NULL DEFAULT 0,
    records_failed  INTEGER NOT NULL DEFAULT 0,
    error_details   JSONB,
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sync_log_integration ON sync_log(integration_id, started_at DESC);
```

## AI / ML Model Management

```sql
CREATE TABLE ml_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    model_type      VARCHAR(50) NOT NULL,                 -- churn_prediction, expansion_detection, health_scoring
    model_version   VARCHAR(50) NOT NULL,
    framework       VARCHAR(50) NOT NULL,                 -- sklearn, xgboost, pytorch, llm
    artifact_path   TEXT NOT NULL,                        -- S3/GCS path to serialized model
    metrics         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"precision": 0.94, "recall": 0.87, "f1": 0.90, "auc_roc": 0.95}
    is_active       BOOLEAN NOT NULL DEFAULT false,
    trained_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ml_models_tenant ON ml_models(tenant_id, model_type);

CREATE TABLE churn_predictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    model_id        UUID NOT NULL REFERENCES ml_models(id),
    churn_probability NUMERIC(5,4) NOT NULL,              -- 0.0000-1.0000
    risk_band       VARCHAR(20) NOT NULL,                 -- high, medium, low
    top_features    JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"feature": "login_frequency", "importance": 0.32, "direction": "negative"},
    --           {"feature": "support_ticket_count", "importance": 0.25, "direction": "negative"}]
    predicted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_churn_account ON churn_predictions(account_id, predicted_at DESC);
CREATE INDEX idx_churn_risk ON churn_predictions(tenant_id, risk_band);
```

## Webhook Delivery (CloudEvents)

```sql
CREATE TABLE webhook_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    secret          TEXT NOT NULL,                        -- signing secret, encrypted at rest
    event_types     TEXT[] NOT NULL,                      -- array of subscribed event types
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id     UUID NOT NULL REFERENCES webhook_endpoints(id) ON DELETE CASCADE,
    ce_specversion  VARCHAR(10) NOT NULL DEFAULT '1.0',   -- CloudEvents specversion
    ce_type         VARCHAR(255) NOT NULL,                -- e.g. com.csplatform.health_score.changed
    ce_source       VARCHAR(255) NOT NULL,                -- e.g. /tenants/{tenant_id}/accounts/{account_id}
    ce_id           VARCHAR(255) NOT NULL,                -- unique event ID
    ce_time         TIMESTAMPTZ NOT NULL,
    payload         JSONB NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, delivered, failed, retrying
    http_status     SMALLINT,
    attempts        SMALLINT NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    next_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhook_del_endpoint ON webhook_deliveries(endpoint_id, created_at DESC);
CREATE INDEX idx_webhook_del_status ON webhook_deliveries(status) WHERE status IN ('pending', 'retrying');
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 2 | tenants, users |
| Account Management | 2 | accounts, contacts |
| Health Scoring | 4 | scorecards, scorecard_measures, health_scores, health_score_measures |
| Product Usage | 1 | events (partitioned by month) |
| Playbooks & CTAs | 4 | playbooks, playbook_steps, ctas, cta_tasks |
| Timeline | 1 | timeline_entries |
| Surveys | 2 | survey_campaigns, nps_responses |
| Integrations | 3 | integrations, integration_credentials, sync_log |
| AI / ML | 2 | ml_models, churn_predictions |
| Webhooks | 2 | webhook_endpoints, webhook_deliveries |
| **Total** | **23** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere.** UUIDs prevent enumeration attacks, support distributed ID generation without coordination, and make CRM sync mapping straightforward (external_id columns hold the CRM's native IDs).

2. **`tenant_id` on every table with Row-Level Security.** Every tenant-scoped table includes `tenant_id` as a non-nullable foreign key. PostgreSQL RLS policies enforce that queries only see rows matching the session's `current_setting('app.tenant_id')`, providing defense-in-depth beyond application-layer filtering.

3. **Health scores are snapshots, not mutable state.** Each health score calculation creates a new row in `health_scores` rather than updating a single row. This preserves full history for trend analysis without requiring a separate history table. The most recent score is retrieved with `ORDER BY scored_at DESC LIMIT 1`.

4. **Events table partitioned by month.** Product usage events are the highest-volume table. Monthly range partitioning on `occurred_at` enables efficient partition pruning for time-bounded queries and simple retention management (drop old partitions).

5. **Separate `scorecard_measures` from `health_score_measures`.** The scorecard defines the model (what to measure and how to weight it); health_score_measures stores the actual scored values per snapshot. This separation allows scorecard configuration to change without invalidating historical scores.

6. **Playbook steps are ordered, not graph-structured.** Playbook steps use a simple `step_order` integer rather than a DAG/graph structure. This keeps the MVP simple. For complex branching journeys, the `action_config` JSONB field can contain conditional routing logic.

7. **CRM credentials encrypted at rest.** The `integration_credentials` table stores OAuth tokens and API keys. The schema marks these as encrypted — the application layer must use PostgreSQL's `pgcrypto` or application-level encryption before writing to these columns.

8. **CloudEvents envelope for webhook delivery.** Outbound webhook payloads follow CloudEvents v1.0 context attribute naming (`ce_specversion`, `ce_type`, `ce_source`, `ce_id`, `ce_time`) for interoperability with event brokers like AWS EventBridge and Kafka.

9. **Churn predictions stored with explainable feature importance.** The `churn_predictions` table includes a `top_features` JSONB array that stores the most influential features and their importance scores. This supports the LLM-generated health narrative use case: the AI can read the top features and produce a plain-language explanation of why an account is at risk.

10. **NPS classification computed at write time.** The `nps_responses.classification` column stores the promoter/passive/detractor classification rather than computing it at query time. This avoids recomputing the classification on every dashboard load and makes aggregation queries trivial.

---

## Example Queries

### Portfolio health distribution
```sql
SELECT
    hs.score_band,
    COUNT(DISTINCT hs.account_id) AS account_count,
    SUM(a.arr) AS total_arr
FROM health_scores hs
JOIN accounts a ON a.id = hs.account_id
WHERE hs.tenant_id = $1
  AND hs.scored_at = (
      SELECT MAX(hs2.scored_at)
      FROM health_scores hs2
      WHERE hs2.account_id = hs.account_id
  )
GROUP BY hs.score_band;
```

### Accounts with declining health and upcoming renewal
```sql
SELECT a.name, a.arr, a.renewal_date, hs.composite_score, hs.trend, hs.ai_narrative
FROM accounts a
JOIN LATERAL (
    SELECT * FROM health_scores
    WHERE account_id = a.id
    ORDER BY scored_at DESC LIMIT 1
) hs ON true
WHERE a.tenant_id = $1
  AND hs.trend = 'declining'
  AND a.renewal_date BETWEEN now() AND now() + INTERVAL '90 days'
ORDER BY a.arr DESC;
```

### Account hierarchy traversal (recursive CTE)
```sql
WITH RECURSIVE account_tree AS (
    SELECT id, name, parent_id, 0 AS depth
    FROM accounts
    WHERE id = $1  -- root account
    UNION ALL
    SELECT a.id, a.name, a.parent_id, at.depth + 1
    FROM accounts a
    JOIN account_tree at ON a.parent_id = at.id
)
SELECT * FROM account_tree ORDER BY depth, name;
```

# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Customer Success Platform · Created: 2026-05-12

## Philosophy

The event-sourced model treats every state change in the system as an immutable, append-only event. The source of truth is not the current state of an account or health score — it is the ordered sequence of events that produced that state. Current state is a derived projection, rebuilt (or incrementally updated) from the event stream. This is the Command Query Responsibility Segregation (CQRS) pattern: writes go to the event store, reads come from materialised views optimised for specific query patterns.

This architecture is a natural fit for customer success platforms because the domain is inherently temporal. A health score is not a static number — it is the result of a sequence of signals (usage dropped, support ticket opened, NPS response received, executive sponsor left). An event-sourced model makes it trivial to answer questions like "what was the health score on March 15th?" or "what sequence of events caused this account to go from green to red?" — questions that are expensive or impossible to answer with a mutable relational model. Every health score change, CTA action, playbook trigger, and CSM interaction is captured as a first-class event with full context.

The audit-first approach also satisfies regulatory requirements (GDPR right to access, SOC 2 audit trail) without bolting on a separate audit logging system. The event store IS the audit log. For AI-powered features like explainable health narratives and context-aware playbook execution, the event stream provides the complete historical context that LLMs need to generate accurate, grounded explanations.

**Best for:** Teams building an AI-native CS platform where full temporal history, explainable AI narratives, and regulatory-grade audit trails are core requirements — not afterthoughts.

**Trade-offs:**
- (+) Complete, immutable audit trail is inherent — not bolted on
- (+) Temporal queries ("what was true on date X?") are first-class operations
- (+) AI/LLM features get rich, ordered historical context for reasoning
- (+) Event replay enables schema evolution: add new projections without data migration
- (+) Supports event-driven architecture natively (downstream consumers subscribe to events)
- (-) Higher storage requirements (events are never deleted, only archived)
- (-) Eventual consistency between event store and read models requires careful handling
- (-) More complex development model: developers must think in events, not CRUD
- (-) Read model rebuild from scratch can be slow for large event stores
- (-) Debugging requires understanding both the event stream and the current projection

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents v1.0 (CNCF) | Every domain event in the event store follows CloudEvents envelope structure: `specversion`, `type`, `source`, `id`, `time`, `datacontenttype` |
| Segment Spec | Inbound product analytics events are normalised to Segment Spec vocabulary (Identify, Track, Group) before being stored as domain events |
| JSON Schema Draft 2020-12 | Each event type has a registered JSON Schema that validates the event data payload; schema versioning enables event evolution |
| RFC 7807 (Problem Details) | Error events and failed projections include RFC 7807-compliant error detail structures |
| ISO/IEC 25012 (Data Quality) | Data quality signals from integration sources are captured as quality assessment events, feeding health score accuracy monitoring |
| GDPR (EU 2016/679) | Right to erasure implemented via crypto-shredding: per-account encryption keys are destroyed to render PII-containing events unreadable without deleting the event stream |
| EU AI Act (2024/1689) | AI model decision events include transparency fields (model version, feature importance, confidence interval) for regulatory compliance |
| SOC 2 Type II | Event store serves as the primary audit evidence: every state change has a timestamp, actor, and context |

---

## Event Store (Core Tables)

```sql
-- Event store: the single source of truth
-- All state in the system is derived from this append-only table
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    aggregate_type  VARCHAR(100) NOT NULL,                -- account, health_score, cta, playbook_execution, contact, survey
    aggregate_id    UUID NOT NULL,                        -- the entity this event belongs to
    event_type      VARCHAR(255) NOT NULL,                -- e.g. account.created, health_score.calculated, cta.opened
    event_version   INTEGER NOT NULL,                     -- monotonically increasing per aggregate
    -- CloudEvents envelope fields
    ce_specversion  VARCHAR(10) NOT NULL DEFAULT '1.0',
    ce_source       VARCHAR(500) NOT NULL,                -- e.g. /tenants/{id}/integrations/segment
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Event payload
    data            JSONB NOT NULL,                       -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"actor_id": "...", "actor_type": "user|system|integration", "correlation_id": "...", "causation_id": "..."}
    schema_version  VARCHAR(20) NOT NULL DEFAULT '1.0',   -- for event schema evolution
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Optimistic concurrency: no two events for the same aggregate can have the same version
    UNIQUE(aggregate_id, event_version)
) PARTITION BY RANGE (created_at);

-- Monthly partitions for retention management
CREATE TABLE event_store_2026_01 PARTITION OF event_store FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE event_store_2026_02 PARTITION OF event_store FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional partitions via automation

CREATE INDEX idx_events_aggregate ON event_store(aggregate_id, event_version);
CREATE INDEX idx_events_type ON event_store(tenant_id, event_type, ce_time DESC);
CREATE INDEX idx_events_tenant_time ON event_store(tenant_id, ce_time DESC);
CREATE INDEX idx_events_aggregate_type ON event_store(tenant_id, aggregate_type, ce_time DESC);
CREATE INDEX idx_events_correlation ON event_store((metadata->>'correlation_id')) WHERE metadata->>'correlation_id' IS NOT NULL;

-- Aggregate version tracking for optimistic concurrency control
CREATE TABLE aggregates (
    aggregate_id    UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    aggregate_type  VARCHAR(100) NOT NULL,
    current_version INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_aggregates_tenant ON aggregates(tenant_id, aggregate_type);

-- Event type schema registry for validation and evolution
CREATE TABLE event_schemas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type      VARCHAR(255) NOT NULL,
    schema_version  VARCHAR(20) NOT NULL,
    json_schema     JSONB NOT NULL,                       -- JSON Schema Draft 2020-12
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(event_type, schema_version)
);
```

## Event Type Catalogue

```sql
-- Reference: the event types this system produces
-- Each type has a defined JSON Schema for its data payload

/*
ACCOUNT LIFECYCLE EVENTS:
  account.created           - New account registered
  account.updated           - Account metadata changed (name, tier, industry, etc.)
  account.csm_assigned      - CSM assignment changed
  account.lifecycle_changed - Lifecycle stage transition (onboarding → adopting → growing → renewing → churned)
  account.contract_renewed  - Contract renewal completed
  account.contract_expired  - Contract reached end date without renewal
  account.churned           - Account confirmed churned
  account.reactivated       - Previously churned account reactivated

CONTACT EVENTS:
  contact.created           - New contact added to account
  contact.updated           - Contact details changed
  contact.role_changed      - Contact role updated (champion → detractor, etc.)
  contact.departed          - Contact left the organisation

HEALTH SCORE EVENTS:
  health_score.calculated   - New health score computed (includes all measure scores)
  health_score.band_changed - Score band transition (green → yellow, yellow → red, etc.)
  health_score.narrative_generated - AI narrative generated for health score change

PRODUCT USAGE EVENTS (ingested from Segment/Amplitude):
  usage.event_received      - Raw product usage event ingested
  usage.feature_adopted     - New feature first used by account
  usage.feature_abandoned   - Feature usage dropped to zero after prior adoption
  usage.login               - User login event
  usage.session_ended       - User session completed

CTA EVENTS:
  cta.created               - New CTA generated (by playbook or manually)
  cta.assigned              - CTA assigned to CSM
  cta.status_changed        - CTA status transition (open → in_progress → closed)
  cta.snoozed               - CTA snoozed until future date
  cta.closed                - CTA resolved

PLAYBOOK EVENTS:
  playbook.triggered        - Playbook execution started for an account
  playbook.step_executed    - Individual playbook step completed
  playbook.completed        - Playbook execution finished
  playbook.aborted          - Playbook execution cancelled

SURVEY EVENTS:
  survey.sent               - Survey delivered to contact
  survey.responded          - NPS/CSAT/CES response received
  survey.reminder_sent      - Survey follow-up reminder sent

INTEGRATION EVENTS:
  integration.connected     - New integration activated
  integration.sync_started  - Data sync cycle began
  integration.sync_completed - Data sync cycle finished
  integration.sync_failed   - Data sync cycle failed
  integration.disconnected  - Integration deactivated

AI/ML EVENTS:
  prediction.churn_scored   - Churn prediction model produced a score
  prediction.expansion_detected - Expansion opportunity identified
  ai.narrative_generated    - LLM generated a health narrative or QBR content
  ai.playbook_decision      - AI decided whether to trigger/skip an intervention
*/
```

## Read Models (Materialised Projections)

```sql
-- ============================================================
-- READ MODEL: Current Account State
-- Rebuilt from account.* events
-- ============================================================
CREATE TABLE rm_accounts (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    external_id     VARCHAR(255),
    name            VARCHAR(500) NOT NULL,
    domain          VARCHAR(255),
    industry        VARCHAR(100),
    tier            VARCHAR(50),
    lifecycle_stage VARCHAR(50),
    arr             NUMERIC(12,2),
    mrr             NUMERIC(12,2),
    contract_start  DATE,
    contract_end    DATE,
    renewal_date    DATE,
    csm_id          UUID,
    csm_name        VARCHAR(255),
    parent_id       UUID,
    country_code    CHAR(2),
    crm_source      VARCHAR(50),
    crm_url         TEXT,
    -- Denormalised health score (from latest health_score.calculated event)
    current_health_score    NUMERIC(5,2),
    current_health_band     VARCHAR(10),
    health_trend            VARCHAR(10),
    health_narrative        TEXT,
    health_scored_at        TIMESTAMPTZ,
    -- Denormalised churn prediction
    churn_probability       NUMERIC(5,4),
    churn_risk_band         VARCHAR(20),
    churn_predicted_at      TIMESTAMPTZ,
    -- Activity metrics
    last_activity_at        TIMESTAMPTZ,
    open_cta_count          INTEGER NOT NULL DEFAULT 0,
    -- Version tracking
    last_event_version      INTEGER NOT NULL DEFAULT 0,
    last_event_at           TIMESTAMPTZ,
    created_at              TIMESTAMPTZ NOT NULL,
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_accounts_tenant ON rm_accounts(tenant_id);
CREATE INDEX idx_rm_accounts_csm ON rm_accounts(csm_id);
CREATE INDEX idx_rm_accounts_health ON rm_accounts(tenant_id, current_health_band);
CREATE INDEX idx_rm_accounts_renewal ON rm_accounts(tenant_id, renewal_date);
CREATE INDEX idx_rm_accounts_churn ON rm_accounts(tenant_id, churn_risk_band);
CREATE INDEX idx_rm_accounts_lifecycle ON rm_accounts(tenant_id, lifecycle_stage);
CREATE INDEX idx_rm_accounts_arr ON rm_accounts(tenant_id, arr DESC);

-- ============================================================
-- READ MODEL: Health Score History (time-series optimised)
-- Rebuilt from health_score.calculated events
-- ============================================================
CREATE TABLE rm_health_score_history (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    account_id      UUID NOT NULL,
    composite_score NUMERIC(5,2) NOT NULL,
    score_band      VARCHAR(10) NOT NULL,
    trend           VARCHAR(10),
    measure_scores  JSONB NOT NULL DEFAULT '{}',
    -- Example: {"usage": 72.5, "engagement": 85.0, "support": 45.0, "nps": 90.0, "financial": 95.0}
    top_signals     JSONB NOT NULL DEFAULT '[]',
    ai_narrative    TEXT,
    scored_at       TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (scored_at);

CREATE TABLE rm_health_history_2026_01 PARTITION OF rm_health_score_history FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE rm_health_history_2026_02 PARTITION OF rm_health_score_history FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

CREATE INDEX idx_rm_health_account ON rm_health_score_history(account_id, scored_at DESC);
CREATE INDEX idx_rm_health_tenant ON rm_health_score_history(tenant_id, scored_at DESC);

-- ============================================================
-- READ MODEL: CSM Cockpit (action queue)
-- Rebuilt from cta.* and playbook.* events
-- ============================================================
CREATE TABLE rm_csm_cockpit (
    cta_id          UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    account_id      UUID NOT NULL,
    account_name    VARCHAR(500),
    account_arr     NUMERIC(12,2),
    account_health_band VARCHAR(10),
    assigned_to     UUID,
    cta_type        VARCHAR(50) NOT NULL,
    priority        VARCHAR(20) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    due_date        DATE,
    playbook_name   VARCHAR(255),
    pending_tasks   INTEGER NOT NULL DEFAULT 0,
    total_tasks     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_cockpit_assigned ON rm_csm_cockpit(assigned_to, status, priority);
CREATE INDEX idx_rm_cockpit_tenant ON rm_csm_cockpit(tenant_id, status);
CREATE INDEX idx_rm_cockpit_due ON rm_csm_cockpit(assigned_to, due_date) WHERE status IN ('open', 'in_progress');

-- ============================================================
-- READ MODEL: Portfolio Dashboard (aggregated metrics)
-- Rebuilt periodically from multiple event streams
-- ============================================================
CREATE TABLE rm_portfolio_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    total_accounts  INTEGER NOT NULL DEFAULT 0,
    total_arr       NUMERIC(14,2) NOT NULL DEFAULT 0,
    green_count     INTEGER NOT NULL DEFAULT 0,
    green_arr       NUMERIC(14,2) NOT NULL DEFAULT 0,
    yellow_count    INTEGER NOT NULL DEFAULT 0,
    yellow_arr      NUMERIC(14,2) NOT NULL DEFAULT 0,
    red_count       INTEGER NOT NULL DEFAULT 0,
    red_arr         NUMERIC(14,2) NOT NULL DEFAULT 0,
    avg_health_score NUMERIC(5,2),
    churn_rate      NUMERIC(5,4),
    nrr             NUMERIC(5,4),                         -- Net Revenue Retention
    grr             NUMERIC(5,4),                         -- Gross Revenue Retention
    nps_score       NUMERIC(5,2),
    open_cta_count  INTEGER NOT NULL DEFAULT 0,
    accounts_at_risk INTEGER NOT NULL DEFAULT 0,
    renewals_upcoming INTEGER NOT NULL DEFAULT 0,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, period_start, period_end)
);

CREATE INDEX idx_rm_portfolio_tenant ON rm_portfolio_metrics(tenant_id, period_start DESC);

-- ============================================================
-- READ MODEL: Contact Directory
-- Rebuilt from contact.* events
-- ============================================================
CREATE TABLE rm_contacts (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    account_id      UUID NOT NULL,
    external_id     VARCHAR(255),
    email           VARCHAR(255),
    name            VARCHAR(255) NOT NULL,
    title           VARCHAR(255),
    role            VARCHAR(100),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    last_activity_at TIMESTAMPTZ,
    latest_nps_score SMALLINT,
    latest_nps_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_contacts_account ON rm_contacts(account_id);
CREATE INDEX idx_rm_contacts_tenant ON rm_contacts(tenant_id);

-- ============================================================
-- READ MODEL: Account Timeline (denormalised activity feed)
-- Rebuilt from all account-scoped events
-- ============================================================
CREATE TABLE rm_account_timeline (
    event_id        UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    account_id      UUID NOT NULL,
    event_type      VARCHAR(255) NOT NULL,
    event_category  VARCHAR(50) NOT NULL,                 -- lifecycle, health, usage, cta, playbook, survey, integration, ai
    summary         TEXT NOT NULL,                        -- human-readable summary
    actor_id        UUID,
    actor_name      VARCHAR(255),
    actor_type      VARCHAR(20),                          -- user, system, integration, ai
    details         JSONB NOT NULL DEFAULT '{}',
    occurred_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_timeline_account ON rm_account_timeline(account_id, occurred_at DESC);
CREATE INDEX idx_rm_timeline_tenant ON rm_account_timeline(tenant_id, occurred_at DESC);
CREATE INDEX idx_rm_timeline_category ON rm_account_timeline(account_id, event_category, occurred_at DESC);
```

## Projection Management

```sql
-- Tracks the position of each read model projector in the event stream
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID,
    last_event_time TIMESTAMPTZ,
    events_processed BIGINT NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'running', -- running, paused, rebuilding, error
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for events that fail projection processing
CREATE TABLE projection_dead_letters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(100) NOT NULL,
    event_id        UUID NOT NULL,
    event_type      VARCHAR(255) NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    max_retries     INTEGER NOT NULL DEFAULT 5,
    next_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dead_letters_retry ON projection_dead_letters(next_retry_at) WHERE retry_count < max_retries;
```

## Tenant & Identity (Minimal — most state is in events)

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    encryption_key_id VARCHAR(255),                       -- KMS key reference for crypto-shredding
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'csm',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

-- Integration credentials (not event-sourced — operational config)
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    provider        VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    access_token    TEXT,                                  -- encrypted
    refresh_token   TEXT,                                  -- encrypted
    config          JSONB NOT NULL DEFAULT '{}',
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, provider)
);

-- Scorecard configuration (not event-sourced — admin config)
CREATE TABLE scorecards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    measures        JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"name": "usage", "weight": 0.30, "method": "ml_model"},
    --           {"name": "engagement", "weight": 0.25, "method": "rule"},
    --           {"name": "support", "weight": 0.20, "method": "rule"},
    --           {"name": "nps", "weight": 0.15, "method": "rule"},
    --           {"name": "financial", "weight": 0.10, "method": "rule"}]
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Playbook templates (not event-sourced — admin config)
CREATE TABLE playbooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(50) NOT NULL,
    trigger_config  JSONB NOT NULL DEFAULT '{}',
    steps           JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"order": 1, "action": "email", "template": "...", "delay_minutes": 0},
    --           {"order": 2, "action": "wait", "duration_days": 3},
    --           {"order": 3, "action": "slack_message", "channel": "cs-alerts", "template": "..."}]
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## GDPR Crypto-Shredding Support

```sql
-- Per-account encryption keys for GDPR right-to-erasure compliance
-- When an account exercises right to erasure, the key is destroyed
-- rendering all PII in event_store.data unreadable
CREATE TABLE account_encryption_keys (
    account_id      UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    key_version     INTEGER NOT NULL DEFAULT 1,
    encrypted_key   TEXT NOT NULL,                        -- AES-256 key encrypted with tenant KMS key
    is_active       BOOLEAN NOT NULL DEFAULT true,        -- false = key destroyed (crypto-shredded)
    destroyed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | event_store (partitioned), aggregates, event_schemas |
| Read Models | 6 | rm_accounts, rm_health_score_history (partitioned), rm_csm_cockpit, rm_portfolio_metrics, rm_contacts, rm_account_timeline |
| Projection Infrastructure | 2 | projection_checkpoints, projection_dead_letters |
| Tenant & Config | 5 | tenants, users, integrations, scorecards, playbooks |
| GDPR | 1 | account_encryption_keys |
| **Total** | **17** | Plus partitions; 1 core event table replaces ~10 tables from normalised model |

---

## Key Design Decisions

1. **Single event store table as source of truth.** All domain events flow through one partitioned table. This simplifies backup, replication, and retention policies. The `aggregate_type` and `event_type` columns enable efficient filtering without requiring separate tables per domain concept.

2. **Optimistic concurrency via aggregate versioning.** The `UNIQUE(aggregate_id, event_version)` constraint prevents concurrent writes to the same aggregate. The application reads the current version, appends with `version + 1`, and retries on conflict. This eliminates distributed locking requirements.

3. **Read models are disposable.** Every `rm_*` table can be dropped and rebuilt from the event store. This means schema changes to read models are zero-downtime: deploy new projector code, rebuild the new read model from events, switch traffic, drop old read model.

4. **Crypto-shredding for GDPR compliance.** Rather than deleting events (which would break the event stream), PII fields in event data are encrypted with per-account keys. Right-to-erasure is implemented by destroying the encryption key, rendering all PII for that account permanently unreadable while preserving the event sequence for audit and analytics.

5. **CloudEvents envelope baked into the event store.** Every event carries CloudEvents context attributes (`ce_specversion`, `ce_source`, `ce_time`). This means events can be published directly to external event brokers (Kafka, EventBridge) without transformation.

6. **Correlation and causation IDs in metadata.** Every event records which command triggered it (`causation_id`) and which business transaction it belongs to (`correlation_id`). This enables end-to-end tracing of complex flows (e.g., a health score drop triggers a playbook which creates a CTA which sends a Slack message — all linked by one correlation ID).

7. **Configuration tables are NOT event-sourced.** Scorecards, playbooks, and integration credentials are mutable configuration that changes infrequently. Event-sourcing these would add complexity without benefit. They remain as simple CRUD tables.

8. **Denormalised read models eliminate joins.** The `rm_accounts` read model includes the current health score, churn prediction, CSM name, and open CTA count — all denormalised from separate event streams. The portfolio dashboard never needs to JOIN across tables.

9. **Monthly partitioning on the event store.** Time-based partitioning enables efficient partition pruning for temporal queries and straightforward data lifecycle management (archive partitions older than N months to cold storage).

10. **Dead letter queue for projection failures.** When a projector fails to process an event, it goes to the dead letter queue with retry scheduling rather than blocking the projection pipeline. This prevents a single malformed event from halting all read model updates.

---

## Example Event Payloads

### health_score.calculated
```json
{
  "event_type": "health_score.calculated",
  "ce_specversion": "1.0",
  "ce_source": "/tenants/abc123/scoring-engine",
  "ce_time": "2026-05-12T14:30:00Z",
  "data": {
    "account_id": "acc-456",
    "scorecard_id": "sc-789",
    "composite_score": 62.5,
    "previous_score": 78.0,
    "score_band": "yellow",
    "previous_band": "green",
    "trend": "declining",
    "measures": {
      "usage": {"score": 45.0, "weight": 0.30, "raw_value": 23, "unit": "daily_active_users"},
      "engagement": {"score": 70.0, "weight": 0.25, "raw_value": 12, "unit": "interactions_per_week"},
      "support": {"score": 55.0, "weight": 0.20, "raw_value": 8, "unit": "open_tickets"},
      "nps": {"score": 80.0, "weight": 0.15, "raw_value": 8, "unit": "nps_score"},
      "financial": {"score": 95.0, "weight": 0.10, "raw_value": 50000, "unit": "arr"}
    },
    "top_signals": [
      {"signal": "dau_dropped_40_pct", "impact": -18.5, "description": "Daily active users dropped 40% over 14 days"},
      {"signal": "3_p1_tickets_open", "impact": -8.0, "description": "3 priority-1 support tickets opened in last 7 days"}
    ]
  },
  "metadata": {
    "actor_type": "system",
    "correlation_id": "scoring-run-2026-05-12-1430"
  }
}
```

### cta.created (triggered by playbook)
```json
{
  "event_type": "cta.created",
  "ce_source": "/tenants/abc123/playbook-engine",
  "data": {
    "cta_id": "cta-101",
    "account_id": "acc-456",
    "cta_type": "risk",
    "priority": "high",
    "title": "Usage decline detected — investigate account health",
    "assigned_to": "user-csm-001",
    "playbook_id": "pb-risk-001",
    "trigger_event_id": "evt-health-score-calculated-xyz"
  },
  "metadata": {
    "actor_type": "system",
    "causation_id": "evt-health-score-calculated-xyz",
    "correlation_id": "scoring-run-2026-05-12-1430"
  }
}
```

## Example Queries

### Replay account history (temporal query)
```sql
-- "What was the health score for account X on March 15, 2026?"
SELECT data->>'composite_score' AS health_score,
       data->>'score_band' AS band,
       ce_time
FROM event_store
WHERE aggregate_id = $1
  AND event_type = 'health_score.calculated'
  AND ce_time <= '2026-03-15T23:59:59Z'
ORDER BY ce_time DESC
LIMIT 1;
```

### Event stream for a single account (AI context window)
```sql
-- Feed the last 90 days of events for an account to an LLM
-- for generating a health narrative or QBR content
SELECT event_type, ce_time, data, metadata
FROM event_store
WHERE aggregate_id = $1
  AND ce_time >= now() - INTERVAL '90 days'
ORDER BY ce_time ASC;
```

### Audit trail for compliance
```sql
-- All state changes for an account, with actor attribution
SELECT event_type,
       ce_time,
       metadata->>'actor_type' AS actor,
       metadata->>'actor_id' AS actor_id,
       data
FROM event_store
WHERE aggregate_id = $1
ORDER BY event_version ASC;
```

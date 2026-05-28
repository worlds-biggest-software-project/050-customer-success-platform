# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Customer Success Platform · Created: 2026-05-12

## Philosophy

The hybrid relational + JSONB model uses strongly-typed relational columns for core, well-known fields that every CS platform deployment shares (account name, ARR, health score, lifecycle stage) while pushing variable, tenant-specific, and integration-dependent data into PostgreSQL JSONB columns. This approach recognises a fundamental reality of customer success platforms: the core domain model is stable (accounts, contacts, health scores, CTAs), but the details vary enormously across tenants — different CRM field mappings, different health score dimensions, different product usage event schemas, different compliance requirements by jurisdiction.

This is the architecture that Vitally and modern mid-market CS platforms use: a small, clean relational core with JSONB "traits" or "custom_fields" columns that extend every major entity without schema migrations. PostgreSQL's GIN indexes on JSONB columns enable efficient querying of custom fields, and the `@>` containment operator supports filtering on nested JSON structures. The trade-off versus full normalisation is that JSONB columns are less self-documenting and harder to constrain — but for a multi-tenant SaaS platform where every tenant has different custom fields, this flexibility is essential.

This model is designed for rapid MVP development. A team of 2-4 engineers can ship the initial schema in days rather than weeks, and adding new integration sources or custom fields requires zero DDL changes. As the product matures, frequently-queried JSONB fields can be promoted to typed columns without breaking existing data.

**Best for:** Startups and small teams building a multi-tenant CS platform MVP where speed-to-market and per-tenant flexibility matter more than strict normalisation — especially when tenants bring diverse CRM configurations and custom field requirements.

**Trade-offs:**
- (+) Far fewer tables than normalised model (~15 vs. ~25+) — simpler to reason about
- (+) Zero-migration extensibility: new fields are JSONB keys, not ALTER TABLE statements
- (+) Per-tenant customisation without custom field tables or EAV anti-patterns
- (+) Faster MVP development: schema changes are application-level, not DDL
- (+) Natural fit for diverse integration payloads (each CRM maps different fields)
- (-) JSONB columns lack foreign key constraints — referential integrity is application-enforced
- (-) Complex JSONB queries can be slower than typed column queries without careful GIN indexing
- (-) Schema documentation must be maintained externally (JSONB is not self-documenting)
- (-) Type safety is application-layer, not database-layer — risk of inconsistent data in JSONB
- (-) Harder to write BI/reporting queries against nested JSONB structures

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Segment Spec | `events.properties` JSONB column stores raw Segment Track/Identify payloads without schema transformation |
| CloudEvents v1.0 | Webhook payloads wrap event data in CloudEvents envelope; stored as JSONB in `outbound_webhooks.payload` |
| ISO 3166-1/2 | `accounts.country_code` is a typed column; jurisdiction-specific compliance fields go in `accounts.custom_fields` |
| OAuth 2.0 (RFC 6749) | `integrations.credentials` JSONB stores provider-specific OAuth token shapes without separate tables per provider |
| JSON Schema Draft 2020-12 | `custom_field_schemas` table stores JSON Schemas per entity type per tenant for runtime validation of JSONB custom fields |
| OData v4 (ISO/IEC 20802-1) | Reporting API exposes typed columns via OData; JSONB fields available via `$select` with JSON path expressions |
| Vitally REST API pattern | Entity design mirrors Vitally's API objects: accounts have `traits`, users have `traits`, tasks have `traits` — JSONB extension columns |
| NPS (Bain/Satmetrix) | NPS score and classification are typed columns; additional survey metadata in JSONB |

---

## Core Schema

```sql
-- ============================================================
-- TENANTS & USERS
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example: {
    --   "timezone": "America/New_York",
    --   "health_score_schedule": "realtime",
    --   "default_scorecard_id": "...",
    --   "branding": {"logo_url": "...", "primary_color": "#1a73e8"},
    --   "features": {"ai_narratives": true, "expansion_detection": true}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'csm',
    avatar_url      TEXT,
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example: {"timezone": "Europe/London", "notification_channels": ["slack", "email"],
    --                        "dashboard_layout": "compact", "digest_frequency": "daily"}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

-- ============================================================
-- ACCOUNTS & CONTACTS
-- Typed columns for universal fields; JSONB for tenant-specific
-- ============================================================

CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),
    name            VARCHAR(500) NOT NULL,
    domain          VARCHAR(255),
    
    -- Universal typed fields (every CS deployment uses these)
    industry        VARCHAR(100),
    tier            VARCHAR(50) DEFAULT 'standard',
    lifecycle_stage VARCHAR(50) DEFAULT 'onboarding',
    arr             NUMERIC(12,2),
    mrr             NUMERIC(12,2),
    contract_start  DATE,
    contract_end    DATE,
    renewal_date    DATE,
    country_code    CHAR(2),
    csm_id          UUID REFERENCES users(id),
    parent_id       UUID REFERENCES accounts(id),
    
    -- Denormalised health score (updated by scoring engine)
    health_score    NUMERIC(5,2),
    health_band     VARCHAR(10),
    health_trend    VARCHAR(10),
    health_narrative TEXT,
    health_scored_at TIMESTAMPTZ,
    
    -- Denormalised churn prediction
    churn_probability NUMERIC(5,4),
    churn_risk_band VARCHAR(20),
    
    -- CRM sync metadata
    crm_source      VARCHAR(50),
    crm_url         TEXT,
    crm_data        JSONB NOT NULL DEFAULT '{}',
    -- crm_data example: {
    --   "salesforce": {"Id": "001xxx", "Owner.Name": "Jane Doe", "Custom_Field__c": "value"},
    --   "hubspot": {"vid": 12345, "deal_stage": "Closed Won"}
    -- }
    
    -- Tenant-specific custom fields
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- custom_fields example: {
    --   "region": "EMEA",
    --   "csql_status": "qualified",
    --   "executive_sponsor": "John Smith",
    --   "contract_type": "annual",
    --   "deployment_type": "cloud",
    --   "compliance_tier": "soc2_required"
    -- }
    
    -- Computed feature flags for fast filtering
    tags            TEXT[] NOT NULL DEFAULT '{}',
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_accounts_tenant ON accounts(tenant_id);
CREATE INDEX idx_accounts_external ON accounts(tenant_id, external_id);
CREATE INDEX idx_accounts_csm ON accounts(csm_id);
CREATE INDEX idx_accounts_lifecycle ON accounts(tenant_id, lifecycle_stage);
CREATE INDEX idx_accounts_renewal ON accounts(tenant_id, renewal_date);
CREATE INDEX idx_accounts_health ON accounts(tenant_id, health_band);
CREATE INDEX idx_accounts_arr ON accounts(tenant_id, arr DESC);
CREATE INDEX idx_accounts_tags ON accounts USING GIN(tags);
CREATE INDEX idx_accounts_custom ON accounts USING GIN(custom_fields jsonb_path_ops);
CREATE INDEX idx_accounts_crm ON accounts USING GIN(crm_data jsonb_path_ops);

CREATE TABLE contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),
    email           VARCHAR(255),
    name            VARCHAR(255) NOT NULL,
    title           VARCHAR(255),
    role            VARCHAR(100),                         -- champion, decision_maker, end_user, executive_sponsor
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    last_activity_at TIMESTAMPTZ,
    
    -- Flexible contact traits (mirrors Vitally's traits pattern)
    traits          JSONB NOT NULL DEFAULT '{}',
    -- traits example: {
    --   "phone": "+1-555-0123",
    --   "linkedin_url": "https://linkedin.com/in/...",
    --   "preferred_channel": "slack",
    --   "timezone": "America/Chicago",
    --   "latest_nps_score": 9,
    --   "latest_nps_at": "2026-04-15T10:00:00Z"
    -- }
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_account ON contacts(account_id);
CREATE INDEX idx_contacts_tenant ON contacts(tenant_id);
CREATE INDEX idx_contacts_email ON contacts(tenant_id, email);
CREATE INDEX idx_contacts_traits ON contacts USING GIN(traits jsonb_path_ops);
```

## Health Scoring

```sql
-- Scorecard configuration (lightweight — measures are JSONB, not separate table)
CREATE TABLE scorecards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    
    -- Measures defined as JSONB array — no need for a separate measures table
    measures        JSONB NOT NULL DEFAULT '[]',
    -- measures example: [
    --   {"name": "usage", "weight": 0.30, "method": "ml_model", "thresholds": {"green": 80, "yellow": 50, "red": 0}},
    --   {"name": "engagement", "weight": 0.25, "method": "rule", "thresholds": {"green": 75, "yellow": 40, "red": 0}},
    --   {"name": "support", "weight": 0.20, "method": "rule", "thresholds": {"green": 85, "yellow": 60, "red": 0}},
    --   {"name": "nps", "weight": 0.15, "method": "rule", "thresholds": {"green": 70, "yellow": 40, "red": 0}},
    --   {"name": "financial", "weight": 0.10, "method": "rule", "thresholds": {"green": 90, "yellow": 70, "red": 0}}
    -- ]
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

-- Health score history: one row per scoring event per account
CREATE TABLE health_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    scorecard_id    UUID NOT NULL REFERENCES scorecards(id),
    
    -- Typed core fields
    composite_score NUMERIC(5,2) NOT NULL,
    score_band      VARCHAR(10) NOT NULL,
    trend           VARCHAR(10) NOT NULL DEFAULT 'stable',
    
    -- All measure scores in one JSONB column (avoids a separate measure_scores table)
    measure_scores  JSONB NOT NULL DEFAULT '{}',
    -- measure_scores example: {
    --   "usage": {"score": 45.0, "band": "red", "raw_value": 23, "signals": [
    --     {"signal": "dau_drop", "impact": -18.5, "description": "DAU dropped 40% in 14 days"}
    --   ]},
    --   "engagement": {"score": 70.0, "band": "yellow", "raw_value": 12},
    --   "support": {"score": 55.0, "band": "yellow", "raw_value": 8},
    --   "nps": {"score": 80.0, "band": "green", "raw_value": 8},
    --   "financial": {"score": 95.0, "band": "green", "raw_value": 50000}
    -- }
    
    -- AI-generated narrative
    ai_narrative    TEXT,
    
    -- Top signals driving this score (denormalised for fast dashboard rendering)
    top_signals     JSONB NOT NULL DEFAULT '[]',
    -- top_signals example: [
    --   {"signal": "dau_dropped_40_pct", "measure": "usage", "impact": -18.5},
    --   {"signal": "3_p1_tickets", "measure": "support", "impact": -8.0}
    -- ]
    
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (scored_at);

CREATE TABLE health_scores_2026_q1 PARTITION OF health_scores FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE health_scores_2026_q2 PARTITION OF health_scores FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE health_scores_2026_q3 PARTITION OF health_scores FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE health_scores_2026_q4 PARTITION OF health_scores FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_hs_account ON health_scores(account_id, scored_at DESC);
CREATE INDEX idx_hs_tenant ON health_scores(tenant_id, scored_at DESC);
CREATE INDEX idx_hs_band ON health_scores(tenant_id, score_band);
CREATE INDEX idx_hs_signals ON health_scores USING GIN(top_signals jsonb_path_ops);
```

## Events (Product Usage)

```sql
-- High-volume event ingestion table
-- Raw event payloads stored as JSONB — no schema transformation on ingest
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    account_id      UUID NOT NULL,
    contact_id      UUID,
    
    -- Segment Spec standard fields (typed for fast filtering)
    event_type      VARCHAR(50) NOT NULL,                 -- track, identify, page, screen, group
    event_name      VARCHAR(255) NOT NULL,                -- e.g. "feature_used", "login"
    source          VARCHAR(100) NOT NULL,                -- segment, amplitude, mixpanel, pendo, custom
    
    -- Full event payload as JSONB (preserves source schema without transformation)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Varies by source and event — stored as-is from Segment/Amplitude
    
    context         JSONB NOT NULL DEFAULT '{}',
    -- Segment context: {"page": {"path": "..."}, "userAgent": "...", "ip": "..."}
    
    source_event_id VARCHAR(255),                         -- deduplication key
    occurred_at     TIMESTAMPTZ NOT NULL,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

-- Monthly partitions
CREATE TABLE events_2026_05 PARTITION OF events FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE events_2026_06 PARTITION OF events FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_events_account ON events(account_id, occurred_at DESC);
CREATE INDEX idx_events_tenant ON events(tenant_id, occurred_at DESC);
CREATE INDEX idx_events_name ON events(tenant_id, event_name, occurred_at DESC);
CREATE INDEX idx_events_dedup ON events(tenant_id, source, source_event_id);
CREATE INDEX idx_events_props ON events USING GIN(properties jsonb_path_ops);
```

## Playbooks & CTAs

```sql
-- Playbook templates with steps as JSONB (not a separate table)
CREATE TABLE playbooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(50) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    
    -- Trigger conditions as JSONB (flexible per playbook type)
    trigger_config  JSONB NOT NULL DEFAULT '{}',
    -- trigger_config examples:
    -- Health threshold: {"type": "health_threshold", "measure": "usage", "condition": "drops_below", "threshold": 40}
    -- Lifecycle: {"type": "lifecycle_change", "from": "onboarding", "to": "adopting"}
    -- Scheduled: {"type": "scheduled", "days_before_renewal": 90}
    
    -- Steps as ordered JSONB array (eliminates playbook_steps table)
    steps           JSONB NOT NULL DEFAULT '[]',
    -- steps example: [
    --   {"order": 1, "action": "create_cta", "config": {"type": "risk", "priority": "high", "title_template": "Usage decline — {{account.name}}"}},
    --   {"order": 2, "action": "send_email", "config": {"template_id": "risk-alert-csm", "to": "csm", "delay_minutes": 0}},
    --   {"order": 3, "action": "wait", "config": {"duration_days": 3}},
    --   {"order": 4, "action": "send_slack", "config": {"channel": "#cs-alerts", "template": "risk-escalation"}},
    --   {"order": 5, "action": "ai_decision", "config": {"prompt": "Should we escalate to executive outreach?", "actions": {"yes": "escalate", "no": "close"}}}
    -- ]
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_playbooks_tenant ON playbooks(tenant_id);
CREATE INDEX idx_playbooks_category ON playbooks(tenant_id, category);

-- CTAs: typed columns for universal fields, JSONB for context
CREATE TABLE ctas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    assigned_to     UUID REFERENCES users(id),
    playbook_id     UUID REFERENCES playbooks(id),
    
    -- Typed universal fields
    cta_type        VARCHAR(50) NOT NULL,
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
    title           VARCHAR(500) NOT NULL,
    due_date        DATE,
    closed_at       TIMESTAMPTZ,
    
    -- Flexible context and task tracking
    context         JSONB NOT NULL DEFAULT '{}',
    -- context example: {
    --   "trigger_event": "health_score.band_changed",
    --   "trigger_details": {"from_band": "green", "to_band": "yellow", "score_drop": 18.5},
    --   "description": "Usage declined significantly — 40% DAU drop over 14 days",
    --   "suggested_actions": ["Schedule executive call", "Review product adoption data", "Check for support escalations"]
    -- }
    
    -- Tasks embedded as JSONB array (eliminates cta_tasks table)
    tasks           JSONB NOT NULL DEFAULT '[]',
    -- tasks example: [
    --   {"id": "t1", "title": "Review usage data", "status": "completed", "completed_at": "2026-05-12T10:00:00Z"},
    --   {"id": "t2", "title": "Call executive sponsor", "status": "in_progress", "assigned_to": "user-csm-001"},
    --   {"id": "t3", "title": "Send QBR deck", "status": "pending", "due_date": "2026-05-20"}
    -- ]
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ctas_account ON ctas(account_id);
CREATE INDEX idx_ctas_assigned ON ctas(assigned_to, status);
CREATE INDEX idx_ctas_tenant_status ON ctas(tenant_id, status);
CREATE INDEX idx_ctas_due ON ctas(tenant_id, due_date) WHERE status IN ('open', 'in_progress');
CREATE INDEX idx_ctas_context ON ctas USING GIN(context jsonb_path_ops);
```

## Timeline, Surveys & Integrations

```sql
-- Timeline: activity feed with flexible payload
CREATE TABLE timeline_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    author_id       UUID REFERENCES users(id),
    entry_type      VARCHAR(50) NOT NULL,                 -- call, email, meeting, note, milestone, system
    subject         VARCHAR(500),
    body            TEXT,
    sentiment       VARCHAR(20),
    
    -- Flexible metadata per entry type
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- For calls: {"duration_minutes": 30, "attendees": ["john@acme.com"], "recording_url": "..."}
    -- For emails: {"from": "csm@us.com", "to": ["john@acme.com"], "thread_id": "..."}
    -- For milestones: {"milestone_name": "Go-live", "milestone_date": "2026-06-01"}
    
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_timeline_account ON timeline_entries(account_id, occurred_at DESC);
CREATE INDEX idx_timeline_type ON timeline_entries(account_id, entry_type);
CREATE INDEX idx_timeline_metadata ON timeline_entries USING GIN(metadata jsonb_path_ops);

-- Survey responses: typed score columns, JSONB for extra survey data
CREATE TABLE survey_responses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    contact_id      UUID REFERENCES contacts(id),
    survey_type     VARCHAR(20) NOT NULL,                 -- nps, csat, ces, custom
    
    -- Typed score fields
    score           SMALLINT NOT NULL,                    -- NPS: 0-10, CSAT: 1-5, CES: 1-7
    classification  VARCHAR(20),                          -- promoter, passive, detractor (NPS only)
    feedback        TEXT,
    
    -- Flexible survey metadata
    survey_data     JSONB NOT NULL DEFAULT '{}',
    -- survey_data example: {
    --   "campaign_id": "nps-q2-2026",
    --   "channel": "email",
    --   "follow_up_questions": [
    --     {"question": "What could we improve?", "answer": "Faster support response times"}
    --   ]
    -- }
    
    responded_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_surveys_account ON survey_responses(account_id, responded_at DESC);
CREATE INDEX idx_surveys_type ON survey_responses(tenant_id, survey_type);
CREATE INDEX idx_surveys_score ON survey_responses(tenant_id, classification);

-- Integrations: single table for all providers, JSONB for provider-specific config
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,                 -- salesforce, hubspot, segment, amplitude, zendesk, slack, pendo
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    
    -- Provider-specific credentials and config as JSONB
    -- (eliminates separate credentials table — each provider has different token shapes)
    credentials     JSONB NOT NULL DEFAULT '{}',
    -- Salesforce: {"access_token": "...", "refresh_token": "...", "instance_url": "https://na1.salesforce.com", "expires_at": "..."}
    -- Segment: {"write_key": "...", "workspace_id": "..."}
    -- Slack: {"bot_token": "xoxb-...", "channel_id": "C1234567890"}
    
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example: {
    --   "sync_direction": "bidirectional",
    --   "field_mapping": {"Account.Name": "name", "Account.ARR__c": "arr", "Account.Industry": "industry"},
    --   "sync_frequency_minutes": 15,
    --   "filters": {"Account.Type": "Customer"}
    -- }
    
    last_sync_at    TIMESTAMPTZ,
    sync_error      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, provider)
);

CREATE INDEX idx_integrations_tenant ON integrations(tenant_id);
```

## AI/ML & Custom Field Schema Registry

```sql
-- ML model tracking
CREATE TABLE ml_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    model_type      VARCHAR(50) NOT NULL,
    model_version   VARCHAR(50) NOT NULL,
    artifact_path   TEXT NOT NULL,
    
    -- Model metrics and config as JSONB (varies by model type)
    metrics         JSONB NOT NULL DEFAULT '{}',
    -- {"precision": 0.94, "recall": 0.87, "f1": 0.90, "auc_roc": 0.95, "training_samples": 50000}
    
    config          JSONB NOT NULL DEFAULT '{}',
    -- {"features": ["login_frequency", "support_tickets_7d", "nps_trend", ...], "hyperparams": {...}}
    
    is_active       BOOLEAN NOT NULL DEFAULT false,
    trained_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ml_models_tenant ON ml_models(tenant_id, model_type);

-- Custom field schema registry (validates JSONB custom_fields at application layer)
CREATE TABLE custom_field_schemas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    entity_type     VARCHAR(50) NOT NULL,                 -- account, contact, cta
    field_name      VARCHAR(100) NOT NULL,
    field_type      VARCHAR(50) NOT NULL,                 -- string, number, boolean, date, enum, url
    display_name    VARCHAR(255) NOT NULL,
    description     TEXT,
    is_required     BOOLEAN NOT NULL DEFAULT false,
    enum_values     TEXT[],                               -- for enum type fields
    display_order   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, entity_type, field_name)
);

CREATE INDEX idx_custom_schemas_tenant ON custom_field_schemas(tenant_id, entity_type);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 2 | tenants, users |
| Account Management | 2 | accounts (with custom_fields JSONB), contacts (with traits JSONB) |
| Health Scoring | 2 | scorecards (measures as JSONB), health_scores (partitioned, measure_scores as JSONB) |
| Product Usage | 1 | events (partitioned, properties as JSONB) |
| Playbooks & CTAs | 2 | playbooks (steps as JSONB), ctas (tasks as JSONB) |
| Timeline | 1 | timeline_entries (metadata as JSONB) |
| Surveys | 1 | survey_responses (survey_data as JSONB) |
| Integrations | 1 | integrations (credentials and config as JSONB) |
| AI/ML | 1 | ml_models (metrics and config as JSONB) |
| Schema Registry | 1 | custom_field_schemas |
| **Total** | **14** | ~40% fewer tables than normalised model |

---

## Key Design Decisions

1. **JSONB "traits" pattern on every entity.** Accounts have `custom_fields`, contacts have `traits`, CTAs have `context`, timeline entries have `metadata`. This mirrors how Vitally and modern SaaS platforms handle per-tenant extensibility without EAV tables or custom field junction tables.

2. **Playbook steps and CTA tasks are embedded JSONB arrays.** A playbook is a single document with an ordered `steps` array. A CTA is a single document with an embedded `tasks` array. This eliminates two tables and simplifies CRUD: creating a CTA with 5 tasks is a single INSERT, not 6 INSERTs across 2 tables.

3. **GIN indexes on all JSONB columns.** The `jsonb_path_ops` GIN index class supports the `@>` containment operator, enabling efficient queries like `WHERE custom_fields @> '{"region": "EMEA"}'`. This provides near-typed-column performance for frequently filtered custom fields.

4. **Scorecard measures as JSONB, not a separate table.** Since measures are always read and written as a unit (you never query a single measure in isolation from its scorecard), storing them as a JSONB array on the scorecard row eliminates a JOIN and simplifies the scoring engine code.

5. **Integration credentials as JSONB.** Each provider (Salesforce, HubSpot, Segment, Slack) has a different credential shape. Rather than a polymorphic credentials table or separate tables per provider, a single JSONB column stores whatever the provider needs. The application layer validates the shape based on the `provider` column.

6. **Health score `measure_scores` JSONB includes signals.** Each measure in the JSONB object includes not just the score but the raw signals that drove it. This gives the AI narrative generator everything it needs in a single row read — no joins to a signals table.

7. **Custom field schema registry for runtime validation.** The `custom_field_schemas` table stores JSON Schema-like definitions per entity per tenant. The application validates incoming JSONB custom fields against these definitions. This provides tenant-level type safety without database-level constraints.

8. **Quarterly partitioning for health scores.** Health scores are written frequently but queried primarily over recent periods. Quarterly partitions (vs. monthly for events) balance partition count against retention flexibility.

9. **CRM data preserved as-is in `crm_data` JSONB.** Rather than mapping every CRM field to a typed column, the raw CRM record is stored alongside the mapped typed columns. This preserves original data for debugging, re-mapping, and fields that are not yet mapped to typed columns.

10. **Tags as TEXT array for fast account filtering.** The `accounts.tags` column uses PostgreSQL's native TEXT array with a GIN index, enabling efficient filtering like `WHERE tags @> ARRAY['at-risk', 'enterprise']` without JSONB overhead.

---

## Example Queries

### Filter accounts by custom field
```sql
-- Find all EMEA enterprise accounts with SOC 2 compliance requirement
SELECT id, name, arr, health_score, health_band
FROM accounts
WHERE tenant_id = $1
  AND custom_fields @> '{"region": "EMEA", "compliance_tier": "soc2_required"}'
  AND tier = 'enterprise'
ORDER BY arr DESC;
```

### Health score with measure breakdown (single query, no joins)
```sql
SELECT
    a.name,
    a.arr,
    hs.composite_score,
    hs.score_band,
    hs.measure_scores->'usage'->>'score' AS usage_score,
    hs.measure_scores->'engagement'->>'score' AS engagement_score,
    hs.measure_scores->'support'->>'score' AS support_score,
    hs.ai_narrative,
    hs.top_signals
FROM accounts a
JOIN LATERAL (
    SELECT * FROM health_scores
    WHERE account_id = a.id
    ORDER BY scored_at DESC LIMIT 1
) hs ON true
WHERE a.tenant_id = $1
  AND a.health_band = 'red'
ORDER BY a.arr DESC;
```

### JSONB containment query on event properties
```sql
-- Find all accounts where users triggered the "export_report" event
-- with format = "pdf" in the last 30 days
SELECT DISTINCT e.account_id, a.name
FROM events e
JOIN accounts a ON a.id = e.account_id
WHERE e.tenant_id = $1
  AND e.event_name = 'export_report'
  AND e.properties @> '{"format": "pdf"}'
  AND e.occurred_at >= now() - INTERVAL '30 days';
```

### Promote a JSONB field to a typed column (field graduation)
```sql
-- When a custom field is queried so often it deserves a typed column,
-- add the column and backfill from JSONB:
ALTER TABLE accounts ADD COLUMN region VARCHAR(50);
UPDATE accounts SET region = custom_fields->>'region' WHERE custom_fields ? 'region';
CREATE INDEX idx_accounts_region ON accounts(tenant_id, region);
-- Application code starts writing to both the typed column and JSONB
-- Eventually remove the JSONB field from custom_fields
```

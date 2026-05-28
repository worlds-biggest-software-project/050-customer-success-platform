# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Customer Success Platform · Created: 2026-05-12

## Philosophy

The graph-relational hybrid model uses a traditional PostgreSQL relational core for operational CRUD (accounts, health scores, CTAs, integrations) combined with a property graph layer for relationship-heavy queries that are difficult or slow in pure relational systems. The graph layer — implemented either as PostgreSQL tables (`graph_nodes`/`graph_edges`) with recursive CTEs, or as a dedicated graph database like Neo4j alongside PostgreSQL — models the rich web of relationships that drive customer success: stakeholder influence networks, account hierarchies, product dependency graphs, and cross-account churn contagion patterns.

Customer success is fundamentally a relationship domain. An account's health depends not just on its own usage metrics but on the relationships between people: is the executive sponsor still engaged? Has the champion left? Are multiple contacts at the same account showing disengagement simultaneously? Are accounts owned by the same parent company churning in sequence (contagion)? These are graph traversal problems. A normalized relational model can answer them with recursive CTEs and multi-way JOINs, but the queries become complex and slow as the relationship graph grows. A graph model makes these queries natural and performant.

This architecture is inspired by how Neo4j is used in financial services for ownership chain analysis and in CRM for 360-degree customer views. For a CS platform, the graph layer adds unique analytical capabilities: stakeholder influence mapping (who is the real decision-maker?), churn contagion detection (is parent company strategic direction changing?), and expansion signal detection (are contacts from one account appearing in conversations about a second product?).

**Best for:** CS platforms serving enterprise accounts with complex organisational hierarchies, multi-stakeholder relationships, and parent-subsidiary structures — where understanding relationship dynamics is as important as tracking usage metrics.

**Trade-offs:**
- (+) Stakeholder relationship queries (influence mapping, champion tracking) are natural graph traversals
- (+) Account hierarchy traversal is O(depth) in graph vs. O(n) recursive CTE in relational
- (+) Churn contagion analysis across related accounts becomes a simple multi-hop query
- (+) Expansion signal detection via cross-account relationship patterns
- (+) Graph visualisation of stakeholder maps provides CSMs with intuitive account intelligence
- (-) Additional infrastructure complexity (graph database or complex recursive CTEs)
- (-) Data consistency between relational store and graph store requires synchronisation
- (-) Fewer developers have graph database experience compared to SQL
- (-) Graph queries are powerful but harder to optimise and debug than SQL
- (-) If the CS platform targets SMB accounts with simple relationships, the graph layer is over-engineered

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 3166-1/2 | Account jurisdiction modeled as graph node properties; geographic hierarchy as graph edges |
| Segment Spec | Product usage events ingested into relational event store; graph layer derives relationship edges from event patterns |
| CloudEvents v1.0 | Webhook payloads and inter-service events follow CloudEvents envelope |
| OAuth 2.0 (RFC 6749) | Integration credentials follow standard OAuth token structure |
| OpenAPI 3.1 | REST API for relational CRUD; separate graph query API for relationship traversal |
| NPS (Bain/Satmetrix) | NPS scores stored relationally; NPS trends surface as graph edge weights (contact sentiment → account health) |
| ISO/IEC 25012 (Data Quality) | Graph edge confidence scores reflect data quality of the underlying signal |
| Property Graph Model (ISO/IEC 39075 GQL) | Graph schema aligns with the emerging GQL standard for property graphs |

---

## Relational Core (PostgreSQL)

### Tenant, User, Account, Contact

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'csm',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),
    name            VARCHAR(500) NOT NULL,
    domain          VARCHAR(255),
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
    -- Note: parent_id is modeled as a graph edge, not a self-referential FK
    -- This allows more complex hierarchies (multi-parent, temporal relationships)
    health_score    NUMERIC(5,2),
    health_band     VARCHAR(10),
    health_trend    VARCHAR(10),
    health_narrative TEXT,
    health_scored_at TIMESTAMPTZ,
    churn_probability NUMERIC(5,4),
    churn_risk_band VARCHAR(20),
    crm_source      VARCHAR(50),
    crm_url         TEXT,
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_accounts_tenant ON accounts(tenant_id);
CREATE INDEX idx_accounts_external ON accounts(tenant_id, external_id);
CREATE INDEX idx_accounts_csm ON accounts(csm_id);
CREATE INDEX idx_accounts_health ON accounts(tenant_id, health_band);
CREATE INDEX idx_accounts_renewal ON accounts(tenant_id, renewal_date);

CREATE TABLE contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),
    email           VARCHAR(255),
    name            VARCHAR(255) NOT NULL,
    title           VARCHAR(255),
    role            VARCHAR(100),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    last_activity_at TIMESTAMPTZ,
    traits          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_account ON contacts(account_id);
CREATE INDEX idx_contacts_tenant ON contacts(tenant_id);
CREATE INDEX idx_contacts_email ON contacts(tenant_id, email);
```

### Health Scores, Events, CTAs, Integrations

```sql
CREATE TABLE scorecards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    measures        JSONB NOT NULL DEFAULT '[]',
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE health_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    scorecard_id    UUID NOT NULL REFERENCES scorecards(id),
    composite_score NUMERIC(5,2) NOT NULL,
    score_band      VARCHAR(10) NOT NULL,
    trend           VARCHAR(10) NOT NULL DEFAULT 'stable',
    measure_scores  JSONB NOT NULL DEFAULT '{}',
    ai_narrative    TEXT,
    top_signals     JSONB NOT NULL DEFAULT '[]',
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (scored_at);

CREATE TABLE health_scores_2026_q1 PARTITION OF health_scores FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE health_scores_2026_q2 PARTITION OF health_scores FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');

CREATE INDEX idx_hs_account ON health_scores(account_id, scored_at DESC);
CREATE INDEX idx_hs_tenant ON health_scores(tenant_id, scored_at DESC);

CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    account_id      UUID NOT NULL,
    contact_id      UUID,
    event_type      VARCHAR(50) NOT NULL,
    event_name      VARCHAR(255) NOT NULL,
    source          VARCHAR(100) NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    occurred_at     TIMESTAMPTZ NOT NULL,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2026_05 PARTITION OF events FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_events_account ON events(account_id, occurred_at DESC);
CREATE INDEX idx_events_tenant ON events(tenant_id, occurred_at DESC);

CREATE TABLE playbooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(50) NOT NULL,
    trigger_config  JSONB NOT NULL DEFAULT '{}',
    steps           JSONB NOT NULL DEFAULT '[]',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ctas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    assigned_to     UUID REFERENCES users(id),
    playbook_id     UUID REFERENCES playbooks(id),
    cta_type        VARCHAR(50) NOT NULL,
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
    title           VARCHAR(500) NOT NULL,
    due_date        DATE,
    context         JSONB NOT NULL DEFAULT '{}',
    tasks           JSONB NOT NULL DEFAULT '[]',
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ctas_account ON ctas(account_id);
CREATE INDEX idx_ctas_assigned ON ctas(assigned_to, status);

CREATE TABLE survey_responses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    contact_id      UUID REFERENCES contacts(id),
    survey_type     VARCHAR(20) NOT NULL,
    score           SMALLINT NOT NULL,
    classification  VARCHAR(20),
    feedback        TEXT,
    responded_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_surveys_account ON survey_responses(account_id, responded_at DESC);

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    credentials     JSONB NOT NULL DEFAULT '{}',
    config          JSONB NOT NULL DEFAULT '{}',
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, provider)
);
```

---

## Graph Layer (PostgreSQL Implementation)

The graph layer uses two tables — `graph_nodes` and `graph_edges` — to model the relationship network. This approach keeps the entire system in PostgreSQL without requiring a separate graph database, while supporting graph traversal via recursive CTEs and `ltree` paths.

```sql
-- ============================================================
-- GRAPH NODES
-- Every entity that participates in relationships gets a graph node
-- Nodes reference their relational table counterpart via entity_type + entity_id
-- ============================================================
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    entity_type     VARCHAR(50) NOT NULL,                 -- account, contact, user, product, feature, team
    entity_id       UUID NOT NULL,                        -- FK to the relational table (not enforced to allow flexibility)
    label           VARCHAR(255) NOT NULL,                -- human-readable label for visualisation
    node_type       VARCHAR(50) NOT NULL,                 -- more specific than entity_type (e.g. "enterprise_account", "champion_contact")
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Cached properties for graph queries without joining back to relational tables
    -- account node: {"name": "Acme Corp", "arr": 120000, "health_band": "yellow", "industry": "Technology"}
    -- contact node: {"name": "Jane Smith", "title": "VP Engineering", "role": "champion", "nps": 9}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, entity_type, entity_id)
);

CREATE INDEX idx_gnodes_tenant ON graph_nodes(tenant_id);
CREATE INDEX idx_gnodes_type ON graph_nodes(tenant_id, entity_type);
CREATE INDEX idx_gnodes_entity ON graph_nodes(entity_type, entity_id);
CREATE INDEX idx_gnodes_props ON graph_nodes USING GIN(properties jsonb_path_ops);

-- ============================================================
-- GRAPH EDGES
-- Directed relationships between nodes with typed labels,
-- temporal validity, and confidence scoring
-- ============================================================
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(100) NOT NULL,
    -- Edge types catalogue:
    --   ACCOUNT HIERARCHY:
    --     parent_of         - account → child account
    --     subsidiary_of     - account → parent account
    --   PEOPLE:
    --     works_at          - contact → account
    --     manages           - contact → contact (reporting structure)
    --     influences        - contact → contact (influence network)
    --     champions         - contact → account (champion relationship)
    --     sponsors          - contact → account (executive sponsor)
    --   CS ASSIGNMENT:
    --     manages_account   - user (CSM) → account
    --     escalated_to      - user → user (escalation chain)
    --   PRODUCT:
    --     uses_feature      - contact → feature node
    --     depends_on        - feature → feature (product dependency)
    --   CROSS-ACCOUNT:
    --     same_company_as   - account → account (shared parent)
    --     referred_by       - account → account (referral relationship)
    --     shares_contact    - account → account (contact works at both)
    
    -- Edge properties
    properties      JSONB NOT NULL DEFAULT '{}',
    -- For influence edges: {"influence_score": 0.85, "influence_type": "technical"}
    -- For uses_feature: {"adoption_date": "2026-01-15", "usage_frequency": "daily", "depth": "power_user"}
    -- For parent_of: {"ownership_pct": 100, "relationship_type": "wholly_owned"}
    
    -- Edge weight for graph algorithms (PageRank, shortest path, community detection)
    weight          NUMERIC(5,4) NOT NULL DEFAULT 1.0,
    
    -- Temporal validity: when was this relationship active?
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,                          -- NULL = currently active
    
    -- Confidence: how certain are we this relationship exists?
    -- 1.0 = manually verified, 0.5 = inferred from data patterns
    confidence      NUMERIC(3,2) NOT NULL DEFAULT 1.0,
    source          VARCHAR(50) NOT NULL DEFAULT 'manual', -- manual, crm_sync, inferred, ai
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gedges_source ON graph_edges(source_node_id);
CREATE INDEX idx_gedges_target ON graph_edges(target_node_id);
CREATE INDEX idx_gedges_type ON graph_edges(tenant_id, edge_type);
CREATE INDEX idx_gedges_tenant ON graph_edges(tenant_id);
CREATE INDEX idx_gedges_active ON graph_edges(tenant_id, edge_type) WHERE valid_to IS NULL;
CREATE INDEX idx_gedges_temporal ON graph_edges(source_node_id, valid_from, valid_to);
CREATE INDEX idx_gedges_props ON graph_edges USING GIN(properties jsonb_path_ops);

-- ============================================================
-- GRAPH TRAVERSAL MATERIALISED VIEWS
-- Pre-computed relationship summaries for common query patterns
-- ============================================================

-- Account hierarchy paths (materialised for fast traversal)
CREATE MATERIALIZED VIEW mv_account_hierarchy AS
WITH RECURSIVE hierarchy AS (
    -- Root accounts (no parent edge)
    SELECT
        n.entity_id AS account_id,
        n.tenant_id,
        n.entity_id AS root_account_id,
        n.label AS account_name,
        0 AS depth,
        ARRAY[n.entity_id] AS path
    FROM graph_nodes n
    WHERE n.entity_type = 'account'
      AND n.is_active = true
      AND NOT EXISTS (
          SELECT 1 FROM graph_edges e
          JOIN graph_nodes parent ON parent.id = e.target_node_id
          WHERE e.source_node_id = n.id
            AND e.edge_type = 'subsidiary_of'
            AND e.valid_to IS NULL
      )
    UNION ALL
    -- Child accounts
    SELECT
        child_node.entity_id AS account_id,
        child_node.tenant_id,
        h.root_account_id,
        child_node.label AS account_name,
        h.depth + 1,
        h.path || child_node.entity_id
    FROM hierarchy h
    JOIN graph_nodes parent_node ON parent_node.entity_type = 'account' AND parent_node.entity_id = h.account_id
    JOIN graph_edges e ON e.target_node_id = parent_node.id AND e.edge_type = 'subsidiary_of' AND e.valid_to IS NULL
    JOIN graph_nodes child_node ON child_node.id = e.source_node_id AND child_node.entity_type = 'account'
    WHERE child_node.entity_id != ALL(h.path)  -- prevent cycles
)
SELECT * FROM hierarchy;

CREATE UNIQUE INDEX idx_mv_hierarchy_account ON mv_account_hierarchy(account_id);
CREATE INDEX idx_mv_hierarchy_root ON mv_account_hierarchy(root_account_id);
CREATE INDEX idx_mv_hierarchy_tenant ON mv_account_hierarchy(tenant_id);

-- Stakeholder influence summary per account
CREATE MATERIALIZED VIEW mv_stakeholder_influence AS
SELECT
    a_node.entity_id AS account_id,
    a_node.tenant_id,
    c_node.entity_id AS contact_id,
    c_node.properties->>'name' AS contact_name,
    c_node.properties->>'title' AS contact_title,
    c_node.properties->>'role' AS contact_role,
    e.edge_type AS relationship_type,
    e.weight AS influence_weight,
    e.confidence,
    e.properties AS relationship_properties,
    -- Count of other contacts this person influences
    (SELECT COUNT(*) FROM graph_edges e2
     WHERE e2.source_node_id = c_node.id
       AND e2.edge_type = 'influences'
       AND e2.valid_to IS NULL) AS influence_reach
FROM graph_nodes a_node
JOIN graph_edges e ON (e.target_node_id = a_node.id OR e.source_node_id = a_node.id)
JOIN graph_nodes c_node ON (
    (c_node.id = e.source_node_id AND e.target_node_id = a_node.id) OR
    (c_node.id = e.target_node_id AND e.source_node_id = a_node.id)
)
WHERE a_node.entity_type = 'account'
  AND c_node.entity_type = 'contact'
  AND e.edge_type IN ('works_at', 'champions', 'sponsors', 'influences')
  AND e.valid_to IS NULL;

CREATE INDEX idx_mv_stakeholder_account ON mv_stakeholder_influence(account_id);
CREATE INDEX idx_mv_stakeholder_tenant ON mv_stakeholder_influence(tenant_id);
```

## Relationship Inference Engine

```sql
-- Stores AI-inferred relationships that need human review
CREATE TABLE inferred_relationships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    source_entity_type VARCHAR(50) NOT NULL,
    source_entity_id   UUID NOT NULL,
    target_entity_type VARCHAR(50) NOT NULL,
    target_entity_id   UUID NOT NULL,
    proposed_edge_type VARCHAR(100) NOT NULL,
    confidence      NUMERIC(3,2) NOT NULL,
    evidence        JSONB NOT NULL DEFAULT '[]',
    -- evidence example: [
    --   {"type": "email_domain_match", "detail": "john@acme.com appears in both accounts", "weight": 0.4},
    --   {"type": "crm_parent_field", "detail": "Salesforce ParentId links these accounts", "weight": 0.5},
    --   {"type": "shared_ip_range", "detail": "Product logins from same IP range", "weight": 0.1}
    -- ]
    status          VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, accepted, rejected
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inferred_tenant ON inferred_relationships(tenant_id, status);
CREATE INDEX idx_inferred_source ON inferred_relationships(source_entity_type, source_entity_id);
```

## Churn Contagion Analysis

```sql
-- Pre-computed churn contagion risk scores
-- Updated when any account in a hierarchy changes health band
CREATE TABLE churn_contagion_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    account_id      UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    
    -- Direct risk (from this account's own health score)
    direct_risk     NUMERIC(5,4) NOT NULL,
    
    -- Contagion risk (from related accounts)
    contagion_risk  NUMERIC(5,4) NOT NULL,
    
    -- Combined risk
    combined_risk   NUMERIC(5,4) NOT NULL,
    
    -- Which related accounts are contributing to contagion risk
    contagion_sources JSONB NOT NULL DEFAULT '[]',
    -- contagion_sources example: [
    --   {"account_id": "...", "account_name": "Acme Corp Parent", "relationship": "parent_of",
    --    "their_health_band": "red", "contagion_weight": 0.35},
    --   {"account_id": "...", "account_name": "Acme Corp UK", "relationship": "same_company_as",
    --    "their_health_band": "yellow", "contagion_weight": 0.15}
    -- ]
    
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contagion_account ON churn_contagion_scores(account_id, computed_at DESC);
CREATE INDEX idx_contagion_risk ON churn_contagion_scores(tenant_id, combined_risk DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 2 | tenants, users |
| Account Management | 2 | accounts, contacts |
| Health Scoring | 2 | scorecards, health_scores (partitioned) |
| Product Usage | 1 | events (partitioned) |
| Playbooks & CTAs | 2 | playbooks, ctas |
| Surveys | 1 | survey_responses |
| Integrations | 1 | integrations |
| Graph Layer | 2 | graph_nodes, graph_edges |
| Graph Materialised Views | 2 | mv_account_hierarchy, mv_stakeholder_influence |
| Graph Intelligence | 2 | inferred_relationships, churn_contagion_scores |
| **Total** | **17** | Plus 2 materialised views |

---

## Key Design Decisions

1. **Graph layer as an overlay, not a replacement.** The relational core handles all operational CRUD (creating accounts, updating health scores, managing CTAs). The graph layer is an analytical overlay that enables relationship queries. This means the system works perfectly well without the graph layer — it is additive, not foundational.

2. **PostgreSQL-native graph, not a separate database.** By implementing the graph as `graph_nodes` + `graph_edges` tables in PostgreSQL, we avoid the operational complexity of running a separate Neo4j instance, the data synchronisation problem between two databases, and the need for graph database expertise on the team. Recursive CTEs and materialised views provide adequate performance for CS platform graph queries (typically <1000 nodes per tenant).

3. **Temporal edges with `valid_from`/`valid_to`.** Relationships change over time. A champion might leave (valid_to set to their departure date). An executive sponsor might be replaced. Temporal edges enable "who was the champion when the account went red?" queries that are critical for post-churn analysis.

4. **Confidence scores on edges.** Relationships inferred from data patterns (e.g., "these two accounts share a domain, so they may be related") carry lower confidence than manually-verified relationships. The `confidence` column enables CSMs to see which relationships are confirmed vs. suggested.

5. **Separate inference review pipeline.** AI-inferred relationships are staged in `inferred_relationships` for human review before being promoted to `graph_edges`. This prevents false relationship signals from corrupting the stakeholder map.

6. **Materialised views for common traversals.** Account hierarchy and stakeholder influence maps are materialised and refreshed periodically (or on trigger). This avoids running recursive CTEs on every dashboard load. The materialised views are disposable — they can be rebuilt from the underlying graph tables at any time.

7. **Churn contagion as a first-class metric.** The `churn_contagion_scores` table pre-computes how much risk each account absorbs from its related accounts. This is a unique analytical capability that purely relational models cannot easily provide.

8. **Node properties cache relational data.** Graph node `properties` JSONB caches key fields from the relational tables (account name, ARR, health band, contact title). This enables graph traversal queries to return useful information without joining back to the relational core — critical for graph visualisation UIs.

9. **Edge types serve as the domain vocabulary.** The `edge_type` column on `graph_edges` establishes a controlled vocabulary of relationship types. Each type has documented semantics (see the edge types catalogue in the DDL comments). New relationship types can be added without schema changes.

10. **Account hierarchies modeled as graph edges, not self-referential FKs.** Instead of a `parent_id` column on `accounts`, parent-child relationships are `subsidiary_of` edges in the graph. This supports more complex hierarchies (multi-parent, joint ventures, temporal ownership changes) that a single self-referential FK cannot model.

---

## Example Queries

### Find all accounts in a hierarchy with their total ARR
```sql
-- Using the materialised hierarchy view
SELECT
    h.root_account_id,
    ra.name AS root_account_name,
    COUNT(*) AS accounts_in_hierarchy,
    SUM(a.arr) AS total_hierarchy_arr,
    SUM(CASE WHEN a.health_band = 'red' THEN 1 ELSE 0 END) AS red_accounts
FROM mv_account_hierarchy h
JOIN accounts a ON a.id = h.account_id
JOIN accounts ra ON ra.id = h.root_account_id
WHERE h.tenant_id = $1
GROUP BY h.root_account_id, ra.name
ORDER BY total_hierarchy_arr DESC;
```

### Stakeholder influence map for an account
```sql
-- All contacts related to this account, ordered by influence
SELECT
    contact_name,
    contact_title,
    contact_role,
    relationship_type,
    influence_weight,
    influence_reach,
    confidence
FROM mv_stakeholder_influence
WHERE account_id = $1
ORDER BY influence_weight DESC, influence_reach DESC;
```

### Churn contagion: find accounts at risk due to related account health
```sql
-- Accounts where contagion risk exceeds their own direct risk
SELECT
    a.name,
    a.arr,
    a.health_band,
    c.direct_risk,
    c.contagion_risk,
    c.combined_risk,
    c.contagion_sources
FROM churn_contagion_scores c
JOIN accounts a ON a.id = c.account_id
WHERE c.tenant_id = $1
  AND c.contagion_risk > c.direct_risk
  AND c.computed_at = (
      SELECT MAX(c2.computed_at)
      FROM churn_contagion_scores c2
      WHERE c2.account_id = c.account_id
  )
ORDER BY c.combined_risk DESC;
```

### Multi-hop relationship traversal (raw recursive CTE)
```sql
-- Find all entities within 3 hops of a given account
WITH RECURSIVE traversal AS (
    -- Start node
    SELECT
        n.id AS node_id,
        n.entity_type,
        n.entity_id,
        n.label,
        0 AS depth,
        ARRAY[n.id] AS path
    FROM graph_nodes n
    WHERE n.entity_type = 'account' AND n.entity_id = $1
    
    UNION ALL
    
    -- Traverse outbound edges
    SELECT
        target.id,
        target.entity_type,
        target.entity_id,
        target.label,
        t.depth + 1,
        t.path || target.id
    FROM traversal t
    JOIN graph_edges e ON e.source_node_id = t.node_id AND e.valid_to IS NULL
    JOIN graph_nodes target ON target.id = e.target_node_id
    WHERE t.depth < 3
      AND target.id != ALL(t.path)  -- prevent cycles
    
    UNION ALL
    
    -- Traverse inbound edges
    SELECT
        source.id,
        source.entity_type,
        source.entity_id,
        source.label,
        t.depth + 1,
        t.path || source.id
    FROM traversal t
    JOIN graph_edges e ON e.target_node_id = t.node_id AND e.valid_to IS NULL
    JOIN graph_nodes source ON source.id = e.source_node_id
    WHERE t.depth < 3
      AND source.id != ALL(t.path)
)
SELECT DISTINCT entity_type, entity_id, label, depth
FROM traversal
ORDER BY depth, entity_type, label;
```

### Detect shared contacts across accounts (expansion signal)
```sql
-- Find contacts who work at multiple accounts (cross-sell opportunity)
SELECT
    c_node.entity_id AS contact_id,
    c_node.properties->>'name' AS contact_name,
    c_node.properties->>'title' AS contact_title,
    array_agg(a_node.entity_id) AS account_ids,
    array_agg(a_node.label) AS account_names,
    COUNT(DISTINCT a_node.id) AS account_count
FROM graph_nodes c_node
JOIN graph_edges e ON e.source_node_id = c_node.id AND e.edge_type = 'works_at' AND e.valid_to IS NULL
JOIN graph_nodes a_node ON a_node.id = e.target_node_id AND a_node.entity_type = 'account'
WHERE c_node.tenant_id = $1
  AND c_node.entity_type = 'contact'
GROUP BY c_node.entity_id, c_node.properties->>'name', c_node.properties->>'title'
HAVING COUNT(DISTINCT a_node.id) > 1
ORDER BY account_count DESC;
```

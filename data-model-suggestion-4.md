# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Market Intelligence Aggregator · Created: 2026-05-11

## Philosophy

This model adds a **knowledge graph layer** on top of a relational foundation to capture the complex, multi-hop relationships that are central to competitive intelligence: "Company A acquired Company B, whose former CTO joined Company C, which just filed a patent in the same space as Company D's core product." These relationship chains are the highest-value intelligence that current CI tools cannot surface automatically.

The architecture uses PostgreSQL for all operational CRUD (organizations, users, sources, articles, battlecards) and adds a **graph layer** implemented either as dedicated graph tables within PostgreSQL (using `ltree` or adjacency list patterns with recursive CTEs) or as a complementary Neo4j/Apache AGE instance for complex traversal queries. The relational tables own transactional data; the graph owns the relationship intelligence.

This approach is inspired by Contify's "Athena" knowledge graph engine, Neo4j's use in fraud detection and competitive analysis, and the growing adoption of property graphs in enterprise intelligence platforms. The graph naturally models the entities (companies, persons, products, events) and relationships (competes_with, acquired, hired_from, partners_with, filed_patent_in) that CI analysts spend most of their time mapping manually.

**Best for:** Platforms targeting advanced competitive analysis — ownership chains, influence mapping, talent flow analysis, supply chain intelligence, conflict-of-interest detection — where relationship traversal across 2+ hops is a core feature.

**Trade-offs:**
- (+) Multi-hop relationship queries are natural and performant (vs. multi-join SQL)
- (+) Graph visualization of competitive landscapes is directly supported by the data model
- (+) New relationship types are added as edge labels, not schema changes
- (+) Enables AI-powered "connect the dots" analysis across entity networks
- (+) Pattern matching (e.g., "find companies with the same hiring pattern as X") is native
- (-) Dual storage (relational + graph) increases operational complexity
- (-) Graph query languages (Cypher, GQL) require team learning investment
- (-) PostgreSQL recursive CTEs are adequate for small graphs but degrade beyond ~10K nodes per org
- (-) Neo4j adds infrastructure cost and another system to maintain
- (-) Transaction coordination between relational and graph stores requires careful design

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RSS 2.0 / Atom 1.0 | Source management in relational tables; feed ingestion unchanged from other models |
| Schema.org NewsArticle | Entity extraction from Schema.org markup feeds into graph node/edge creation |
| ISO 8601 | All timestamps in both relational and graph layers use ISO 8601 |
| ISO 3166-1 | Company nodes include `country_code` property using ISO 3166-1 alpha-2 |
| ISO 17442 (LEI) | Company nodes include LEI for legal entity identification and cross-referencing |
| RFC 9309 (robots.txt) | Source compliance tracking in relational tables |
| GDPR / CCPA | Person nodes marked with consent status; graph edges involving PII respect erasure |
| SEC EDGAR XBRL | Filing events create graph edges (company --filed--> filing_event) with XBRL properties |
| W3C RDF / Property Graph Model | Graph schema follows the property graph model (nodes with properties, labeled directed edges) |
| GQL (ISO/IEC 39075:2024) | Graph queries written in GQL (or Cypher as a precursor) for portability |

---

## Relational Foundation (PostgreSQL)

### Core Operational Tables

```sql
-- Organizations and users are purely relational (same as other models)
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    profile         JSONB NOT NULL DEFAULT '{}',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_users_org ON users (organization_id);

-- Sources (same as hybrid model — JSONB config for source-type flexibility)
CREATE TABLE sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(500) NOT NULL,
    source_type     VARCHAR(50) NOT NULL,
    url             TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    config          JSONB NOT NULL DEFAULT '{}',
    health          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sources_org ON sources (organization_id);
CREATE INDEX idx_sources_active ON sources (is_active) WHERE is_active = TRUE;

-- Articles (raw ingested content — relational, feeds into graph)
CREATE TABLE articles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    source_id       UUID NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    title           TEXT NOT NULL,
    summary         TEXT,
    full_text       TEXT,
    content_hash    VARCHAR(64),
    published_at    TIMESTAMPTZ,
    fetched_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_duplicate    BOOLEAN NOT NULL DEFAULT FALSE,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_articles_org ON articles (organization_id, published_at DESC);
CREATE INDEX idx_articles_source ON articles (source_id);
CREATE INDEX idx_articles_hash ON articles (content_hash) WHERE content_hash IS NOT NULL;
CREATE INDEX idx_articles_fts ON articles USING GIN (
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(summary, ''))
);
```

### Battlecards & Win/Loss (Relational — Not Graph)

```sql
-- Battlecards are operational documents, not graph entities
CREATE TABLE battlecards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    competitor_node_id UUID NOT NULL,      -- references graph_nodes.id for the competitor company
    title           VARCHAR(500) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    version         INT NOT NULL DEFAULT 1,
    sections        JSONB NOT NULL DEFAULT '[]',
    governance      JSONB NOT NULL DEFAULT '{}',
    created_by      UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    approved_by     UUID REFERENCES users(id) ON DELETE SET NULL,
    approved_at     TIMESTAMPTZ,
    last_refreshed_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_battlecards_org ON battlecards (organization_id);
CREATE INDEX idx_battlecards_competitor ON battlecards (competitor_node_id);

-- Win/loss records
CREATE TABLE win_loss_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    competitor_node_id UUID,               -- references graph_nodes.id
    outcome         VARCHAR(20) NOT NULL,
    deal_date       DATE,
    details         JSONB NOT NULL DEFAULT '{}',
    recorded_by     UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_win_loss_org ON win_loss_records (organization_id, deal_date DESC);
```

### Alerts & Delivery (Relational)

```sql
CREATE TABLE alert_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    delivery_channel VARCHAR(50) NOT NULL,
    delivery_frequency VARCHAR(50) NOT NULL DEFAULT 'realtime',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    filters         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_alert_subs_user ON alert_subscriptions (user_id);
```

---

## Knowledge Graph Layer (PostgreSQL Implementation)

This section implements a property graph using two tables: `graph_nodes` and `graph_edges`. This approach works within PostgreSQL without requiring Neo4j, using recursive CTEs for traversal queries.

### Graph Nodes

```sql
-- Graph nodes represent entities in the competitive intelligence universe
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    node_type       VARCHAR(50) NOT NULL,  -- company, person, product, event, technology, market, patent
    label           VARCHAR(500) NOT NULL,  -- display name
    canonical_label VARCHAR(500) NOT NULL,  -- normalized for dedup/matching
    -- Properties vary by node type — stored as JSONB
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Node type-specific property examples:
    --
    -- company node:
    -- {
    --   "domain": "crayon.co",
    --   "country_code": "US",
    --   "industry": "Competitive Intelligence",
    --   "employee_count_range": "201-500",
    --   "funding_stage": "acquired",
    --   "stock_ticker": null,
    --   "sec_cik": null,
    --   "lei": null,
    --   "logo_url": "https://...",
    --   "aliases": ["Crayon Inc", "Crayon CI"],
    --   "signal_count": 45,
    --   "latest_signal_at": "2026-05-10T..."
    -- }
    --
    -- person node:
    -- {
    --   "title": "Chief Executive Officer",
    --   "current_company_node_id": "uuid",
    --   "linkedin_url": "https://...",
    --   "pii_consent_status": "public_professional",
    --   "data_retention_expiry": "2027-05-11T..."
    -- }
    --
    -- product node:
    -- {
    --   "company_node_id": "uuid",
    --   "product_type": "saas_platform",
    --   "pricing_model": "subscription",
    --   "launch_date": "2020-01-15",
    --   "category": "competitive_intelligence"
    -- }
    --
    -- event node (represents a signal/occurrence):
    -- {
    --   "event_type": "acquisition",
    --   "event_date": "2025-07-15",
    --   "source_article_id": "uuid",
    --   "summary": "SoftwareOne acquired Crayon for $1.4B",
    --   "sentiment": "neutral",
    --   "confidence": 0.98
    -- }
    --
    -- technology node:
    -- {
    --   "tech_category": "ai_ml",
    --   "sub_category": "nlp",
    --   "maturity": "growth"
    -- }
    --
    -- patent node:
    -- {
    --   "patent_number": "US-2026-12345",
    --   "filing_date": "2025-09-01",
    --   "title": "Method for dynamic pricing using ML",
    --   "status": "pending",
    --   "source_url": "https://..."
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_graph_nodes_org ON graph_nodes (organization_id);
CREATE INDEX idx_graph_nodes_type ON graph_nodes (organization_id, node_type);
CREATE INDEX idx_graph_nodes_label ON graph_nodes (organization_id, canonical_label);
CREATE INDEX idx_graph_nodes_properties ON graph_nodes USING GIN (properties jsonb_path_ops);
-- Specific index for domain lookups (company nodes)
CREATE INDEX idx_graph_nodes_domain ON graph_nodes ((properties->>'domain'))
    WHERE node_type = 'company';
```

### Graph Edges

```sql
-- Graph edges represent relationships between nodes
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(100) NOT NULL,  -- relationship label
    -- Edge types by source/target node types:
    --
    -- company -> company:
    --   competes_with, acquired, merged_with, partners_with, invested_in, supplies_to
    --
    -- person -> company:
    --   works_at, formerly_worked_at, founded, advises, board_member_of
    --
    -- person -> person:
    --   reports_to, co_founded_with
    --
    -- company -> product:
    --   offers, discontinued
    --
    -- product -> product:
    --   competes_with, integrates_with, replaced_by
    --
    -- company -> technology:
    --   uses, develops, acquired_for
    --
    -- company -> event:
    --   involved_in, caused, affected_by
    --
    -- company -> patent:
    --   filed, granted, cited_by
    --
    -- person -> event:
    --   announced, participated_in
    --
    -- event -> event:
    --   caused, followed_by, related_to
    --
    -- Edge properties (JSONB) for relationship metadata
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties:
    -- {
    --   "confidence": 0.95,
    --   "source": "ai_detected",          -- manual, ai_detected, imported, sec_filing
    --   "source_article_id": "uuid",
    --   "source_signal_id": "uuid",
    --   "effective_date": "2025-07-15",
    --   "end_date": null,                  -- null = still active
    --   "details": "Acquired for $1.4B in cash"
    -- }
    --
    -- Temporal properties for person -> company (works_at):
    -- {
    --   "confidence": 1.0,
    --   "source": "manual",
    --   "title": "VP Product",
    --   "start_date": "2023-06-01",
    --   "end_date": null,
    --   "is_current": true
    -- }
    weight          NUMERIC(5,4) DEFAULT 1.0,  -- for weighted graph algorithms
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_graph_edges_org ON graph_edges (organization_id);
CREATE INDEX idx_graph_edges_source ON graph_edges (source_node_id);
CREATE INDEX idx_graph_edges_target ON graph_edges (target_node_id);
CREATE INDEX idx_graph_edges_type ON graph_edges (edge_type);
CREATE INDEX idx_graph_edges_source_type ON graph_edges (source_node_id, edge_type);
CREATE INDEX idx_graph_edges_target_type ON graph_edges (target_node_id, edge_type);
CREATE INDEX idx_graph_edges_properties ON graph_edges USING GIN (properties jsonb_path_ops);
-- Unique constraint to prevent duplicate edges of the same type
CREATE UNIQUE INDEX idx_graph_edges_unique ON graph_edges
    (organization_id, source_node_id, target_node_id, edge_type)
    WHERE is_active = TRUE;
```

---

## Signals as Graph Events

```sql
-- Signals are stored in both the relational layer (for feeds/queries)
-- and linked to the graph (for relationship analysis)
CREATE TABLE signals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    article_id      UUID REFERENCES articles(id) ON DELETE SET NULL,
    -- Graph node references (signals may reference event nodes in the graph)
    event_node_id   UUID REFERENCES graph_nodes(id) ON DELETE SET NULL,
    company_node_id UUID REFERENCES graph_nodes(id) ON DELETE SET NULL,
    signal_type     VARCHAR(100) NOT NULL,
    signal_category VARCHAR(100) NOT NULL,
    title           TEXT NOT NULL,
    summary         TEXT,
    sentiment       VARCHAR(20),
    sentiment_score NUMERIC(4,3),
    confidence      NUMERIC(3,2) NOT NULL DEFAULT 0.50,
    relevance_score NUMERIC(3,2),
    impact_level    VARCHAR(20),
    source_url      TEXT,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_dismissed    BOOLEAN NOT NULL DEFAULT FALSE,
    -- JSONB for NLP extraction details and entity references
    extraction      JSONB NOT NULL DEFAULT '{}',
    -- Example extraction:
    -- {
    --   "entities": [
    --     {"type": "company", "value": "Crayon", "node_id": "uuid", "confidence": 0.98},
    --     {"type": "person", "value": "John CEO", "node_id": "uuid", "confidence": 0.90},
    --     {"type": "amount", "value": "$1.4B", "context": "acquisition_price"}
    --   ],
    --   "topics": ["acquisition", "consolidation"],
    --   "graph_edges_created": ["edge-uuid-1", "edge-uuid-2"],
    --   "nlp_model_version": "ner-v3.2"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_signals_org ON signals (organization_id, detected_at DESC);
CREATE INDEX idx_signals_company ON signals (company_node_id, detected_at DESC);
CREATE INDEX idx_signals_event ON signals (event_node_id) WHERE event_node_id IS NOT NULL;
CREATE INDEX idx_signals_type ON signals (signal_type);
CREATE INDEX idx_signals_category ON signals (signal_category);
CREATE INDEX idx_signals_active ON signals (organization_id, is_dismissed, detected_at DESC)
    WHERE is_dismissed = FALSE;
CREATE INDEX idx_signals_fts ON signals USING GIN (
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(summary, ''))
);
```

---

## Graph Traversal Queries (PostgreSQL Recursive CTEs)

```sql
-- 1. Find all companies connected to Competitor X within 2 hops
WITH RECURSIVE connected AS (
    -- Start from Competitor X
    SELECT
        gn.id AS node_id,
        gn.label AS node_label,
        gn.node_type,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.id = '<competitor-x-uuid>'
      AND gn.organization_id = '<org-uuid>'

    UNION ALL

    -- Traverse edges outward
    SELECT
        gn2.id AS node_id,
        gn2.label AS node_label,
        gn2.node_type,
        c.depth + 1 AS depth,
        c.path || gn2.id
    FROM connected c
    JOIN graph_edges ge ON (ge.source_node_id = c.node_id OR ge.target_node_id = c.node_id)
    JOIN graph_nodes gn2 ON gn2.id = CASE
        WHEN ge.source_node_id = c.node_id THEN ge.target_node_id
        ELSE ge.source_node_id
    END
    WHERE c.depth < 2
      AND NOT gn2.id = ANY(c.path)  -- prevent cycles
      AND ge.is_active = TRUE
      AND ge.organization_id = '<org-uuid>'
)
SELECT DISTINCT node_id, node_label, node_type, depth
FROM connected
WHERE depth > 0
ORDER BY depth, node_label;

-- 2. Talent flow analysis: Where did Competitor X's former employees go?
SELECT
    target_company.label AS joined_company,
    person.label AS person_name,
    left_edge.properties->>'title' AS previous_title,
    left_edge.properties->>'end_date' AS left_date,
    joined_edge.properties->>'title' AS new_title,
    joined_edge.properties->>'start_date' AS joined_date
FROM graph_edges left_edge
JOIN graph_nodes person ON person.id = left_edge.source_node_id AND person.node_type = 'person'
JOIN graph_edges joined_edge ON joined_edge.source_node_id = person.id
    AND joined_edge.edge_type = 'works_at'
    AND joined_edge.is_active = TRUE
JOIN graph_nodes target_company ON target_company.id = joined_edge.target_node_id
    AND target_company.node_type = 'company'
WHERE left_edge.target_node_id = '<competitor-x-uuid>'
  AND left_edge.edge_type = 'formerly_worked_at'
  AND left_edge.organization_id = '<org-uuid>'
ORDER BY left_edge.properties->>'end_date' DESC;

-- 3. Find companies that share technology with Competitor X
SELECT
    other_company.label AS company_name,
    tech.label AS shared_technology,
    tech.properties->>'tech_category' AS tech_category
FROM graph_edges e1
JOIN graph_nodes tech ON tech.id = e1.target_node_id AND tech.node_type = 'technology'
JOIN graph_edges e2 ON e2.target_node_id = tech.id
    AND e2.edge_type IN ('uses', 'develops')
    AND e2.source_node_id != e1.source_node_id
JOIN graph_nodes other_company ON other_company.id = e2.source_node_id
    AND other_company.node_type = 'company'
WHERE e1.source_node_id = '<competitor-x-uuid>'
  AND e1.edge_type IN ('uses', 'develops')
  AND e1.organization_id = '<org-uuid>';

-- 4. Competitive threat score: Count signals and relationship density for a company
SELECT
    gn.label AS company_name,
    gn.properties->>'industry' AS industry,
    COUNT(DISTINCT ge.id) AS relationship_count,
    COUNT(DISTINCT s.id) AS signal_count_30d,
    AVG(s.sentiment_score) AS avg_sentiment_30d,
    COUNT(DISTINCT CASE WHEN ge.edge_type = 'competes_with' THEN ge.id END) AS competitor_edge_count,
    COUNT(DISTINCT CASE WHEN s.signal_category = 'pricing' THEN s.id END) AS pricing_signal_count
FROM graph_nodes gn
LEFT JOIN graph_edges ge ON (ge.source_node_id = gn.id OR ge.target_node_id = gn.id)
    AND ge.is_active = TRUE
LEFT JOIN signals s ON s.company_node_id = gn.id
    AND s.detected_at >= NOW() - INTERVAL '30 days'
    AND s.is_dismissed = FALSE
WHERE gn.organization_id = '<org-uuid>'
  AND gn.node_type = 'company'
  AND gn.is_active = TRUE
GROUP BY gn.id, gn.label, gn.properties->>'industry'
ORDER BY signal_count_30d DESC, relationship_count DESC;

-- 5. Cross-signal synthesis: Find companies with correlated signal patterns
-- (hiring + pricing change + patent filing = potential pivot)
SELECT
    gn.label AS company_name,
    COUNT(DISTINCT CASE WHEN s.signal_category = 'talent' THEN s.id END) AS hiring_signals,
    COUNT(DISTINCT CASE WHEN s.signal_category = 'pricing' THEN s.id END) AS pricing_signals,
    COUNT(DISTINCT CASE WHEN s.signal_category = 'ip' THEN s.id END) AS patent_signals,
    COUNT(DISTINCT s.id) AS total_signals
FROM graph_nodes gn
JOIN signals s ON s.company_node_id = gn.id
    AND s.detected_at >= NOW() - INTERVAL '90 days'
    AND s.is_dismissed = FALSE
WHERE gn.organization_id = '<org-uuid>'
  AND gn.node_type = 'company'
GROUP BY gn.id, gn.label
HAVING COUNT(DISTINCT CASE WHEN s.signal_category = 'talent' THEN s.id END) >= 2
   AND COUNT(DISTINCT CASE WHEN s.signal_category = 'pricing' THEN s.id END) >= 1
ORDER BY total_signals DESC;
```

---

## Graph Maintenance

```sql
-- NLP pipeline output: entities extracted and linked to graph
CREATE TABLE entity_resolution_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    signal_id       UUID NOT NULL REFERENCES signals(id) ON DELETE CASCADE,
    raw_entity_text VARCHAR(500) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    resolved_node_id UUID REFERENCES graph_nodes(id) ON DELETE SET NULL,
    resolution_method VARCHAR(50) NOT NULL,  -- exact_match, fuzzy_match, alias_match, created_new
    confidence      NUMERIC(3,2) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_entity_resolution_signal ON entity_resolution_log (signal_id);
CREATE INDEX idx_entity_resolution_node ON entity_resolution_log (resolved_node_id);

-- Graph snapshots for temporal analysis
-- (Periodically snapshot node/edge counts and key metrics per company)
CREATE TABLE graph_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    snapshot_date   DATE NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    node_id         UUID NOT NULL,
    metrics         JSONB NOT NULL DEFAULT '{}',
    -- Example metrics for company node:
    -- {
    --   "edge_count": 25,
    --   "competitor_count": 8,
    --   "employee_node_count": 15,
    --   "signal_count_30d": 12,
    --   "avg_sentiment_30d": -0.3,
    --   "technology_count": 5
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_graph_snapshots_org ON graph_snapshots (organization_id, snapshot_date DESC);
CREATE INDEX idx_graph_snapshots_node ON graph_snapshots (node_id, snapshot_date DESC);

-- Audit log for graph mutations
CREATE TABLE graph_audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,  -- node_created, node_updated, edge_created, edge_deactivated
    node_id         UUID,
    edge_id         UUID,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_graph_audit_org ON graph_audit_log (organization_id, created_at DESC);
```

---

## Neo4j Alternative Schema (For Large-Scale Deployments)

For organizations with large competitive graphs (10,000+ nodes), the PostgreSQL recursive CTE approach may become slow. In that case, maintain the relational tables in PostgreSQL and sync the graph to Neo4j.

```cypher
// Neo4j node and relationship definitions (Cypher)

// Company nodes
CREATE (c:Company {
  id: "uuid",
  org_id: "uuid",
  name: "Crayon",
  canonical_name: "crayon",
  domain: "crayon.co",
  country_code: "US",
  industry: "Competitive Intelligence",
  employee_count_range: "201-500",
  funding_stage: "acquired",
  signal_count: 45,
  latest_signal_at: datetime("2026-05-10T14:30:00Z")
})

// Person nodes
CREATE (p:Person {
  id: "uuid",
  org_id: "uuid",
  name: "John Doe",
  title: "CEO",
  pii_consent_status: "public_professional"
})

// Product nodes
CREATE (pr:Product {
  id: "uuid",
  org_id: "uuid",
  name: "Crayon Platform",
  company_id: "uuid",
  category: "competitive_intelligence"
})

// Event nodes
CREATE (e:Event {
  id: "uuid",
  org_id: "uuid",
  event_type: "acquisition",
  event_date: date("2025-07-15"),
  summary: "SoftwareOne acquired Crayon for $1.4B"
})

// Relationships
CREATE (c1)-[:COMPETES_WITH {confidence: 0.95, source: "manual"}]->(c2)
CREATE (c1)-[:ACQUIRED {effective_date: date("2025-07-15"), amount: "$1.4B"}]->(c2)
CREATE (p)-[:WORKS_AT {title: "CEO", start_date: date("2020-01-01"), is_current: true}]->(c)
CREATE (p)-[:FORMERLY_WORKED_AT {title: "VP Product", end_date: date("2023-06-01")}]->(c2)
CREATE (c)-[:OFFERS]->(pr)
CREATE (c)-[:INVOLVED_IN]->(e)

// Talent flow query in Cypher (much more readable than recursive CTE):
MATCH (source:Company {id: "competitor-x-uuid"})
      <-[:FORMERLY_WORKED_AT]-(person:Person)
      -[:WORKS_AT]->(target:Company)
RETURN person.name, source.name AS from_company, target.name AS to_company,
       person.title AS current_title
ORDER BY person.name

// 2-hop competitive landscape:
MATCH (c:Company {id: "competitor-x-uuid"})-[r*1..2]-(connected)
WHERE connected.org_id = "org-uuid"
RETURN DISTINCT connected, length(r) AS distance
ORDER BY distance
```

---

## Sync Between PostgreSQL and Neo4j

```sql
-- Change data capture table for syncing relational -> graph
CREATE TABLE graph_sync_queue (
    id              BIGSERIAL PRIMARY KEY,
    operation       VARCHAR(20) NOT NULL,  -- create_node, update_node, create_edge, deactivate_edge
    entity_type     VARCHAR(50) NOT NULL,  -- node or edge
    entity_id       UUID NOT NULL,
    payload         JSONB NOT NULL,
    synced          BOOLEAN NOT NULL DEFAULT FALSE,
    synced_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_graph_sync_pending ON graph_sync_queue (synced, created_at)
    WHERE synced = FALSE;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organizations & Users | 2 | organizations, users |
| Sources | 1 | sources (JSONB config) |
| Articles | 1 | articles (raw content) |
| Graph Layer | 2 | graph_nodes, graph_edges |
| Signals | 1 | signals (linked to graph nodes) |
| Battlecards & Win/Loss | 2 | battlecards, win_loss_records |
| Alerts | 1 | alert_subscriptions |
| Graph Maintenance | 3 | entity_resolution_log, graph_snapshots, graph_audit_log |
| Graph Sync | 1 | graph_sync_queue (only if using Neo4j) |
| **Total** | **14** | (13 without Neo4j sync) |

---

## Key Design Decisions

1. **Graph as a first-class layer, not an afterthought** — the `graph_nodes` and `graph_edges` tables are central to the architecture, not a bolt-on. Companies, persons, products, technologies, and events are all graph nodes. This means the system naturally captures "Company A hired Person B from Company C to build Product D using Technology E" as a connected subgraph.

2. **PostgreSQL-native graph with Neo4j escape hatch** — the default implementation uses PostgreSQL's recursive CTEs, which perform well for graphs up to ~10K nodes per organization. For larger deployments, Neo4j can be added as a read-only graph store synced from PostgreSQL via the `graph_sync_queue`. This avoids premature infrastructure complexity.

3. **Signals bridge relational and graph layers** — each signal has both relational fields (for feed queries, filtering, and dashboards) and graph references (`event_node_id`, `company_node_id`). When a signal is detected, the NLP pipeline both creates the relational signal record AND creates/updates graph nodes and edges.

4. **Entity resolution is explicit** — the `entity_resolution_log` tracks how extracted entity mentions were resolved to graph nodes. This is critical for debugging NLP accuracy and for improving entity resolution over time.

5. **Temporal graph via edge properties** — rather than using bitemporal tables, temporal information is stored as edge properties (`effective_date`, `end_date`, `is_current`). The `graph_snapshots` table provides periodic metric snapshots for trend analysis without requiring full graph replay.

6. **Edge types are labels, not schema** — new relationship types (e.g., `supplies_to`, `cited_in_patent`) are added by creating edges with new `edge_type` values. No schema migration required. This matches the property graph model and enables rapid evolution of the intelligence taxonomy.

7. **Battlecards reference graph nodes, not a separate companies table** — there is no separate `companies` table. Company data lives in `graph_nodes` where `node_type = 'company'`. Battlecards reference `competitor_node_id` directly. This avoids data duplication between a companies table and company graph nodes.

8. **Graph visualization is directly queryable** — the node/edge structure maps directly to graph visualization libraries (D3.js, Cytoscape.js, vis.js). A single query returns the nodes and edges needed to render a competitive landscape visualization without post-processing.

9. **Weight field on edges for graph algorithms** — the `weight` field enables PageRank-style importance scoring, community detection, and shortest-path algorithms. These can power features like "most influential company in your competitive landscape" or "companies most likely to affect your market."

10. **GDPR compliance via node properties and edge deactivation** — person nodes include `pii_consent_status` and `data_retention_expiry`. Erasure is handled by deactivating edges and redacting node properties, not deleting nodes (which would break graph integrity). The `graph_audit_log` records all mutations for compliance auditing.

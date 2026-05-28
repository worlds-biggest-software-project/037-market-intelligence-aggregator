# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Market Intelligence Aggregator · Created: 2026-05-11

## Philosophy

This model uses a **pragmatic hybrid** of traditional relational tables for stable, well-understood entities (organizations, users, companies, sources) and **JSONB columns for variable, evolving, or domain-specific data** (signal payloads, entity extractions, NLP metadata, jurisdiction-specific fields, connector configurations). The core structural relationships are enforced via foreign keys, but the "shape" of intelligence data remains flexible without requiring schema migrations.

This is the architecture favoured by modern SaaS platforms that need to ship quickly, support diverse customer use cases, and evolve their data model without coordinated downtime. PostgreSQL's JSONB type with GIN indexing provides document-database-like flexibility while retaining the transactional guarantees, joins, and query planner of a relational system. Platforms like Notion, Linear, and Shopify use this hybrid approach for their flexible metadata layers.

The key advantage is **development velocity and flexibility**. New signal types, new NLP extractors, new source connectors, and multi-tenant customisation can all be supported by adding fields to JSONB columns rather than running schema migrations. The key cost is **weaker type safety** for JSONB fields — validation must be enforced at the application layer rather than the database layer.

**Best for:** Rapid MVP development, teams expecting frequent schema evolution, multi-region deployments where signal types vary by jurisdiction, and projects that want to balance structure with flexibility.

**Trade-offs:**
- (+) New signal types and metadata fields added without schema migrations
- (+) Each organization can track custom fields without affecting others
- (+) JSONB GIN indexes provide fast containment queries on nested data
- (+) Fewer tables than normalized model (~20 vs. ~31) — simpler to understand
- (+) PostgreSQL JSONB is battle-tested at scale (used by Notion, Linear, etc.)
- (-) JSONB fields lack database-level type enforcement — bugs can corrupt data silently
- (-) Complex JSONB queries are harder to write and optimize than simple column queries
- (-) JSONB columns can become "junk drawers" without discipline
- (-) Schema documentation must be maintained manually (or via JSON Schema validation)
- (-) Reporting/BI tools handle JSONB less well than flat columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RSS 2.0 / Atom 1.0 | Source `config` JSONB stores feed-specific settings (format, auth, user-agent) |
| Schema.org NewsArticle | Article `metadata` JSONB preserves full Schema.org extraction without fixed columns |
| ISO 8601 | All timestamp columns use `TIMESTAMPTZ`; JSONB dates stored as ISO 8601 strings |
| RFC 9309 (robots.txt) | Source `config.robots_txt_status` tracked in JSONB alongside other source settings |
| ISO 3166-1 | Company `details.country_code` in JSONB uses ISO 3166-1 alpha-2 |
| GDPR / CCPA | Person `privacy` JSONB stores consent status, retention policy, erasure requests |
| SEC EDGAR XBRL | Filing signal `payload` JSONB stores structured XBRL facts natively as nested JSON |
| JSON Schema (Draft 2020-12) | Application layer validates JSONB payloads against registered JSON Schemas |
| OpenAPI 3.1 | API documentation includes JSONB field schemas as `additionalProperties` |
| MCP (Model Context Protocol) | Intelligence queries exposed via MCP tools; JSONB payloads map directly to MCP responses |

---

## Organizations & Users

```sql
-- Multi-tenant organization
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    -- JSONB for org-level settings that vary by customer
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "default_crawl_frequency_minutes": 60,
    --   "max_tracked_companies": 50,
    --   "features_enabled": ["battlecards", "narratives", "mcp_server"],
    --   "notification_defaults": {"channel": "email", "frequency": "daily"},
    --   "custom_signal_types": [
    --     {"id": "customer_churn_mention", "category": "market", "display_name": "Customer Churn Signal"}
    --   ],
    --   "custom_fields": {
    --     "company": [
    --       {"key": "segment", "label": "Market Segment", "type": "select", "options": ["SMB", "Mid-Market", "Enterprise"]}
    --     ]
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    -- JSONB for user preferences and profile data
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example profile:
    -- {
    --   "job_function": "product_marketing",
    --   "notification_preferences": {
    --     "channels": ["slack", "email"],
    --     "frequency": "realtime",
    --     "min_relevance": 0.7,
    --     "categories": ["pricing", "product", "talent"]
    --   },
    --   "oidc_subject": "auth0|123456",
    --   "scim_external_id": "scim-789",
    --   "timezone": "America/New_York",
    --   "deal_context": {
    --     "active_competitors": ["uuid1", "uuid2"],
    --     "focus_topics": ["ai_pricing", "enterprise_security"]
    --   }
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_users_org ON users (organization_id);
CREATE INDEX idx_users_email ON users (email);
-- Index for querying users by job function
CREATE INDEX idx_users_job_function ON users USING GIN ((profile -> 'job_function'));
```

---

## Companies

```sql
-- Tracked companies — core fields relational, variable metadata in JSONB
CREATE TABLE companies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(500) NOT NULL,
    canonical_name  VARCHAR(500) NOT NULL,
    domain          VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- JSONB for all variable company attributes
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "country_code": "US",
    --   "industry": "Enterprise Software",
    --   "sub_industry": "Competitive Intelligence",
    --   "employee_count_range": "51-200",
    --   "funding_stage": "series_b",
    --   "stock_ticker": "SMWB",
    --   "sec_cik": "0001856028",
    --   "lei": "5493001KJTIIGC8Y1R12",
    --   "logo_url": "https://...",
    --   "description": "...",
    --   "aliases": ["SMWB", "SimilarWeb Ltd", "Similar Web"],
    --   "relationships": [
    --     {"target_id": "uuid", "type": "competitor", "confidence": 0.95},
    --     {"target_id": "uuid", "type": "partner", "confidence": 0.80}
    --   ],
    --   "custom_fields": {
    --     "segment": "Mid-Market",
    --     "priority": "high",
    --     "assigned_analyst": "uuid"
    --   }
    -- }
    -- Denormalized counters for dashboard performance
    signal_count    INT NOT NULL DEFAULT 0,
    latest_signal_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, canonical_name)
);

CREATE INDEX idx_companies_org ON companies (organization_id);
CREATE INDEX idx_companies_domain ON companies (domain);
CREATE INDEX idx_companies_active ON companies (organization_id, is_active) WHERE is_active = TRUE;
-- GIN index for JSONB containment queries
CREATE INDEX idx_companies_details ON companies USING GIN (details jsonb_path_ops);
-- Specific index for industry filtering
CREATE INDEX idx_companies_industry ON companies ((details->>'industry'));
-- Index for ticker lookups
CREATE INDEX idx_companies_ticker ON companies ((details->>'stock_ticker'))
    WHERE details->>'stock_ticker' IS NOT NULL;

-- Example query: Find all fintech companies in the US
-- SELECT * FROM companies
-- WHERE organization_id = '...'
--   AND details @> '{"country_code": "US", "industry": "Fintech"}';
--
-- Example query: Search by alias
-- SELECT * FROM companies
-- WHERE organization_id = '...'
--   AND details->'aliases' ? 'MSFT';
```

---

## Sources

```sql
-- Intelligence sources with connector-specific config in JSONB
CREATE TABLE sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(500) NOT NULL,
    source_type     VARCHAR(50) NOT NULL,  -- rss, atom, website, api, sec_edgar, review_site, job_board, gdelt
    url             TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- JSONB for source-type-specific configuration
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config for RSS:
    -- {
    --   "feed_format": "rss_2_0",
    --   "crawl_frequency_minutes": 60,
    --   "robots_txt_status": "compliant",
    --   "robots_txt_checked_at": "2026-05-10T...",
    --   "user_agent": "MarketIntelBot/1.0 (+https://example.com/bot)",
    --   "auth": null
    -- }
    --
    -- Example config for SEC EDGAR:
    -- {
    --   "crawl_frequency_minutes": 360,
    --   "filing_types": ["10-K", "10-Q", "8-K"],
    --   "company_ciks": ["0001326801", "0001652044"],
    --   "user_agent": "CompanyName admin@company.com"
    -- }
    --
    -- Example config for website scraper:
    -- {
    --   "crawl_frequency_minutes": 1440,
    --   "robots_txt_status": "compliant",
    --   "selectors": {
    --     "article_list": "div.blog-post",
    --     "title": "h2.post-title",
    --     "date": "time.post-date",
    --     "content": "div.post-body"
    --   },
    --   "screenshot_on_change": true
    -- }
    -- Health tracking
    health          JSONB NOT NULL DEFAULT '{}',
    -- Example health:
    -- {
    --   "last_fetched_at": "2026-05-11T...",
    --   "last_successful_at": "2026-05-11T...",
    --   "consecutive_failures": 0,
    --   "total_articles": 1250,
    --   "total_signals": 340,
    --   "avg_fetch_duration_ms": 850
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sources_org ON sources (organization_id);
CREATE INDEX idx_sources_type ON sources (source_type);
CREATE INDEX idx_sources_active ON sources (organization_id, is_active) WHERE is_active = TRUE;
CREATE INDEX idx_sources_health ON sources USING GIN (health jsonb_path_ops);

-- Source-to-company coverage (which companies does this source cover?)
CREATE TABLE source_companies (
    source_id       UUID NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    company_id      UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    PRIMARY KEY (source_id, company_id)
);
```

---

## Articles & Signals (The Intelligence Core)

```sql
-- Ingested articles/content — core fields relational, extracted metadata in JSONB
CREATE TABLE articles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    source_id       UUID NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    title           TEXT NOT NULL,
    summary         TEXT,
    content_hash    VARCHAR(64),
    published_at    TIMESTAMPTZ,
    fetched_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_duplicate    BOOLEAN NOT NULL DEFAULT FALSE,
    -- JSONB for all extracted and variable metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "author": "Jane Smith",
    --   "publisher": "TechCrunch",
    --   "language": "en",
    --   "external_id": "rss-guid-12345",
    --   "full_text": "...",
    --   "word_count": 1250,
    --   "schema_org": {
    --     "@type": "NewsArticle",
    --     "datePublished": "2026-05-10",
    --     "publisher": {"@type": "Organization", "name": "TechCrunch"}
    --   },
    --   "images": ["https://..."],
    --   "tags": ["ai", "pricing", "enterprise"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_articles_org ON articles (organization_id, published_at DESC);
CREATE INDEX idx_articles_source ON articles (source_id);
CREATE INDEX idx_articles_hash ON articles (content_hash) WHERE content_hash IS NOT NULL;
CREATE INDEX idx_articles_metadata ON articles USING GIN (metadata jsonb_path_ops);

-- Full-text search
CREATE INDEX idx_articles_fts ON articles USING GIN (
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(summary, ''))
);

-- Competitive signals — the primary intelligence unit
-- Core classification fields are relational; payload and entities are JSONB
CREATE TABLE signals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    article_id      UUID REFERENCES articles(id) ON DELETE SET NULL,
    company_id      UUID REFERENCES companies(id) ON DELETE SET NULL,
    signal_type     VARCHAR(100) NOT NULL,
    signal_category VARCHAR(100) NOT NULL,
    title           TEXT NOT NULL,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_dismissed    BOOLEAN NOT NULL DEFAULT FALSE,
    -- JSONB for all NLP outputs, entities, scores, and variable data
    payload         JSONB NOT NULL DEFAULT '{}',
    -- Example payload:
    -- {
    --   "summary": "Competitor X announced a 20% price reduction on Enterprise tier",
    --   "sentiment": "negative",
    --   "sentiment_score": -0.75,
    --   "confidence": 0.92,
    --   "relevance_score": 0.88,
    --   "impact_level": "high",
    --   "source_url": "https://...",
    --   "published_at": "2026-05-10T14:30:00Z",
    --   "entities": [
    --     {"type": "company", "value": "Competitor X", "resolved_id": "uuid", "confidence": 0.98},
    --     {"type": "amount", "value": "20%", "context": "price_reduction"},
    --     {"type": "product", "value": "Enterprise tier", "confidence": 0.85},
    --     {"type": "person", "value": "CEO Jane Smith", "resolved_id": "uuid", "title": "CEO"}
    --   ],
    --   "topics": [
    --     {"id": "uuid", "name": "pricing", "confidence": 0.95},
    --     {"id": "uuid", "name": "enterprise", "confidence": 0.80}
    --   ],
    --   "nlp_model_version": "ner-v3.2",
    --   "dismissed_by": null,
    --   "dismissed_at": null,
    --   "dismiss_reason": null,
    --   "filing_data": null,
    --   "xbrl_facts": null
    -- }
    --
    -- Example payload for SEC filing signal:
    -- {
    --   "summary": "Q4 2025 earnings: revenue up 15% YoY",
    --   "sentiment": "positive",
    --   "sentiment_score": 0.6,
    --   "confidence": 0.99,
    --   "filing_data": {
    --     "filing_type": "10-K",
    --     "filing_date": "2026-02-28",
    --     "accession_number": "0001326801-26-000012",
    --     "sec_url": "https://data.sec.gov/..."
    --   },
    --   "xbrl_facts": {
    --     "revenue": {"value": 50000000, "unit": "USD", "period": "2025-Q4"},
    --     "net_income": {"value": 5000000, "unit": "USD", "period": "2025-Q4"}
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_signals_org ON signals (organization_id, detected_at DESC);
CREATE INDEX idx_signals_company ON signals (company_id, detected_at DESC);
CREATE INDEX idx_signals_type ON signals (signal_type);
CREATE INDEX idx_signals_category ON signals (signal_category);
CREATE INDEX idx_signals_active ON signals (organization_id, is_dismissed, detected_at DESC)
    WHERE is_dismissed = FALSE;
-- GIN index for JSONB payload queries
CREATE INDEX idx_signals_payload ON signals USING GIN (payload jsonb_path_ops);
-- Specific indexes for common JSONB queries
CREATE INDEX idx_signals_sentiment ON signals ((payload->>'sentiment'));
CREATE INDEX idx_signals_impact ON signals ((payload->>'impact_level'));
CREATE INDEX idx_signals_relevance ON signals (
    organization_id,
    ((payload->>'relevance_score')::numeric) DESC NULLS LAST
) WHERE is_dismissed = FALSE;

-- Full-text search on signal titles and summaries
CREATE INDEX idx_signals_fts ON signals USING GIN (
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(payload->>'summary', ''))
);

-- Example queries:
--
-- Find high-impact pricing signals for a company:
-- SELECT * FROM signals
-- WHERE organization_id = '...'
--   AND company_id = '...'
--   AND signal_category = 'pricing'
--   AND payload @> '{"impact_level": "high"}'
--   AND is_dismissed = FALSE
-- ORDER BY detected_at DESC;
--
-- Find signals mentioning a specific entity:
-- SELECT * FROM signals
-- WHERE organization_id = '...'
--   AND payload->'entities' @> '[{"type": "person", "value": "Jane Smith"}]'
-- ORDER BY detected_at DESC;
--
-- Aggregate sentiment by company over time:
-- SELECT
--     company_id,
--     date_trunc('week', detected_at) AS week,
--     AVG((payload->>'sentiment_score')::numeric) AS avg_sentiment,
--     COUNT(*) AS signal_count
-- FROM signals
-- WHERE organization_id = '...' AND is_dismissed = FALSE
-- GROUP BY company_id, date_trunc('week', detected_at)
-- ORDER BY week DESC;
```

---

## Battlecards

```sql
-- Battlecards — relational structure with JSONB sections
CREATE TABLE battlecards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    competitor_id   UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    version         INT NOT NULL DEFAULT 1,
    created_by      UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    approved_by     UUID REFERENCES users(id) ON DELETE SET NULL,
    approved_at     TIMESTAMPTZ,
    last_refreshed_at TIMESTAMPTZ,
    -- JSONB for modular sections — avoids a separate sections table
    sections        JSONB NOT NULL DEFAULT '[]',
    -- Example sections:
    -- [
    --   {
    --     "id": "uuid",
    --     "section_type": "overview",
    --     "title": "Company Overview",
    --     "content": "Competitor X is a ...",
    --     "sort_order": 0,
    --     "ai_generated": false,
    --     "source_signal_ids": ["uuid1", "uuid2"],
    --     "last_verified_at": "2026-05-10T...",
    --     "verified_by": "uuid"
    --   },
    --   {
    --     "id": "uuid",
    --     "section_type": "objections",
    --     "title": "Common Objections & Responses",
    --     "content": "## 'Their product is cheaper'\n\nTalk track: ...",
    --     "sort_order": 3,
    --     "ai_generated": true,
    --     "model_version": "claude-4-sonnet",
    --     "source_signal_ids": ["uuid3", "uuid4", "uuid5"],
    --     "last_verified_at": null,
    --     "verified_by": null
    --   }
    -- ]
    -- Governance metadata
    governance      JSONB NOT NULL DEFAULT '{}',
    -- Example governance:
    -- {
    --   "review_cycle_days": 30,
    --   "last_review_date": "2026-04-15",
    --   "next_review_date": "2026-05-15",
    --   "reviewers": ["uuid1", "uuid2"],
    --   "change_log": [
    --     {"date": "2026-04-15", "user": "uuid1", "action": "approved", "sections_updated": ["pricing", "objections"]}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_battlecards_org ON battlecards (organization_id);
CREATE INDEX idx_battlecards_competitor ON battlecards (competitor_id);
CREATE INDEX idx_battlecards_status ON battlecards (organization_id, status);
CREATE INDEX idx_battlecards_sections ON battlecards USING GIN (sections jsonb_path_ops);
```

---

## Win/Loss & Collaboration

```sql
-- Win/loss records with flexible attributes
CREATE TABLE win_loss_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    competitor_id   UUID REFERENCES companies(id) ON DELETE SET NULL,
    outcome         VARCHAR(20) NOT NULL,
    recorded_by     UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    deal_date       DATE,
    -- JSONB for all deal-specific details (varies by org)
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "deal_value": 150000,
    --   "deal_currency": "USD",
    --   "loss_reason": "pricing",
    --   "win_reason": null,
    --   "notes": "Lost on price; competitor offered 30% discount",
    --   "interview_transcript": "...",
    --   "deal_stage": "final_evaluation",
    --   "buyer_persona": "VP Engineering",
    --   "custom_fields": {
    --     "region": "EMEA",
    --     "deal_type": "new_business"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_win_loss_org ON win_loss_records (organization_id, deal_date DESC);
CREATE INDEX idx_win_loss_competitor ON win_loss_records (competitor_id);
CREATE INDEX idx_win_loss_outcome ON win_loss_records (organization_id, outcome);

-- Intelligence boards
CREATE TABLE boards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    visibility      VARCHAR(50) NOT NULL DEFAULT 'team',
    created_by      UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    -- JSONB for board items — keeps it simple, avoids junction table
    items           JSONB NOT NULL DEFAULT '[]',
    -- Example items:
    -- [
    --   {
    --     "id": "uuid",
    --     "type": "signal",
    --     "ref_id": "signal-uuid",
    --     "note": "This pricing change is critical for Q3 deals",
    --     "added_by": "user-uuid",
    --     "added_at": "2026-05-11T...",
    --     "sort_order": 0
    --   }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_boards_org ON boards (organization_id);
```

---

## Alerts, Digests & Narratives

```sql
-- Alert subscriptions with JSONB filters
CREATE TABLE alert_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    delivery_channel VARCHAR(50) NOT NULL,
    delivery_frequency VARCHAR(50) NOT NULL DEFAULT 'realtime',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- JSONB for flexible filter criteria
    filters         JSONB NOT NULL DEFAULT '{}',
    -- Example filters:
    -- {
    --   "companies": ["uuid1", "uuid2"],
    --   "signal_types": ["pricing_change", "executive_hire"],
    --   "categories": ["pricing", "talent"],
    --   "min_relevance": 0.7,
    --   "min_impact": "high",
    --   "topics": ["enterprise", "ai"],
    --   "sentiment": ["negative", "mixed"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_alert_subs_user ON alert_subscriptions (user_id);
CREATE INDEX idx_alert_subs_org ON alert_subscriptions (organization_id);

-- AI-generated narratives with flexible evidence
CREATE TABLE narratives (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    company_id      UUID REFERENCES companies(id) ON DELETE SET NULL,
    narrative_type  VARCHAR(50) NOT NULL,
    title           TEXT NOT NULL,
    content         TEXT NOT NULL,
    -- JSONB for all AI generation metadata and evidence
    ai_metadata     JSONB NOT NULL DEFAULT '{}',
    -- Example ai_metadata:
    -- {
    --   "model_version": "claude-4-opus",
    --   "confidence": 0.85,
    --   "evidence_signals": ["uuid1", "uuid2", "uuid3"],
    --   "evidence_summary": "Based on 3 signals: 2 hiring signals and 1 patent filing",
    --   "generation_params": {"temperature": 0.3, "max_tokens": 2000},
    --   "reviewed_by": "uuid",
    --   "reviewed_at": "2026-05-11T...",
    --   "review_status": "approved"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_narratives_org ON narratives (organization_id, created_at DESC);
CREATE INDEX idx_narratives_company ON narratives (company_id);
CREATE INDEX idx_narratives_type ON narratives (narrative_type);
```

---

## NLP Pipeline & Webhook Integration

```sql
-- NLP processing queue and results
CREATE TABLE nlp_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    article_id      UUID REFERENCES articles(id) ON DELETE CASCADE,
    job_type        VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    -- JSONB for job-specific config and results
    config          JSONB NOT NULL DEFAULT '{}',
    result          JSONB,
    -- Example result for NER job:
    -- {
    --   "entities_found": 5,
    --   "signals_created": 2,
    --   "processing_time_ms": 340,
    --   "model_version": "ner-v3.2",
    --   "entities": [
    --     {"type": "company", "value": "Acme Corp", "start": 45, "end": 54, "confidence": 0.96}
    --   ]
    -- }
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_nlp_jobs_status ON nlp_jobs (status, created_at)
    WHERE status IN ('pending', 'processing');

-- Webhook endpoints and delivery (Standard Webhooks compatible)
CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    secret_hash     VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- JSONB for flexible webhook configuration
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "event_types": ["signal.created", "battlecard.updated", "narrative.generated"],
    --   "filters": {"min_impact": "high", "categories": ["pricing"]},
    --   "retry_policy": {"max_retries": 5, "backoff_multiplier": 2},
    --   "headers": {"X-Custom-Header": "value"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_webhooks_org ON webhooks (organization_id);

-- Audit log (lightweight — not event-sourced, just operational tracking)
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    -- JSONB for action-specific details
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "changes": {"status": {"from": "draft", "to": "active"}},
    --   "ip_address": "192.168.1.1",
    --   "user_agent": "Mozilla/5.0..."
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_log_org ON audit_log (organization_id, created_at DESC);
CREATE INDEX idx_audit_log_resource ON audit_log (resource_type, resource_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organizations & Users | 2 | organizations, users |
| Companies | 1 | Single table with JSONB details (no separate aliases/relationships tables) |
| Sources | 2 | sources, source_companies |
| Articles & Signals | 2 | articles, signals (JSONB replaces entity/topic junction tables) |
| Battlecards & Win/Loss | 2 | battlecards, win_loss_records |
| Alerts & Narratives | 2 | alert_subscriptions, narratives |
| Boards | 1 | boards (items in JSONB array) |
| NLP & Integration | 2 | nlp_jobs, webhooks |
| Audit | 1 | audit_log |
| **Total** | **15** | ~50% fewer tables than normalized model |

---

## Key Design Decisions

1. **JSONB for variable/evolving data, relational for stable identifiers** — `organization_id`, `company_id`, `signal_type`, `signal_category`, `status`, and timestamps are relational columns with indexes. Everything that varies by signal type, by customer, or by NLP model version goes in JSONB. This gives you the best of both worlds: foreign key integrity for structural relationships, flexibility for content.

2. **No separate junction tables for entities and topics** — in the normalized model, `signal_entities` and `signal_topics` are separate tables requiring joins. Here, entities and topics are arrays inside `signals.payload`. This trades join-based querying for JSONB containment queries (`@>` operator), which are fast with GIN indexes and avoid the N+1 query problem.

3. **Battlecard sections as JSONB array, not separate table** — a battlecard's sections are always read and written together (you never query a section independently of its battlecard). Storing them as a JSONB array eliminates a table and a join, and the entire battlecard loads in a single query.

4. **Organization-level custom fields via `settings.custom_fields`** — each organization can define custom fields for companies (e.g., "market segment", "priority") without a separate custom fields infrastructure. These are stored in `companies.details.custom_fields` and validated at the application layer against the org's settings.

5. **Source configuration in JSONB** — different source types (RSS, website scraper, SEC EDGAR, GDELT) have entirely different configuration needs. Rather than a union of nullable columns or a table-per-type pattern, each source stores its config in JSONB. New source types are added without any schema change.

6. **Single `audit_log` table for operational auditing** — this is NOT event sourcing. It is a simple append-only log for compliance and debugging. For teams that need full event sourcing, this can be swapped for the event store from Suggestion 2.

7. **GIN indexes on all JSONB columns** — `jsonb_path_ops` GIN indexes enable fast containment queries. Specific expression indexes (e.g., `(details->>'industry')`) are added for the most common query patterns to avoid full GIN scans.

8. **`content_hash` for article deduplication** — same approach as the normalized model. SHA-256 hash enables fast dedup across sources.

9. **Schema validation at the application layer** — JSON Schema definitions for each JSONB structure are maintained in the application code (or in a `json_schemas` table) and enforced before writes. This provides type safety equivalent to relational constraints, but errors are caught at the application layer rather than the database layer.

10. **Board items as JSONB array** — intelligence boards have a small number of items (typically 10-50) that are always loaded together. Storing them as a JSONB array is simpler than a junction table and works well for drag-and-drop reordering (just update the array).

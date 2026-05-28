# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Market Intelligence Aggregator · Created: 2026-05-11

## Philosophy

This model follows classical third-normal-form (3NF) relational design, where every concept in the domain gets its own table, relationships are expressed through foreign keys and junction tables, and data integrity is enforced at the database level through constraints. The source of truth for every entity — companies, signals, sources, battlecards, users — is a dedicated table with well-defined columns and types.

This is the architecture used by most mature SaaS platforms when the domain is well-understood and query patterns are predictable. PostgreSQL's robust constraint system, materialized views, and full-text search make it a strong fit for a normalized CI platform where cross-entity reporting (e.g., "show me all pricing signals for companies in the fintech sector over the last 90 days") is a core use case.

The key advantage is **queryability and referential integrity**. Every relationship is explicit, every field is typed, and ad-hoc analytical queries work without decoding nested structures. The key cost is **rigidity** — adding new signal types or metadata fields requires schema migrations.

**Best for:** Teams building a well-scoped MVP with predictable signal types, strong reporting requirements, and a preference for data integrity over schema flexibility.

**Trade-offs:**
- (+) Full referential integrity — no orphaned records, no inconsistent relationships
- (+) Rich ad-hoc querying with standard SQL joins across any entity combination
- (+) Easy to reason about and debug — every concept has a clear home
- (+) Well-suited for BI tool integration (Metabase, Grafana, Looker)
- (-) Schema migrations required for every new signal type or metadata field
- (-) Junction tables proliferate — the table count is high (~35-45 tables)
- (-) Multi-jurisdiction or multi-type flexibility requires either many nullable columns or many subtypes
- (-) Write-heavy signal ingestion may bottleneck on foreign key constraint checks at scale

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RSS 2.0 / Atom 1.0 (RFC 4287) | `sources` table stores feed URLs with format type; `source_fetch_log` tracks crawl history |
| OPML 2.0 | Bulk import/export of source lists maps directly to `sources` table rows |
| Schema.org NewsArticle | Extracted metadata (headline, author, datePublished, publisher) maps to `articles` columns |
| ISO 8601 | All timestamp columns use `TIMESTAMPTZ` in ISO 8601 format |
| RFC 9309 (robots.txt) | `sources.robots_txt_status` tracks compliance posture per source |
| ISO 3166-1 | `companies.country_code` uses ISO 3166-1 alpha-2 codes |
| GDPR / CCPA | `persons` table includes `pii_consent_status` and `data_retention_expiry` columns |
| SEC EDGAR XBRL | `filing_signals` table stores structured filing data with XBRL taxonomy references |
| OpenAPI 3.1 / JSON Schema | API layer publishes OAS 3.1 spec; internal validation uses JSON Schema |
| OAuth 2.0 / OIDC | `users` and `api_tokens` tables support OAuth 2.0 and OIDC authentication flows |
| SCIM 2.0 | `users` table includes SCIM-compatible fields for enterprise provisioning |

---

## Entity Management

### Organizations & Tenancy

```sql
-- Multi-tenant organization (the customer using the platform)
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, team, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_organizations_slug ON organizations (slug);

-- Users within an organization
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- admin, member, viewer
    job_function    VARCHAR(100),  -- product_marketing, sales, strategy, executive
    oidc_subject    VARCHAR(255),  -- OpenID Connect subject identifier
    scim_external_id VARCHAR(255), -- SCIM provisioning identifier
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_users_org ON users (organization_id);
CREATE INDEX idx_users_email ON users (email);

-- API tokens for programmatic access
CREATE TABLE api_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE SET NULL,
    name            VARCHAR(255) NOT NULL,
    token_hash      VARCHAR(255) NOT NULL UNIQUE,  -- bcrypt hash of the token
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_api_tokens_org ON api_tokens (organization_id);
```

### Tracked Companies (Competitors & Market Entities)

```sql
-- Companies being tracked for competitive intelligence
CREATE TABLE companies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(500) NOT NULL,
    canonical_name  VARCHAR(500) NOT NULL,  -- normalized lowercase for dedup
    domain          VARCHAR(255),           -- primary website domain
    country_code    CHAR(2),               -- ISO 3166-1 alpha-2
    industry        VARCHAR(255),
    sub_industry    VARCHAR(255),
    employee_count_range VARCHAR(50),       -- 1-10, 11-50, 51-200, etc.
    funding_stage   VARCHAR(50),            -- seed, series_a, public, etc.
    stock_ticker    VARCHAR(20),
    sec_cik         VARCHAR(20),            -- SEC Central Index Key for EDGAR
    lei             VARCHAR(20),            -- Legal Entity Identifier (ISO 17442)
    logo_url        TEXT,
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, canonical_name)
);

CREATE INDEX idx_companies_org ON companies (organization_id);
CREATE INDEX idx_companies_domain ON companies (domain);
CREATE INDEX idx_companies_ticker ON companies (stock_ticker) WHERE stock_ticker IS NOT NULL;
CREATE INDEX idx_companies_cik ON companies (sec_cik) WHERE sec_cik IS NOT NULL;

-- Company aliases for NER matching (e.g., "MSFT", "Microsoft Corp", "Microsoft Corporation")
CREATE TABLE company_aliases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id      UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    alias           VARCHAR(500) NOT NULL,
    alias_type      VARCHAR(50) NOT NULL,  -- ticker, abbreviation, legal_name, trade_name
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_company_aliases_company ON company_aliases (company_id);
CREATE INDEX idx_company_aliases_alias ON company_aliases (lower(alias));

-- Relationship between companies (competitor, partner, subsidiary, acquirer)
CREATE TABLE company_relationships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    source_company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    target_company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    relationship_type VARCHAR(50) NOT NULL,  -- competitor, partner, subsidiary, acquirer, vendor
    confidence      NUMERIC(3,2) DEFAULT 1.0,  -- 0.00 to 1.00
    source          VARCHAR(50),  -- manual, ai_detected, imported
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, source_company_id, target_company_id, relationship_type)
);

CREATE INDEX idx_company_rels_org ON company_relationships (organization_id);
CREATE INDEX idx_company_rels_source ON company_relationships (source_company_id);
CREATE INDEX idx_company_rels_target ON company_relationships (target_company_id);
```

### Persons (Executives & Key Individuals)

```sql
-- Tracked individuals (executives, founders, analysts)
-- GDPR/CCPA: Only publicly available professional data; no private contact info
CREATE TABLE persons (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    full_name       VARCHAR(500) NOT NULL,
    canonical_name  VARCHAR(500) NOT NULL,
    current_title   VARCHAR(255),
    current_company_id UUID REFERENCES companies(id) ON DELETE SET NULL,
    linkedin_url    TEXT,
    pii_consent_status VARCHAR(50) DEFAULT 'public_professional',  -- public_professional, opted_out
    data_retention_expiry TIMESTAMPTZ,  -- GDPR: when to purge
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_persons_org ON persons (organization_id);
CREATE INDEX idx_persons_company ON persons (current_company_id) WHERE current_company_id IS NOT NULL;
```

---

## Source Management

```sql
-- Intelligence sources (RSS feeds, websites, APIs, etc.)
CREATE TABLE sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(500) NOT NULL,
    source_type     VARCHAR(50) NOT NULL,  -- rss, atom, website, api, sec_edgar, review_site, job_board
    url             TEXT NOT NULL,
    feed_format     VARCHAR(20),           -- rss_2_0, atom_1_0, null for non-feed sources
    crawl_frequency_minutes INT NOT NULL DEFAULT 60,
    robots_txt_status VARCHAR(50) DEFAULT 'unchecked',  -- compliant, blocked, unchecked
    robots_txt_checked_at TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_fetched_at TIMESTAMPTZ,
    last_successful_at TIMESTAMPTZ,
    consecutive_failures INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sources_org ON sources (organization_id);
CREATE INDEX idx_sources_type ON sources (source_type);
CREATE INDEX idx_sources_active ON sources (is_active, last_fetched_at) WHERE is_active = TRUE;

-- Log of every source fetch attempt (for debugging and compliance)
CREATE TABLE source_fetch_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    fetched_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status          VARCHAR(50) NOT NULL,  -- success, error, rate_limited, robots_blocked
    http_status     INT,
    items_found     INT DEFAULT 0,
    items_new       INT DEFAULT 0,
    error_message   TEXT,
    duration_ms     INT
);

CREATE INDEX idx_fetch_log_source ON source_fetch_log (source_id, fetched_at DESC);

-- Link sources to specific companies they cover
CREATE TABLE source_company_coverage (
    source_id       UUID NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    company_id      UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    PRIMARY KEY (source_id, company_id)
);
```

---

## Signal Ingestion & Storage

```sql
-- Raw articles/content ingested from sources
CREATE TABLE articles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    external_id     VARCHAR(500),          -- guid from RSS, URL hash, etc.
    url             TEXT NOT NULL,
    title           TEXT NOT NULL,
    summary         TEXT,
    full_text       TEXT,
    author          VARCHAR(500),
    publisher       VARCHAR(500),
    published_at    TIMESTAMPTZ,
    fetched_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    language        VARCHAR(10) DEFAULT 'en',  -- ISO 639-1
    content_hash    VARCHAR(64),           -- SHA-256 for dedup
    is_duplicate    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_articles_org ON articles (organization_id);
CREATE INDEX idx_articles_source ON articles (source_id);
CREATE INDEX idx_articles_published ON articles (organization_id, published_at DESC);
CREATE INDEX idx_articles_hash ON articles (content_hash) WHERE content_hash IS NOT NULL;
CREATE UNIQUE INDEX idx_articles_external_id ON articles (source_id, external_id) WHERE external_id IS NOT NULL;

-- Full-text search index
CREATE INDEX idx_articles_fts ON articles USING GIN (
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(summary, '') || ' ' || coalesce(full_text, ''))
);

-- Competitive signals extracted from articles via NLP
-- Each article can produce multiple signals (e.g., pricing change + hiring signal)
CREATE TABLE signals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    article_id      UUID REFERENCES articles(id) ON DELETE SET NULL,
    company_id      UUID REFERENCES companies(id) ON DELETE SET NULL,
    signal_type     VARCHAR(100) NOT NULL,  -- See signal_types reference table
    signal_category VARCHAR(100) NOT NULL,  -- product, pricing, talent, regulatory, funding, partnership
    title           TEXT NOT NULL,
    summary         TEXT,
    sentiment       VARCHAR(20),            -- positive, negative, neutral, mixed
    sentiment_score NUMERIC(4,3),           -- -1.000 to +1.000
    confidence      NUMERIC(3,2) NOT NULL DEFAULT 0.50,  -- NLP confidence 0.00 to 1.00
    relevance_score NUMERIC(3,2),           -- org-specific relevance 0.00 to 1.00
    impact_level    VARCHAR(20),            -- low, medium, high, critical
    source_url      TEXT,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,            -- optional TTL for time-sensitive signals
    is_dismissed    BOOLEAN NOT NULL DEFAULT FALSE,
    dismissed_by    UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_signals_org ON signals (organization_id);
CREATE INDEX idx_signals_company ON signals (company_id);
CREATE INDEX idx_signals_type ON signals (signal_type);
CREATE INDEX idx_signals_category ON signals (signal_category);
CREATE INDEX idx_signals_detected ON signals (organization_id, detected_at DESC);
CREATE INDEX idx_signals_relevance ON signals (organization_id, relevance_score DESC NULLS LAST)
    WHERE is_dismissed = FALSE;

-- Reference table for signal types
CREATE TABLE signal_types (
    id              VARCHAR(100) PRIMARY KEY,  -- e.g., 'pricing_change', 'executive_hire'
    category        VARCHAR(100) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    description     TEXT,
    default_impact  VARCHAR(20) DEFAULT 'medium',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE
);

-- NLP-extracted entities linked to signals
CREATE TABLE signal_entities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    signal_id       UUID NOT NULL REFERENCES signals(id) ON DELETE CASCADE,
    entity_type     VARCHAR(50) NOT NULL,  -- company, person, product, location, amount, date
    entity_value    VARCHAR(500) NOT NULL,
    entity_normalized VARCHAR(500),        -- canonical form after normalization
    company_id      UUID REFERENCES companies(id) ON DELETE SET NULL,  -- resolved company if entity_type=company
    person_id       UUID REFERENCES persons(id) ON DELETE SET NULL,    -- resolved person if entity_type=person
    start_offset    INT,                   -- character offset in source text
    end_offset      INT,
    confidence      NUMERIC(3,2)
);

CREATE INDEX idx_signal_entities_signal ON signal_entities (signal_id);
CREATE INDEX idx_signal_entities_company ON signal_entities (company_id) WHERE company_id IS NOT NULL;
CREATE INDEX idx_signal_entities_type ON signal_entities (entity_type, entity_normalized);

-- Topic/tag assignments for signals (many-to-many)
CREATE TABLE signal_topics (
    signal_id       UUID NOT NULL REFERENCES signals(id) ON DELETE CASCADE,
    topic_id        UUID NOT NULL REFERENCES topics(id) ON DELETE CASCADE,
    confidence      NUMERIC(3,2) DEFAULT 1.0,
    PRIMARY KEY (signal_id, topic_id)
);

-- Topics / taxonomy
CREATE TABLE topics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    parent_topic_id UUID REFERENCES topics(id) ON DELETE SET NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_topics_org ON topics (organization_id);
CREATE INDEX idx_topics_parent ON topics (parent_topic_id) WHERE parent_topic_id IS NOT NULL;
```

---

## SEC EDGAR Filing Signals

```sql
-- Structured data from SEC EDGAR XBRL filings
CREATE TABLE filing_signals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    signal_id       UUID NOT NULL REFERENCES signals(id) ON DELETE CASCADE,
    company_id      UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    filing_type     VARCHAR(20) NOT NULL,   -- 10-K, 10-Q, 8-K, DEF14A
    filing_date     DATE NOT NULL,
    accession_number VARCHAR(30) NOT NULL,  -- SEC accession number
    sec_url         TEXT NOT NULL,
    xbrl_facts      JSONB,                  -- structured XBRL data points extracted
    -- Example xbrl_facts:
    -- {
    --   "revenue": {"value": 50000000, "unit": "USD", "period": "2025-Q4"},
    --   "net_income": {"value": 5000000, "unit": "USD", "period": "2025-Q4"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_filing_signals_company ON filing_signals (company_id, filing_date DESC);
CREATE INDEX idx_filing_signals_type ON filing_signals (filing_type);
```

---

## Battlecards & Intelligence Products

```sql
-- Competitive battlecards
CREATE TABLE battlecards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    competitor_id   UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',  -- draft, in_review, active, archived
    approved_by     UUID REFERENCES users(id) ON DELETE SET NULL,
    approved_at     TIMESTAMPTZ,
    last_refreshed_at TIMESTAMPTZ,
    version         INT NOT NULL DEFAULT 1,
    created_by      UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_battlecards_org ON battlecards (organization_id);
CREATE INDEX idx_battlecards_competitor ON battlecards (competitor_id);
CREATE INDEX idx_battlecards_status ON battlecards (organization_id, status);

-- Battlecard sections (modular content blocks)
CREATE TABLE battlecard_sections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    battlecard_id   UUID NOT NULL REFERENCES battlecards(id) ON DELETE CASCADE,
    section_type    VARCHAR(50) NOT NULL,  -- overview, strengths, weaknesses, objections, proof_points, pricing, talk_track
    title           VARCHAR(255) NOT NULL,
    content         TEXT NOT NULL,          -- markdown content
    sort_order      INT NOT NULL DEFAULT 0,
    source_signal_ids UUID[],              -- signals that informed this section
    ai_generated    BOOLEAN NOT NULL DEFAULT FALSE,
    last_verified_at TIMESTAMPTZ,
    verified_by     UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_battlecard_sections_card ON battlecard_sections (battlecard_id, sort_order);

-- Win/loss records linked to battlecards
CREATE TABLE win_loss_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    competitor_id   UUID REFERENCES companies(id) ON DELETE SET NULL,
    outcome         VARCHAR(20) NOT NULL,  -- win, loss, no_decision
    deal_value      NUMERIC(15,2),
    deal_currency   CHAR(3) DEFAULT 'USD', -- ISO 4217
    loss_reason     VARCHAR(255),
    win_reason      VARCHAR(255),
    notes           TEXT,
    interview_transcript TEXT,
    deal_date       DATE,
    recorded_by     UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_win_loss_org ON win_loss_records (organization_id);
CREATE INDEX idx_win_loss_competitor ON win_loss_records (competitor_id);
CREATE INDEX idx_win_loss_outcome ON win_loss_records (organization_id, outcome, deal_date DESC);
```

---

## Alert & Delivery System

```sql
-- Alert subscriptions (what users want to be notified about)
CREATE TABLE alert_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    filter_companies UUID[],               -- null = all companies
    filter_signal_types VARCHAR(100)[],     -- null = all types
    filter_categories VARCHAR(100)[],       -- null = all categories
    filter_min_relevance NUMERIC(3,2),
    filter_min_impact VARCHAR(20),          -- low, medium, high, critical
    delivery_channel VARCHAR(50) NOT NULL,  -- email, slack, webhook, in_app
    delivery_frequency VARCHAR(50) NOT NULL DEFAULT 'realtime', -- realtime, daily_digest, weekly_digest
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_alert_subs_user ON alert_subscriptions (user_id);
CREATE INDEX idx_alert_subs_org ON alert_subscriptions (organization_id);

-- Delivered alerts / notification log
CREATE TABLE alert_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES alert_subscriptions(id) ON DELETE CASCADE,
    signal_id       UUID REFERENCES signals(id) ON DELETE SET NULL,
    delivered_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    delivery_channel VARCHAR(50) NOT NULL,
    delivery_status VARCHAR(50) NOT NULL,  -- sent, delivered, failed, opened
    opened_at       TIMESTAMPTZ,
    clicked_at      TIMESTAMPTZ
);

CREATE INDEX idx_alert_deliveries_sub ON alert_deliveries (subscription_id, delivered_at DESC);
CREATE INDEX idx_alert_deliveries_signal ON alert_deliveries (signal_id);

-- AI-generated intelligence digests / newsletters
CREATE TABLE digests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    summary         TEXT NOT NULL,          -- AI-generated executive summary
    content_html    TEXT NOT NULL,          -- rendered digest content
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    signal_count    INT NOT NULL DEFAULT 0,
    created_by      UUID REFERENCES users(id) ON DELETE SET NULL,  -- null = auto-generated
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_digests_org ON digests (organization_id, period_end DESC);

-- Signals included in a digest
CREATE TABLE digest_signals (
    digest_id       UUID NOT NULL REFERENCES digests(id) ON DELETE CASCADE,
    signal_id       UUID NOT NULL REFERENCES signals(id) ON DELETE CASCADE,
    sort_order      INT NOT NULL DEFAULT 0,
    PRIMARY KEY (digest_id, signal_id)
);
```

---

## Intelligence Boards & Collaboration

```sql
-- Shared intelligence boards (like Kanban boards for CI)
CREATE TABLE boards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    visibility      VARCHAR(50) NOT NULL DEFAULT 'team',  -- private, team, organization
    created_by      UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_boards_org ON boards (organization_id);

-- Items pinned to a board
CREATE TABLE board_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    board_id        UUID NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
    signal_id       UUID REFERENCES signals(id) ON DELETE SET NULL,
    article_id      UUID REFERENCES articles(id) ON DELETE SET NULL,
    note            TEXT,                   -- analyst annotation
    added_by        UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_board_items_board ON board_items (board_id, sort_order);

-- Comments on signals or board items
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    signal_id       UUID REFERENCES signals(id) ON DELETE CASCADE,
    board_item_id   UUID REFERENCES board_items(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    content         TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (signal_id IS NOT NULL OR board_item_id IS NOT NULL)
);

CREATE INDEX idx_comments_signal ON comments (signal_id) WHERE signal_id IS NOT NULL;
CREATE INDEX idx_comments_board_item ON comments (board_item_id) WHERE board_item_id IS NOT NULL;
```

---

## AI Processing & Narrative Generation

```sql
-- AI-generated competitive narratives
CREATE TABLE narratives (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    company_id      UUID REFERENCES companies(id) ON DELETE SET NULL,
    narrative_type  VARCHAR(50) NOT NULL,  -- threat_assessment, pivot_detection, market_entry, trend
    title           TEXT NOT NULL,
    content         TEXT NOT NULL,         -- AI-generated narrative text
    evidence_summary TEXT,                 -- structured summary of supporting signals
    signal_ids      UUID[] NOT NULL,       -- array of signal IDs that informed this narrative
    model_version   VARCHAR(100),          -- LLM model used for generation
    confidence      NUMERIC(3,2),
    reviewed_by     UUID REFERENCES users(id) ON DELETE SET NULL,
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_narratives_org ON narratives (organization_id);
CREATE INDEX idx_narratives_company ON narratives (company_id);
CREATE INDEX idx_narratives_type ON narratives (narrative_type);
CREATE INDEX idx_narratives_created ON narratives (organization_id, created_at DESC);

-- NLP processing jobs tracking
CREATE TABLE nlp_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    article_id      UUID REFERENCES articles(id) ON DELETE CASCADE,
    job_type        VARCHAR(50) NOT NULL,  -- ner, sentiment, classification, summarization, narrative
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',  -- pending, processing, completed, failed
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    result_metadata JSONB,                 -- processing stats and diagnostics
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_nlp_jobs_status ON nlp_jobs (status, created_at) WHERE status IN ('pending', 'processing');
CREATE INDEX idx_nlp_jobs_article ON nlp_jobs (article_id);
```

---

## Webhook & MCP Integration

```sql
-- Outbound webhook configurations (Standard Webhooks spec)
CREATE TABLE webhook_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    secret_hash     VARCHAR(255) NOT NULL,  -- for Standard Webhooks signature verification
    event_types     VARCHAR(100)[] NOT NULL, -- signal.created, battlecard.updated, digest.generated
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_webhook_endpoints_org ON webhook_endpoints (organization_id);

-- Webhook delivery log
CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id     UUID NOT NULL REFERENCES webhook_endpoints(id) ON DELETE CASCADE,
    event_type      VARCHAR(100) NOT NULL,
    payload_hash    VARCHAR(64) NOT NULL,
    http_status     INT,
    attempt         INT NOT NULL DEFAULT 1,
    delivered_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    response_body   TEXT
);

CREATE INDEX idx_webhook_deliveries_endpoint ON webhook_deliveries (endpoint_id, delivered_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organizations & Users | 3 | organizations, users, api_tokens |
| Companies & Entities | 4 | companies, company_aliases, company_relationships, persons |
| Source Management | 3 | sources, source_fetch_log, source_company_coverage |
| Signal Ingestion | 6 | articles, signals, signal_types, signal_entities, signal_topics, topics |
| SEC EDGAR | 1 | filing_signals |
| Battlecards & Win/Loss | 3 | battlecards, battlecard_sections, win_loss_records |
| Alerts & Delivery | 4 | alert_subscriptions, alert_deliveries, digests, digest_signals |
| Boards & Collaboration | 3 | boards, board_items, comments |
| AI / Narratives | 2 | narratives, nlp_jobs |
| Webhooks / Integration | 2 | webhook_endpoints, webhook_deliveries |
| **Total** | **31** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation without coordination, supports future sharding, and prevents enumeration attacks on the API.

2. **`organization_id` on every content table** — enables row-level security (RLS) for multi-tenancy; queries always filter by organization, and a PostgreSQL RLS policy can enforce this automatically.

3. **Separate `articles` and `signals` tables** — articles are raw ingested content; signals are extracted intelligence. One article can produce multiple signals. This separation keeps the NLP pipeline clean and allows re-processing articles without losing curated signal metadata.

4. **`signal_types` as a reference table** — extensible taxonomy of signal types without altering the `signals` table schema. New signal types are added as rows, not columns.

5. **Modular battlecard sections** — instead of a monolithic content blob, each battlecard is composed of discrete sections (overview, strengths, objections, etc.) that can be individually sourced, verified, and refreshed by AI.

6. **Full-text search via `tsvector` GIN index** — PostgreSQL's built-in FTS provides adequate search for MVP without requiring Elasticsearch. Can be migrated to Elasticsearch later for advanced features.

7. **`content_hash` for deduplication** — SHA-256 hash of article content enables fast dedup across sources without full-text comparison.

8. **GDPR-aware `persons` table** — includes consent status and retention expiry fields. Only stores publicly available professional data (titles, company affiliations), never private contact information.

9. **Standard Webhooks for outbound integration** — webhook endpoints store a secret hash for HMAC signature verification per the Standard Webhooks specification, ensuring payload integrity for downstream consumers.

10. **`robots_txt_status` tracking on sources** — compliance with RFC 9309 is tracked per source and logged, providing an auditable compliance posture for the crawling pipeline.

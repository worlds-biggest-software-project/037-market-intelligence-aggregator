# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Market Intelligence Aggregator · Created: 2026-05-11

## Philosophy

This model treats every piece of intelligence as an **immutable event** in an append-only event store. The event store is the single source of truth — there are no UPDATE or DELETE operations on signal data. All read-facing views (competitor dashboards, battlecards, digest feeds) are **materialised projections** rebuilt from events using the CQRS (Command Query Responsibility Segregation) pattern.

This architecture is inspired by financial ledger systems, fraud detection platforms, and the OCF (Open Cap Table Format) event-based approach where every state change is captured as a discrete, timestamped event. In a market intelligence context, this means every signal detection, every battlecard edit, every analyst annotation, and every dismissal is recorded permanently. You can always answer "what did we know about Competitor X on March 15th?" by replaying events up to that date.

The key advantage is **complete auditability and temporal querying**. Every state the system ever held can be reconstructed. The key cost is **complexity** — you need a projection layer to build read models, and developers must think in terms of events rather than entities.

**Best for:** Organizations requiring full audit trails, regulatory compliance, temporal intelligence analysis ("what changed since last quarter?"), and teams planning to build AI analytics over historical signal patterns.

**Trade-offs:**
- (+) Complete, immutable audit trail — every signal, annotation, and decision is permanently recorded
- (+) Temporal queries are native — "what did we know on date X?" is a simple event replay
- (+) Event replay enables A/B testing of NLP models against historical data
- (+) Natural fit for AI training — historical signal patterns are preserved for model training
- (+) Write performance is excellent — append-only inserts, no lock contention
- (-) Read queries require materialised views — more infrastructure to maintain
- (-) Event store grows unboundedly; requires archival/compaction strategy
- (-) Eventual consistency between event store and read models
- (-) Higher cognitive complexity for developers — events vs. entities mental model
- (-) Schema evolution for events requires careful versioning

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RSS 2.0 / Atom 1.0 | Source events capture feed metadata; article ingestion events preserve original RSS/Atom fields |
| Schema.org NewsArticle | Article ingestion events include Schema.org-extracted metadata as event payload fields |
| ISO 8601 | All event timestamps use ISO 8601 `TIMESTAMPTZ`; `event_time` is the canonical ordering field |
| RFC 9309 (robots.txt) | Compliance events record robots.txt checks and policy decisions |
| GDPR / CCPA | Data subject events (consent changes, erasure requests) are first-class events; replay respects erasure markers |
| SEC EDGAR XBRL | Filing events capture structured XBRL facts as immutable event payloads |
| OCF (Open Cap Format) | Event-based architecture pattern borrowed from OCF's approach to cap table lifecycle events |
| JSON Schema (Draft 2020-12) | Every event type has a registered JSON Schema for payload validation |
| Standard Webhooks | Outbound event delivery uses Standard Webhooks envelope format |

---

## Core Event Store

```sql
-- The append-only event store — THE source of truth for all intelligence data
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    stream_id       UUID NOT NULL,         -- groups events belonging to the same entity/aggregate
    stream_type     VARCHAR(100) NOT NULL,  -- company, signal, battlecard, source, user, narrative
    event_type      VARCHAR(200) NOT NULL,  -- e.g., signal.detected, battlecard.section_updated
    event_version   INT NOT NULL DEFAULT 1, -- schema version for this event type
    sequence_number BIGINT NOT NULL,        -- monotonically increasing per stream
    event_time      TIMESTAMPTZ NOT NULL DEFAULT NOW(),  -- when the event occurred in the real world
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),  -- when we recorded it (may differ)
    actor_id        UUID,                  -- user or system that caused the event
    actor_type      VARCHAR(50) NOT NULL DEFAULT 'system',  -- user, system, ai, api
    payload         JSONB NOT NULL,         -- event-type-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation IDs, causation chain, processing info
    UNIQUE (stream_id, sequence_number)
);

-- Primary query pattern: replay events for a stream in order
CREATE INDEX idx_events_stream ON events (stream_id, sequence_number);

-- Query by organization and time range (for projections and analytics)
CREATE INDEX idx_events_org_time ON events (organization_id, event_time DESC);

-- Query by event type (for building specific projections)
CREATE INDEX idx_events_type ON events (event_type, event_time DESC);

-- Query by stream type within an organization
CREATE INDEX idx_events_stream_type ON events (organization_id, stream_type, event_time DESC);

-- JSONB index for payload queries (e.g., find all events mentioning a company)
CREATE INDEX idx_events_payload ON events USING GIN (payload jsonb_path_ops);

-- Partition by month for performance and archival
-- In production, use PARTITION BY RANGE (event_time)
-- CREATE TABLE events_2026_05 PARTITION OF events FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

### Event Type Registry

```sql
-- Registry of all known event types with their JSON Schema definitions
CREATE TABLE event_type_registry (
    event_type      VARCHAR(200) PRIMARY KEY,
    stream_type     VARCHAR(100) NOT NULL,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,        -- JSON Schema for validating event payloads
    current_version INT NOT NULL DEFAULT 1,
    deprecated      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Example event types and their payload schemas:
--
-- signal.detected:
-- {
--   "article_id": "uuid",
--   "company_id": "uuid",
--   "signal_type": "pricing_change",
--   "signal_category": "pricing",
--   "title": "Competitor X reduces Enterprise tier by 20%",
--   "summary": "...",
--   "sentiment": "negative",
--   "sentiment_score": -0.75,
--   "confidence": 0.92,
--   "source_url": "https://...",
--   "entities": [
--     {"type": "company", "value": "Competitor X", "resolved_id": "uuid"},
--     {"type": "amount", "value": "20%", "context": "price_reduction"}
--   ]
-- }
--
-- signal.dismissed:
-- {
--   "reason": "not_relevant",
--   "comment": "This is about a different product line"
-- }
--
-- signal.relevance_scored:
-- {
--   "relevance_score": 0.87,
--   "model_version": "relevance-v3",
--   "factors": {"recency": 0.9, "topic_match": 0.85, "company_priority": 0.95}
-- }
--
-- battlecard.created:
-- {
--   "competitor_id": "uuid",
--   "title": "Competitor X Battlecard"
-- }
--
-- battlecard.section_updated:
-- {
--   "section_type": "objections",
--   "title": "Common Objections",
--   "content": "...",
--   "ai_generated": true,
--   "source_signal_ids": ["uuid1", "uuid2"],
--   "model_version": "claude-4-sonnet"
-- }
--
-- narrative.generated:
-- {
--   "company_id": "uuid",
--   "narrative_type": "pivot_detection",
--   "title": "Competitor X pivoting to AI-first pricing",
--   "content": "...",
--   "evidence_signals": ["uuid1", "uuid2", "uuid3"],
--   "model_version": "claude-4-opus",
--   "confidence": 0.85
-- }
--
-- source.fetched:
-- {
--   "source_id": "uuid",
--   "status": "success",
--   "http_status": 200,
--   "items_found": 15,
--   "items_new": 3,
--   "duration_ms": 1250
-- }
--
-- article.ingested:
-- {
--   "source_id": "uuid",
--   "url": "https://...",
--   "title": "...",
--   "summary": "...",
--   "author": "...",
--   "published_at": "2026-05-11T10:30:00Z",
--   "content_hash": "sha256:...",
--   "language": "en"
-- }
--
-- gdpr.erasure_requested:
-- {
--   "subject_type": "person",
--   "subject_id": "uuid",
--   "reason": "data_subject_request",
--   "requested_by": "uuid"
-- }
```

---

## Materialised Read Models (Projections)

These tables are **derived** from the event store and can be rebuilt at any time by replaying events. They exist purely for read performance.

### Company Projection

```sql
-- Materialised view of current company state (rebuilt from events)
CREATE TABLE rm_companies (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            VARCHAR(500) NOT NULL,
    canonical_name  VARCHAR(500) NOT NULL,
    domain          VARCHAR(255),
    country_code    CHAR(2),
    industry        VARCHAR(255),
    sub_industry    VARCHAR(255),
    employee_count_range VARCHAR(50),
    funding_stage   VARCHAR(50),
    stock_ticker    VARCHAR(20),
    sec_cik         VARCHAR(20),
    lei             VARCHAR(20),
    logo_url        TEXT,
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Denormalized signal counts for dashboard display
    total_signal_count INT NOT NULL DEFAULT 0,
    signal_count_7d INT NOT NULL DEFAULT 0,
    signal_count_30d INT NOT NULL DEFAULT 0,
    latest_signal_at TIMESTAMPTZ,
    avg_sentiment_30d NUMERIC(4,3),
    -- Projection metadata
    last_event_id   UUID,                  -- last event applied to this projection
    last_event_time TIMESTAMPTZ,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_companies_org ON rm_companies (organization_id);
CREATE INDEX idx_rm_companies_domain ON rm_companies (domain);
CREATE INDEX idx_rm_companies_signals ON rm_companies (organization_id, total_signal_count DESC);
```

### Signal Feed Projection

```sql
-- Materialised view of current signal state (detected, enriched, dismissed)
CREATE TABLE rm_signals (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    article_id      UUID,
    company_id      UUID,
    signal_type     VARCHAR(100) NOT NULL,
    signal_category VARCHAR(100) NOT NULL,
    title           TEXT NOT NULL,
    summary         TEXT,
    sentiment       VARCHAR(20),
    sentiment_score NUMERIC(4,3),
    confidence      NUMERIC(3,2) NOT NULL,
    relevance_score NUMERIC(3,2),
    impact_level    VARCHAR(20),
    source_url      TEXT,
    detected_at     TIMESTAMPTZ NOT NULL,
    published_at    TIMESTAMPTZ,
    is_dismissed    BOOLEAN NOT NULL DEFAULT FALSE,
    dismissed_by    UUID,
    dismissed_at    TIMESTAMPTZ,
    dismiss_reason  VARCHAR(255),
    -- Denormalized entity data for display
    entities        JSONB NOT NULL DEFAULT '[]',
    topics          JSONB NOT NULL DEFAULT '[]',
    -- Projection metadata
    last_event_id   UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_signals_org ON rm_signals (organization_id, detected_at DESC);
CREATE INDEX idx_rm_signals_company ON rm_signals (company_id, detected_at DESC);
CREATE INDEX idx_rm_signals_type ON rm_signals (signal_type);
CREATE INDEX idx_rm_signals_relevance ON rm_signals (organization_id, relevance_score DESC NULLS LAST)
    WHERE is_dismissed = FALSE;

-- Full-text search on the projection
CREATE INDEX idx_rm_signals_fts ON rm_signals USING GIN (
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(summary, ''))
);
```

### Battlecard Projection

```sql
-- Materialised view of current battlecard state
CREATE TABLE rm_battlecards (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    competitor_id   UUID NOT NULL,
    title           VARCHAR(500) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    version         INT NOT NULL DEFAULT 1,
    -- Denormalized sections for single-query retrieval
    sections        JSONB NOT NULL DEFAULT '[]',
    -- Example sections JSONB:
    -- [
    --   {
    --     "section_type": "overview",
    --     "title": "Company Overview",
    --     "content": "...",
    --     "ai_generated": false,
    --     "last_verified_at": "2026-05-10T...",
    --     "source_signal_count": 5
    --   },
    --   ...
    -- ]
    approved_by     UUID,
    approved_at     TIMESTAMPTZ,
    last_refreshed_at TIMESTAMPTZ,
    created_by      UUID,
    created_at      TIMESTAMPTZ,
    -- Projection metadata
    last_event_id   UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_battlecards_org ON rm_battlecards (organization_id);
CREATE INDEX idx_rm_battlecards_competitor ON rm_battlecards (competitor_id);
```

### Source Health Projection

```sql
-- Materialised view of source health and fetch status
CREATE TABLE rm_sources (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            VARCHAR(500) NOT NULL,
    source_type     VARCHAR(50) NOT NULL,
    url             TEXT NOT NULL,
    feed_format     VARCHAR(20),
    crawl_frequency_minutes INT NOT NULL DEFAULT 60,
    robots_txt_status VARCHAR(50) DEFAULT 'unchecked',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_fetched_at TIMESTAMPTZ,
    last_successful_at TIMESTAMPTZ,
    consecutive_failures INT NOT NULL DEFAULT 0,
    total_articles_ingested INT NOT NULL DEFAULT 0,
    total_signals_generated INT NOT NULL DEFAULT 0,
    -- Projection metadata
    last_event_id   UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_sources_org ON rm_sources (organization_id);
CREATE INDEX idx_rm_sources_active ON rm_sources (is_active, last_fetched_at) WHERE is_active = TRUE;
```

---

## Command-Side Tables (Write Path)

These tables support the write path — they are NOT the source of truth but provide lookup and validation during command processing.

```sql
-- Organization and user lookup (populated by events, used for command validation)
CREATE TABLE cmd_organizations (
    id              UUID PRIMARY KEY,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE cmd_users (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES cmd_organizations(id),
    email           VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    UNIQUE (organization_id, email)
);

-- Stream position tracking (for projection rebuilds)
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Idempotency keys to prevent duplicate command processing
CREATE TABLE idempotency_keys (
    key             VARCHAR(255) PRIMARY KEY,
    organization_id UUID NOT NULL,
    result_event_id UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '24 hours'
);

CREATE INDEX idx_idempotency_expires ON idempotency_keys (expires_at);
```

---

## Temporal Query Examples

```sql
-- "What did we know about Competitor X on March 15th, 2026?"
-- Replay all signal.detected events for that company up to the target date
SELECT
    e.event_id,
    e.event_time,
    e.payload->>'title' AS signal_title,
    e.payload->>'signal_type' AS signal_type,
    e.payload->>'sentiment' AS sentiment,
    e.payload->>'confidence' AS confidence
FROM events e
WHERE e.organization_id = '...'
  AND e.stream_type = 'signal'
  AND e.event_type = 'signal.detected'
  AND e.event_time <= '2026-03-15T23:59:59Z'
  AND e.payload->>'company_id' = '...'
ORDER BY e.event_time DESC;

-- "How has sentiment about Competitor X changed over the last 6 months?"
SELECT
    date_trunc('week', e.event_time) AS week,
    AVG((e.payload->>'sentiment_score')::numeric) AS avg_sentiment,
    COUNT(*) AS signal_count
FROM events e
WHERE e.organization_id = '...'
  AND e.event_type = 'signal.detected'
  AND e.payload->>'company_id' = '...'
  AND e.event_time >= NOW() - INTERVAL '6 months'
GROUP BY date_trunc('week', e.event_time)
ORDER BY week;

-- "What battlecard changes were made in the last 30 days?"
SELECT
    e.event_id,
    e.event_time,
    e.event_type,
    e.actor_id,
    e.payload->>'section_type' AS section,
    e.payload->>'ai_generated' AS ai_generated
FROM events e
WHERE e.organization_id = '...'
  AND e.stream_type = 'battlecard'
  AND e.event_type LIKE 'battlecard.%'
  AND e.event_time >= NOW() - INTERVAL '30 days'
ORDER BY e.event_time DESC;

-- "Rebuild the signal feed projection from scratch"
-- (Run this as a batch job when projections drift or after schema changes)
TRUNCATE rm_signals;
INSERT INTO rm_signals (id, organization_id, article_id, company_id, signal_type, ...)
SELECT
    e.stream_id AS id,
    e.organization_id,
    e.payload->>'article_id' AS article_id,
    e.payload->>'company_id' AS company_id,
    e.payload->>'signal_type' AS signal_type,
    -- ... aggregate latest state from events
FROM events e
WHERE e.stream_type = 'signal'
  AND e.event_type = 'signal.detected'
ORDER BY e.stream_id, e.sequence_number;
-- Then apply signal.dismissed, signal.relevance_scored, etc.
```

---

## Event Processing Pipeline

```sql
-- Subscription positions for event consumers (projection builders, webhook dispatchers, etc.)
CREATE TABLE event_subscriptions (
    subscription_id VARCHAR(100) PRIMARY KEY,  -- e.g., 'projection:rm_signals', 'webhook:org123'
    last_event_id   UUID,
    last_event_time TIMESTAMPTZ,
    consumer_group  VARCHAR(100),              -- for parallel processing
    status          VARCHAR(50) NOT NULL DEFAULT 'active',  -- active, paused, rebuilding
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Dead letter queue for failed event processing
CREATE TABLE event_dead_letters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,
    subscription_id VARCHAR(100) NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INT NOT NULL DEFAULT 0,
    max_retries     INT NOT NULL DEFAULT 5,
    next_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_dead_letters_retry ON event_dead_letters (next_retry_at)
    WHERE retry_count < max_retries;
```

---

## GDPR Event Handling

```sql
-- GDPR erasure is handled via events, not DELETE operations
-- When a gdpr.erasure_requested event is recorded:
-- 1. A gdpr.erasure_executed event marks the erasure as processed
-- 2. Projections rebuild, filtering out the erased subject
-- 3. The original events are retained but with PII fields replaced by '[REDACTED]'

-- Example: Redact PII from historical events for a given person
-- (Run as a batch job triggered by gdpr.erasure_requested events)
--
-- UPDATE events
-- SET payload = payload || '{"person_name": "[REDACTED]", "person_title": "[REDACTED]"}'
-- WHERE stream_type = 'signal'
--   AND payload->>'person_id' = '<erased-person-uuid>';
--
-- Note: This is the ONE case where events are mutated. The mutation is itself
-- recorded as a gdpr.erasure_executed event for audit purposes.
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events, event_type_registry |
| Read Models (Projections) | 4 | rm_companies, rm_signals, rm_battlecards, rm_sources |
| Command-Side Lookup | 4 | cmd_organizations, cmd_users, projection_checkpoints, idempotency_keys |
| Event Processing | 2 | event_subscriptions, event_dead_letters |
| **Total** | **12** | Much fewer tables; complexity is in event processing |

---

## Key Design Decisions

1. **Single `events` table as the source of truth** — all state changes across the entire system (signals, battlecards, sources, users) are stored in one append-only table. This dramatically simplifies backup, replication, and disaster recovery — replicate one table and you have the entire system.

2. **`stream_id` + `sequence_number` for aggregate consistency** — each entity (company, signal, battlecard) has a unique stream. The sequence number ensures events within a stream are ordered and that optimistic concurrency control can detect conflicts.

3. **Typed event payloads validated by JSON Schema** — the `event_type_registry` table stores JSON Schema definitions for every event type. This provides the same type safety as relational columns but with the flexibility to evolve event schemas by incrementing `event_version`.

4. **Read models are disposable** — every `rm_*` table can be dropped and rebuilt from the event store. This means read model schema changes are zero-risk: add a new column, rebuild, done. No data migration needed.

5. **Denormalized read models for performance** — the `rm_battlecards` table includes sections as a JSONB array, and `rm_companies` includes signal count rollups. This avoids joins on the read path entirely.

6. **Monthly event partitioning** — the events table should be partitioned by `event_time` for performance and archival. Old partitions can be moved to cold storage (S3/GCS) while remaining queryable via foreign data wrappers.

7. **GDPR erasure via event redaction** — rather than deleting events (which would break the event store contract), PII is redacted in-place. The redaction itself is recorded as an event, maintaining the audit trail even for erasure operations.

8. **Idempotency keys for exactly-once command processing** — prevents duplicate signal detection or battlecard updates when the NLP pipeline retries failed jobs.

9. **Event subscriptions with dead letter queue** — projection builders and webhook dispatchers track their position independently. Failed event processing goes to a dead letter queue with configurable retry, preventing a single bad event from blocking the entire pipeline.

10. **Temporal querying is a first-class feature** — the event store naturally supports "point-in-time" queries. This is critical for competitive intelligence where analysts need to understand what was known at a specific decision point, not just what is known now.

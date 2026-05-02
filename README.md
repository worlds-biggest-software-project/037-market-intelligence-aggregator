# Market Intelligence Aggregator

**Candidate #37** — An open-source competitive intelligence platform that monitors 100+ competitor signals (news, job postings, pricing, reviews) and synthesizes them into actionable battlecards and narratives using AI.

## Market Opportunity

The competitive intelligence tools market is **$0.6–0.9 billion in 2025**, projected to reach **$4+ billion by 2032–2034 at 12–21% CAGR**. The broader "market intelligence" space (including adjacent data vendors) is much larger at **$27+ billion through 2029**.

**Market context:**
- Crayon acquired by SoftwareOne for $1.4B (July 2025); signals consolidation
- AlphaSense alone reports $500M+ ARR; market is fragmented with multiple $100M+ players
- Only 6 major dedicated CI platforms exist; no open-source equivalent at all
- Signal-to-noise ratio is universal pain point: users report 80% of alerts are irrelevant

## What This Platform Solves

An **open-source competitive intelligence aggregator** that monitors 700,000+ companies across 100+ signal types and uses AI to connect the dots into strategic insights and sales enablement battlecards.

**Core value proposition:**
- **700,000+ company coverage**: News, job postings, pricing changes, reviews, patent filings
- **100+ signal types**: Website changes, G2 reviews, Crunchbase funding, LinkedIn updates, etc.
- **AI narrative synthesis**: Connects job postings + pricing drops + patent filings into coherent competitive threats
- **Battlecard generation**: Auto-drafts win/loss talking points from competitor signals with source citations
- **Role-based feeds**: Customized alerts for sales reps vs. product strategists (vs. generic broadcasts)
- **Self-hostable**: Full data sovereignty; no cloud-only lock-in

## Competitive Differentiation

| Aspect | This Platform | Crayon | Klue | Contify | Feedly | Owler |
|--------|---------------|--------|------|---------|--------|-------|
| **Company Coverage** | 700K+ | 50K (named) | 10K (named) | 700K+ | N/A | 20M |
| **Signal Types** | 100+ | 100+ | 50+ | 50+ | 40M sources | News only |
| **Cross-Signal Synthesis** | Yes (AI) | Partial | Partial | Yes | No | No |
| **Battlecard Gen** | Yes (AI) | Yes (AI) | Yes (AI) | Limited | No | No |
| **Open Source** | Yes | No | No | No | No | No |
| **Self-Hostable** | Yes | No | No | No | No | No |
| **Price** | Free/Cloud | $20K–$40K | $20K–$40K | $12K–$25K | $99/mo | Freemium |

## Key Features

### Must-Have (MVP)
- Source monitoring: RSS/Atom feeds, company websites (respecting robots.txt), job postings, news APIs
- NLP entity extraction: company names, executives, products, events
- Team workspace: shared intelligence boards, alert subscriptions
- Basic competitor profiles aggregating recent signals
- GDPR/CCPA-compliant data handling with PII minimization
- Open-source core under MIT or Apache 2.0

### Should-Have (v1.1)
- AI-generated competitive narratives connecting cross-signal patterns
- Battlecard template generation with source citations
- Win/loss interview ingestion and pattern extraction
- MCP server exposure for AI agent interoperability
- SEC EDGAR (XBRL) connector for public company filings

### Nice-to-Have (Backlog)
- Personalized daily briefings by role and deal context
- Dark web and community forum monitoring
- Patent filing monitoring via USPTO API
- Web traffic signal integration (Similarweb alternative)
- Multi-language monitoring with auto-translation

## Technology Stack

**Backend**: Python, Scrapy/Playwright for web crawling  
**NLP**: spaCy for NER, transformers for classification  
**Data Store**: PostgreSQL + Elasticsearch for search  
**Workflow**: Apache Airflow for monitoring schedules  
**Frontend**: React, TypeScript  
**Licensing**: MIT or Apache 2.0 (fully permissive)

## Market Entry Strategy

1. **MVP Launch** (months 1–4): RSS/web monitoring + NLP + basic competitor profiles
2. **AI Features** (months 5–8): Cross-signal synthesis, narrative generation, MCP server
3. **Battlecard Gen** (months 9–12): Automated battlecard drafting with source citations
4. **Monetization**: Open-source core + managed cloud tier ($500–$2K/month), enterprise support ($5K+/month)

## Why This Matters

- **No open-source alternative exists**: Crayon, Klue, Contify are all proprietary. Open-source option captures developer-forward SMBs and enterprises wanting data sovereignty
- **Signal-to-noise problem is universal**: All tools produce alert floods. AI-native approach learns what signals drive decisions for specific organizations and suppresses noise
- **Cross-signal synthesis is manual**: Connecting job postings + pricing shifts + patent filings is the highest-value analysis, but requires manual analyst work. AI-native automation would transform CI workflows
- **Battlecard generation is nascent**: Crayon and Klue are only beginning to auto-draft; quality is inconsistent. LLM-native approach could produce consistently useful talking points
- **EU CSDDD creating regulatory demand**: New German/EU supply chain due diligence laws create demand for supplier risk modules that no CI tool fully addresses

## Success Metrics

- **Year 1**: 500+ active deployments, $200K ARR from managed cloud + support
- **Year 2**: 2,000+ active deployments, $1M ARR; featured in G2 Leaders
- **Year 3**: 5,000+ active deployments, $3M+ ARR; adopted by 100+ mid-market / enterprise teams

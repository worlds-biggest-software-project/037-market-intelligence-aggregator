# Market Intelligence Aggregator — Feature & Functionality Survey

> Candidate #37 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| AlphaSense | Commercial SaaS | Proprietary; ~$24,000/user/yr | https://www.alpha-sense.com |
| Crayon | Commercial SaaS | Proprietary; ~$20,000–$40,000/yr | https://www.crayon.co |
| Klue | Commercial SaaS | Proprietary; ~$20,000–$40,000/yr | https://klue.com |
| Contify | Commercial SaaS | Proprietary; ~$12,000–$25,000/yr est. | https://www.contify.com |
| Owler | Commercial SaaS (Freemium) | Proprietary; Free / $39–$99/mo | https://www.owler.com |
| Similarweb | Commercial SaaS | Proprietary; ~$15,000–$60,000/yr | https://www.similarweb.com |
| ZoomInfo | Commercial SaaS | Proprietary; from ~$15,000/yr | https://www.zoominfo.com |
| 6sense | Commercial SaaS | Proprietary; ~$55,000/yr median | https://6sense.com |
| Feedly for Teams | Commercial SaaS | Proprietary; from ~$99/mo team | https://feedly.com |
| Intelligence2day (Comintelli) | Commercial SaaS | Proprietary; custom enterprise | https://intelligence2day.com |

## Feature Analysis by Solution

### AlphaSense

**Core features**
- Unified search across company filings, earnings transcripts, broker research, expert call transcripts, regulatory filings, press releases, and trade journals in a single interface
- NLP-powered semantic search that surfaces relevant passages even when exact keywords are absent
- Generative Search (relaunched January 2026) functions as an end-to-end research agent: finds documents, synthesises answers, automates workflows, and drafts deliverables
- Financial Data product (launched October 2025) integrates structured quantitative financial metrics alongside qualitative intelligence in a single conversational interface
- Channel Checks: real-time expert conversations converted into structured, repeatable intelligence streams covering demand patterns, pricing movements, and supply-chain shifts
- Deep Research: automated market landscape and company diligence drafting, replacing multi-hour manual analyst work
- Smart synonyms and entity-aware search that maps jargon, ticker symbols, and related concepts
- Sentiment analysis and trend tracking across structured and unstructured document sets

**Differentiating features**
- Largest proprietary premium content library in the market (broker research, expert networks via Tegus acquisition); most competitors rely on public web data
- Named a Leader in the inaugural Gartner Magic Quadrant for Competitive and Market Intelligence (CMI) Platforms; positioned highest on Ability to Execute
- Research agent capable of generating full diligence-ready deliverables, not just retrieved passages
- Channel Checks product provides primary market research at scale without manual survey work

**UX patterns**
- Search-first interface optimised for financial and strategy professionals; not designed for sales enablement
- Document viewer with inline annotations and collaborative workspace for analyst teams
- Conversational research agent interface introduced 2025–2026 lowers query complexity for power users
- Alert digests and email delivery for passive monitoring

**Integration points**
- API access for programmatic data retrieval into enterprise data warehouses
- Salesforce and CRM integrations for surfacing intel at deal stage
- Microsoft Office integration for exporting research into Word and PowerPoint
- SSO/SAML for enterprise identity management

**Known gaps**
- Price point ($24,000/user/yr) excludes SMB and most mid-market buyers entirely
- Focused on financial services and corporate strategy; limited battlecard / sales enablement workflow compared to Crayon or Klue
- No open-source component; locked ecosystem for proprietary data sources
- Gartner peer reviewers note that non-financial coverage (social media, review sites) is weaker than dedicated CI platforms

**Licence / IP notes**
- Fully proprietary commercial product; no open-source components disclosed
- Premium content (broker research, expert transcripts) is subject to third-party licensing agreements — redistribution or re-publishing is restricted
- Tegus acquisition (2024, $930M) brings expert-network transcript rights that competitors cannot replicate without similar deals

---

### Crayon

**Core features**
- Automated tracking of 100+ competitor data types across websites, pricing pages, job postings, press releases, G2 reviews, social media, and news
- AI-scored intelligence feed that ranks signals by estimated market impact
- Dynamic battlecard builder with real-time competitor data, objection-handling scripts, and talk tracks
- Crayon Answers: generative AI assistant for on-demand competitive questions during live sales conversations
- Win/loss analysis workflow with structured interview capture and trend reporting
- Newsletter and digest automation for distributing intel to sales teams via email and Slack
- MCP (Model Context Protocol) server (launched September 2025) enabling AI tools and agents to query Crayon's competitive data over a standardised protocol

**Differentiating features**
- Dedicated battlecard-to-CRM workflow is the most mature in the market; battlecards embed directly in Salesforce and HubSpot deal records
- MCP server makes Crayon's CI data accessible to external AI agents — a first in the category (as of September 2025)
- AI adoption benchmarking: State of CI 2025 report tracks industry trends and is cited by buyers as a credibility signal
- SoftwareOne acquisition (July 2025, $1.4B) adds enterprise distribution across 70 countries

**UX patterns**
- Dashboard centred on competitor profiles with change-log timelines
- Slack-native delivery for reps who never visit the platform directly
- In-Salesforce battlecard widgets reduce context switching for sales reps
- Automated alert subscriptions configurable by topic, competitor, or data type

**Integration points**
- Salesforce, HubSpot, Microsoft Dynamics (native CRM integrations)
- Slack, Microsoft Teams (delivery channels)
- MCP server for AI agent interoperability (September 2025)
- Zapier/webhooks for custom workflow automation

**Known gaps**
- Strong on named-competitor tracking; struggles with broader market or industry signal monitoring (regulatory shifts, macroeconomic signals)
- Review site data primarily sourced from G2 and Capterra; weaker on Trustpilot, Reddit, and community forums
- AI battlecard quality reportedly inconsistent; heavy curation still required by human analysts
- SoftwareOne acquisition introduces uncertainty about independent product roadmap

**Licence / IP notes**
- Fully proprietary commercial product; no open-source components
- Web data collected under Terms of Service; respects robots.txt per company policy statements
- No patent concerns identified in public disclosures

---

### Klue

**Core features**
- Automated web crawling combined with human curator layer to build and maintain competitor intelligence profiles
- Battlecard creation with AI-generated content, objection handling, and positioning guidance
- StaKs Engine: multi-agent system running hundreds of parallel extraction, scoring, and routing agents to populate battlecards continuously
- Compete Agent: agentic AI operative delivering real-time competitive deal intelligence to sellers in Slack, Salesforce, and Klue
- Competitor Profiles: on-demand AI-generated profiles refreshed every 24 hours covering news, pricing, strengths, weaknesses, and market messaging
- Win/loss interview analysis with automated insight extraction from call recordings and transcripts
- Revenue influence and win-rate KPI tracking to quantify CI programme ROI

**Differentiating features**
- StaKs Engine multi-agent architecture represents the most agentic CI data pipeline currently available commercially
- Compete Agent pushes intel proactively to reps at deal-relevant moments without requiring them to query the system
- One-click AI-generated battlecard drafts from G2 reviews and win/loss interview content
- Win/loss programme integration tightly coupled with CI content — competitors have kept these as separate workflows

**UX patterns**
- Curator-assisted content quality review layer differentiates Klue from fully automated tools; perceived accuracy higher among enterprise buyers
- Slack-first delivery model; reps interact via conversational commands without leaving their workflow
- Battlecard templates with structured sections (overview, pricing, objections, proof points) enforce consistency across teams

**Integration points**
- Salesforce, HubSpot, Microsoft Dynamics (native battlecard embedding)
- Slack, Microsoft Teams
- Gong, Chorus (call recording platforms) for win/loss transcript ingestion
- G2 review API for structured review data ingestion

**Known gaps**
- Breadth limited to named competitors; does not monitor broader market signals or emerging entrants
- No web traffic intelligence (no Similarweb-equivalent layer)
- Pricing ($20,000–$40,000/yr) positions it as enterprise-only; no self-serve or SMB tier
- Human curator dependency creates a bottleneck for high-velocity intelligence scenarios

**Licence / IP notes**
- Fully proprietary commercial product
- No patent concerns identified in public disclosures
- G2 review data accessed via third-party API under G2 licensing terms

---

### Contify

**Core features**
- Monitors 700,000+ companies and 1M+ editorially curated sources for market and competitive signals
- News and content taxonomy with customisable topic classification for industry-specific signal filtering
- Athena Agentic AI Engine (launched 2025): autonomous intelligence extraction across external signals (news, websites, reviews, job boards) and internal datasets (sales transcripts, SharePoint documents)
- Ask Athena: conversational chat interface for on-demand research, built on RAG over enterprise knowledge graphs
- 20+ auto-updating strategic insights per company including strengths, weaknesses, strategic objectives, key offerings, customer segments, and market positioning
- AI-generated newsletters for distributing curated intelligence summaries to stakeholder groups
- Snowflake Cortex integration: SQL-based querying over Contify's intelligence corpus for data teams
- Multi-language support: automatic translation into 50+ languages from global sources

**Differentiating features**
- Breadth of company coverage (700,000+) exceeds dedicated battlecard tools (Crayon, Klue) significantly
- Internal data ingestion (sales transcripts, SharePoint) enables cross-domain intelligence that treats external market signals and internal voice-of-customer data in a unified analysis layer
- Snowflake Cortex integration is the only direct SQL/BI warehouse bridge in the CI category as of 2025
- Knowledge-graph-based RAG approach reduces hallucination risk compared to direct LLM prompting over raw documents

**UX patterns**
- News-feed style intelligence inbox with taxonomy-based filtering
- Alert subscriptions configurable by company, topic, geography, and source type
- AI newsletter builder for low-effort stakeholder digest creation
- Dashboard for intelligence coverage metrics and stakeholder engagement tracking

**Integration points**
- Snowflake (native Cortex AI integration)
- SharePoint and internal document sources
- Slack, email delivery
- 30+ premium data providers (Statista, S&P Global) via content connectors
- API for BI tool and CRM integration

**Known gaps**
- Less known in North American enterprise market; smaller brand than Crayon or Klue in the US
- Social media monitoring is notably absent (confirmed by Gartner peer reviews)
- Users report excessive manual curation still required to keep noise out of feeds
- Webpage extractor functionality noted as needing improvement in public reviews
- Limited battlecard / sales-enablement workflow compared to Crayon or Klue

**Licence / IP notes**
- Fully proprietary commercial product; funding not publicly disclosed
- Athena AI built on RAG over proprietary knowledge graphs; underlying LLM vendor not publicly disclosed

---

### Owler

**Core features**
- Company news aggregation and alert monitoring for 20M+ companies sourced from public news and community contributions
- AI-summarised daily digests covering competitor funding rounds, leadership changes, acquisitions, and product news
- Competitive graph visualisation mapping direct and indirect competitors for a given company
- Company profiles with revenue estimates, headcount, funding history, and key executive data
- Freemium access model: track up to 5 companies at no cost

**Differentiating features**
- Widest SMB accessibility in the category — free tier covers basic competitor monitoring for small teams
- Community-powered data model provides coverage of smaller private companies that data vendors (ZoomInfo, Crunchbase) miss
- Competitive graph feature for visualising competitor relationships is unique in the freemium tier

**UX patterns**
- Daily digest email as primary delivery mechanism — passive consumption model
- Simple company profile pages with timeline of news events
- Mobile-friendly interface designed for on-the-go monitoring

**Integration points**
- Limited; primarily email digest delivery
- No documented CRM or Slack native integrations beyond browser extensions

**Known gaps**
- Very limited signal depth compared to enterprise tools; no NLP extraction, taxonomy, or battlecard workflows
- Community-sourced data accuracy varies significantly, especially for private companies
- No custom intelligence taxonomy, filtering, or win/loss workflows
- Enterprise teams rapidly outgrow the platform
- No structured data analysis, funnel, or attribution layer

**Licence / IP notes**
- Fully proprietary commercial product
- Community-sourced data creates unclear data provenance and accuracy guarantees

---

### Similarweb

**Core features**
- Web traffic intelligence: visits, engagement metrics, geographic breakdowns, device splits, and traffic source attribution for any website globally
- Competitor digital marketing analysis: keyword rankings, paid search spend, referral traffic sources, social media traffic share
- Audience overlap analysis: identifies shared audiences between your site and competitors
- App Intelligence: combined iOS+Android download and engagement tracking across 4.7M mobile apps
- Shopper Intelligence: e-commerce traffic and conversion benchmarking
- AI Brand Visibility (Web Intelligence 4.0, 2025): tracks brand mentions in ChatGPT and other AI chatbot responses
- AI Chatbot Traffic analytics: measures traffic arriving from major AI tools
- AI Strategist and AI Trend Analyzer: AI-powered tools for SEO strategy and content planning

**Differentiating features**
- Only platform offering AI chatbot referral traffic tracking and brand visibility measurement in AI-generated responses — a new signal category created by the generative AI browsing shift
- Broadest web traffic panel globally (210 countries, 100M websites) with proprietary panel methodology
- Shopper Intelligence layer adds e-commerce competitive context unavailable in traditional CI tools

**UX patterns**
- Dashboard-centric with pre-built competitive benchmarking report layouts
- Side-by-side website comparison views for quick competitive snapshots
- Data export to BI tools via API and CSV for analyst workflows

**Integration points**
- Salesforce, HubSpot (CRM integrations for sales intelligence use cases)
- Google Analytics, Looker (analytics tool integrations)
- Ad platforms (Google Ads, Meta) for campaign overlap analysis
- API (Data-as-a-Service tier) for BI dashboard feeds

**Known gaps**
- Coverage is strictly digital signals; no news monitoring, job postings, SEC filings, or review-site intelligence
- Traffic estimates for low-traffic sites (<10,000 visits/month) are unreliable due to panel size limitations
- No battlecard, win/loss, or sales enablement workflow
- Pricing ($15,000–$60,000/yr) is steep for primarily a traffic-data product
- No open-source version or self-host option

**Licence / IP notes**
- Publicly traded company (NYSE: SMWB); proprietary methodology for traffic estimation
- No patent concerns identified in public disclosures; traffic panel methodology is trade secret, not patented

---

### ZoomInfo

**Core features**
- B2B contact and company database covering 300M+ professionals and 100M+ companies
- Intent data signals tracking anonymous digital research behaviour to identify in-market buyers
- Technographic data mapping competitor software stack for targeted displacement selling
- Conversation Intelligence (acquired Chorus.ai) for sales call recording, transcription, and coaching
- ZoomInfo Copilot: AI-assisted prospecting recommendations, account scoring, and outreach sequence generation
- Company firmographic data: revenue, headcount, industry, and organisational hierarchy

**Differentiating features**
- Deepest B2B contact data of any vendor — unmatched for sales prospecting and ABM targeting
- Technographic layer enables competitive displacement plays by identifying which competitors are installed at a prospect
- Conversation Intelligence integration links pipeline activity to market signal context in a single platform

**UX patterns**
- Primarily a sales and marketing workflow tool; CI features are surfaced within prospecting flows rather than a dedicated CI module
- CRM-native delivery via Salesforce and HubSpot enrichment workflows
- AI Copilot surfaces account recommendations within existing rep workflows

**Integration points**
- Salesforce, HubSpot, Microsoft Dynamics (deep native CRM enrichment)
- Outreach, SalesLoft (sales engagement platforms)
- Marketo, Pardot (marketing automation)
- API for data enrichment pipelines

**Known gaps**
- Competitive intelligence is a secondary feature; no news monitoring, patent tracking, or regulatory signal coverage
- Intent data accuracy disputed by some enterprise buyers in G2/Gartner reviews
- Very expensive for pure market intelligence use cases ($15,000–$55,000+/yr)
- Data accuracy for smaller and non-US companies is weaker than for US enterprise targets

**Licence / IP notes**
- Fully proprietary commercial product
- Contact data collection practices have been subject to litigation and regulatory scrutiny under GDPR and CCPA; buyers should conduct due diligence

---

### 6sense

**Core features**
- Account engagement platform tracking anonymous digital buying signals to identify in-market accounts
- Intent data aggregation across 40,000+ B2B websites and the broader 6sense network
- AI-powered account scoring and buying stage prediction (Awareness, Consideration, Decision, Purchase)
- Predictive analytics for pipeline generation: identifies which accounts are most likely to convert in a given quarter
- Advertising module for account-based advertising to in-market buyers
- Revenue AI: cross-team platform coordinating marketing, sales, and customer success around shared account intelligence

**Differentiating features**
- Dark funnel visibility: tracks anonymous buying behaviour before a prospect ever fills out a form or engages with sales — a signal type unavailable in most CI tools
- Account-level buying stage prediction is more accurate than contact-level intent scoring at the enterprise level
- Tightest integration between intent signal and ad targeting of any platform in the category

**UX patterns**
- GTM-focused workflow: signals surface within account planning, pipeline reviews, and advertising campaign management
- Not designed as a passive news-monitoring or research tool
- Executive dashboards for pipeline risk and opportunity prioritisation

**Integration points**
- Salesforce, HubSpot, Microsoft Dynamics
- LinkedIn, Google, Meta (ad platform integrations for ABM)
- Outreach, SalesLoft, Salesloft
- Marketo, Eloqua, HubSpot Marketing

**Known gaps**
- Intent data is GTM-focused, not strategic CI; not useful for monitoring competitor strategy, product launches, or regulatory change
- Very expensive ($55,000/yr median) for the breadth of use cases supported
- No battlecard, win/loss, or news monitoring workflow
- Heavy dependence on 6sense's own network panel for intent data — accuracy disputed outside US markets

**Licence / IP notes**
- Fully proprietary commercial SaaS; Series F stage company (~$200M+ raised)
- No patent concerns identified; intent data methodology relies on proprietary panel and cookie-based tracking, subject to third-party cookie deprecation risk

---

### Feedly for Teams

**Core features**
- RSS/Atom feed aggregation across 40+ million sources with team-level shared workspaces
- Leo AI assistant: automatic article classification, noise filtering, entity tracking, and topic summarisation
- Team Boards for collaborative article prioritisation and annotation
- AI-powered industry intelligence, cybersecurity threat intelligence, biopharma research, and competitive intelligence skill modules (Teams and Enterprise plans)
- Newsletter creation tools for distributing curated intelligence summaries to stakeholders
- Up to 7,500 monitored sources on enterprise plans
- SAML/SSO, API access, and Slack/Microsoft Teams integrations on enterprise plans

**Differentiating features**
- Most affordable team intelligence aggregator with genuine AI filtering capability (starts at $99/month for teams)
- Leo's named-entity tracking learns from user engagement to progressively improve signal quality without manual rule configuration
- Cybersecurity and biopharma specialist AI modules — niche signal categories no other CI tool supports at this price point

**UX patterns**
- News-reader-style UI familiar to knowledge workers; low training requirement
- Leo AI surfaces priority articles at the top of feeds without requiring filter configuration
- Team curation workflow with inline comments and board assignments

**Integration points**
- Slack, Microsoft Teams (delivery integrations)
- API for custom integration into internal tools and portals
- Zapier for no-code workflow automation
- Browser extension for ad-hoc article capture

**Known gaps**
- No battlecard, win/loss, or sales-enablement workflow
- Leo AI filtering is effective for news; no structured competitive taxonomy or entity relationship analysis
- No web traffic, job posting, patent, or SEC filing intelligence
- Enterprise plan recently repriced to $1,600/month — a significant jump from team tiers
- Coverage quality for niche B2B verticals and non-English sources is variable

**Licence / IP notes**
- Fully proprietary commercial product
- No open-source components; underlying Leo AI model not disclosed

---

### Intelligence2day (Comintelli)

**Core features**
- Structured intelligence collection, analysis, and internal dissemination workflow platform
- Automatic NLP classification, translation (50+ languages), and summarisation of ingested content
- 250,000+ publicly available source monitoring plus 30+ premium data providers (Statista, S&P Global)
- Customisable intelligence newsletter and alert workflows for stakeholder distribution
- Knowledge management layer for tagging, categorising, and archiving intelligence artefacts
- API for BI integration

**Differentiating features**
- Strongest structured internal dissemination workflow in the category — enables formalised CI programmes with defined roles, approval workflows, and distribution lists
- Premium content connector ecosystem (30+ providers) rivals AlphaSense for structured data breadth at a significantly lower price point
- Long-standing niche vendor (20+ years) with deep practitioner credibility in European CI community

**UX patterns**
- Workflow-oriented interface designed for CI programme managers rather than individual end users
- Newsletter and alert builder with template management for recurring distribution cycles
- Dashboard for programme coverage metrics and stakeholder engagement tracking

**Integration points**
- Snowflake (via Contify relationship; Intelligence2day has its own connector ecosystem)
- 30+ premium data providers via licensed content connectors
- Slack, email delivery
- API for enterprise data warehouse integration

**Known gaps**
- Social media monitoring is absent (confirmed in Gartner peer reviews)
- Webpage extractor requires significant manual configuration to maintain accuracy
- Limited brand recognition outside European enterprise CI practitioners
- AI capabilities are emerging but trail Contify's Athena and Crayon/Klue's agentic features
- Very small vendor (limited engineering resources for rapid iteration)

**Licence / IP notes**
- Fully proprietary commercial product; Comintelli is a privately held Swedish company
- No patent concerns identified in public disclosures

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Source monitoring covering news, company websites, press releases, and job postings
- Configurable alert subscriptions with deduplication and noise filtering
- Entity extraction identifying companies, executives, products, and events from unstructured text
- Digest or newsletter delivery via email and Slack
- Search across aggregated content with relevance ranking
- Role-based access control and team-level content sharing
- API access for downstream BI and CRM integration
- GDPR-compliant data handling for personal data in scraped content

### Differentiating Features
- AI-generated battlecard drafts sourced from live competitor intelligence
- Agentic intelligence pipelines that autonomously extract and classify signals without rule configuration
- Cross-signal synthesis connecting job postings, product changes, pricing shifts, and patent filings into a coherent competitive narrative
- MCP server exposure making CI data accessible to external AI agents and assistants
- Primary data sources (expert networks, channel checks) inaccessible to web-crawl-only tools
- Web traffic intelligence (digital share of mind, keyword ranking, paid search spend)
- Dark funnel / intent data identifying anonymous in-market buyers before first contact
- Internal data ingestion (sales transcripts, SharePoint) for cross-domain intelligence

### Underserved Areas / Opportunities
- No open-source tool exists combining structured source monitoring, NLP entity extraction, competitive taxonomy, and team workflow
- Cross-signal synthesis (connecting job postings + pricing changes + patent filings into a single threat narrative) is entirely manual in all current tools
- Personalised intelligence delivery — same feed delivered to sales rep and corporate strategist; role-aware filtering is absent
- SMB and mid-market segment is served only by Feedly (limited depth) or Owler (no workflow); a capable open-source tool would address this gap directly
- Social media and community forum monitoring (Reddit, HN, Discord) is weak or absent in all enterprise CI tools
- Patent and regulatory filing monitoring is absent from all dedicated CI tools (only AlphaSense covers filings, and only for financial services contexts)

### AI-Augmentation Candidates
- Signal-to-noise filtering trained on organisation-specific decision patterns (what alerts have driven past actions?)
- Automated competitive narrative generation: "Competitor X is pivoting toward Y — evidence: 3 job postings, 2 review site mentions, 1 patent filing"
- Battlecard section drafting from live sources with citation tracking
- Win/loss pattern extraction from call transcripts and interview notes
- Emerging entrant detection from job posting velocity and funding signal patterns
- Personalised daily briefing generation adapting content to recipient role, current deals, and stated priorities

---

## Legal & IP Summary

All tools surveyed are proprietary commercial products with no open-source components. Web data collection raises compliance considerations under GDPR and CCPA for any tool scraping public web content containing identifiable individuals (executive names, contact details). ZoomInfo has faced prior litigation over data collection practices, and buyers should conduct independent legal review before deploying any platform for European user data. Crayon's MCP server integration and Contify's Snowflake Cortex integration do not introduce additional IP risk. AlphaSense's proprietary content library (broker research, expert transcripts via Tegus) is protected by third-party licensing agreements that prohibit redistribution. No patent-encumbered techniques have been identified in public disclosures for any tool surveyed. A new open-source project in this space would need to comply with SCIP ethical guidelines, robots.txt directives, GDPR/CCPA personal data constraints, and RSS/Atom feed licensing terms — none of which present blocking IP issues.

---

## Recommended Feature Scope

**Must-have (MVP)**:
- Source monitoring across RSS/Atom feeds, company websites (respecting robots.txt), job postings, and news APIs with configurable entity tracking (competitors, products, executives)
- NLP pipeline: named entity recognition, topic classification, relevance scoring, and deduplication
- Team workspace: shared intelligence boards, alert subscriptions, and digest delivery via email and Slack
- Basic competitor profile pages aggregating recent signals, categorised by type (product, pricing, talent, regulatory)
- GDPR/CCPA-compliant data handling with personal data minimisation controls
- Open-source core under a permissive licence (MIT or Apache 2.0) with self-host support

**Should-have (v1.1)**:
- AI-generated competitive narrative summaries connecting cross-signal patterns (job postings + pricing changes + review trends)
- Battlecard template generation with source citations and freshness timestamps
- Win/loss interview ingestion and pattern extraction from call transcripts
- MCP server exposure for AI agent interoperability
- Structured connector to SEC EDGAR (XBRL) for public company filing signals

**Nice-to-have (backlog)**:
- Personalised daily briefing generation adapting content to individual user role and current deal context
- Dark web and community forum monitoring (Reddit, Discord, HackerNews)
- Patent filing monitoring via USPTO public API
- Web traffic signal integration (via Similarweb or open alternatives) for digital share-of-mind tracking
- Multi-language source monitoring with automatic translation (top 10 languages initially)

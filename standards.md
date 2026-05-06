# Standards & API Reference

> Project: Market Intelligence Aggregator · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022 — Information Security Management Systems**
  URL: https://www.iso.org/standard/27001
  Relevant because any aggregator collecting and storing commercial intelligence signals (news, filings, job postings, review data) must demonstrate information security controls acceptable to enterprise buyers. SOC 2 Type II certification (common enterprise requirement) maps closely to ISO 27001 controls.

- **ISO/IEC 42001:2023 — Artificial Intelligence Management Systems**
  URL: https://www.iso.org/standard/81230.html
  Directly relevant for AI-native tools that apply LLM reasoning to synthesise competitive intelligence. Provides a framework for responsible AI use, including transparency, bias management, and human oversight — concerns that arise when AI-generated battlecards or risk assessments drive commercial decisions.

- **ISO 8601 — Date and Time Format**
  URL: https://www.iso.org/iso-8601-date-and-time-format.html
  The baseline standard for representing timestamps in intelligence feeds, article publication dates, and event timelines. Any API response or data export must use ISO 8601 datetime strings (e.g., `2026-05-06T14:30:00Z`) to avoid ambiguity across time zones.

- **ISO/TS 20451:2026 — Health Informatics / Regulated Product Information Exchange**
  URL: https://www.iso.org/standard/
  While domain-specific to pharma, this standard's pattern of unique entity identification and structured data exchange (building on the IDMP suite) illustrates the general approach to canonical entity identification that a market intelligence aggregator should adopt for company, product, and executive entities.

### W3C & IETF Standards

- **RSS 2.0 (Really Simple Syndication)**
  URL: https://www.rssboard.org/rss-specification
  De-facto standard for news and content syndication. Any market intelligence aggregator must support ingesting RSS 2.0 feeds as a primary source. RSS dominates content syndication (62% adoption across major publishers), and virtually every news publisher, blog, and press release wire supports it.

- **RFC 4287 — The Atom Syndication Format**
  URL: https://datatracker.ietf.org/doc/html/rfc4287
  IETF proposed standard for web feed syndication; technically superior to RSS 2.0 (IANA-registered MIME type, URI support, XML namespaces). Atom is preferred in real-time news aggregation pipelines where technical precision matters. Any aggregator should support both RSS 2.0 and Atom 1.0.

- **RFC 5005 — Feed Paging and Archiving**
  URL: https://datatracker.ietf.org/doc/html/rfc5005
  Extends RSS/Atom to support large, paginated, and archived feeds. Relevant for crawling and indexing historical news archives without requiring full re-scrape.

- **RFC 8288 — Web Linking**
  URL: https://datatracker.ietf.org/doc/html/rfc8288
  Standard for expressing links between web resources via HTTP Link headers and HTML `<link>` elements. Relevant for RESTful API navigation patterns (HATEOAS) and for interpreting structured metadata in scraped web content.

- **RFC 7807 — Problem Details for HTTP APIs**
  URL: https://datatracker.ietf.org/doc/html/rfc7807
  Standard for machine-readable error responses in HTTP APIs. Any public API built on top of the aggregator should use Problem Details format for consistent error communication to integrators.

- **Robots Exclusion Protocol (REP) / robots.txt**
  URL: https://www.rfc-editor.org/rfc/rfc9309 (RFC 9309, 2022)
  The formal IETF RFC standardising the `robots.txt` file format. Compliance with robots.txt directives is legally significant (used as evidence in CFAA cases in the US; treated as a key factor in GDPR Legitimate Interest balancing tests by France's CNIL and other EU DPAs). Any aggregator that performs web crawling must respect and log its robots.txt compliance posture.

- **Schema.org NewsArticle Markup**
  URL: https://schema.org/NewsArticle
  W3C community-backed structured data vocabulary embedded in HTML pages as JSON-LD, Microdata, or RDFa. Enables semantic extraction of article metadata (headline, author, datePublished, publisher, mentions) without brittle CSS selector parsing. Widely adopted by major news publishers.

- **OPML 2.0 — Outline Processor Markup Language**
  URL: https://opml.org/spec2.opml
  Standard format for exchanging lists of RSS/Atom feed subscriptions. Useful for bulk feed import and export in the aggregator's source management UI, enabling users to migrate their existing feed lists from tools like Feedly.

### Data Model & API Specifications

- **OpenAPI Specification 3.1 (OAS 3.1)**
  URL: https://spec.openapis.org/oas/v3.1.0.html
  The industry-standard format for describing RESTful API contracts. As a full superset of JSON Schema in v3.1, it provides language-agnostic API documentation, code generation targets, and client SDK scaffolding. The aggregator's public API should publish an OAS 3.1 specification to enable ecosystem adoption.

- **JSON Schema (Draft 2020-12)**
  URL: https://json-schema.org/specification
  The canonical standard for validating and documenting JSON data structures. Defines the data shapes for intelligence events (articles, signals, battlecard entries, source configurations). OAS 3.1 is a superset of JSON Schema, so designing data models in JSON Schema first enables consistent validation across API, webhook, and internal service layers.

- **Model Context Protocol (MCP) — 2025-11-25 Specification**
  URL: https://modelcontextprotocol.io/specification/2025-11-25
  Anthropic-originated open protocol (transport: JSON-RPC 2.0 over STDIO or HTTP+SSE) for integrating LLM applications with external data sources and tools. Directly relevant: both Crayon and Klue have already launched MCP servers (2025), allowing AI agents like Claude to query competitive intelligence data contextually. An open-source aggregator that publishes an MCP server becomes pluggable into any MCP-compatible AI workflow without custom integration code. GitHub: https://github.com/modelcontextprotocol

- **Standard Webhooks Specification**
  URL: https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md
  An emerging open standard for consistent, verifiable webhook payloads, covering payload format, signature verification (`svix-signature` / `webhook-id` headers), and retry semantics. Adopting this standard for the aggregator's outbound alert webhooks reduces integration friction for consumers.

- **Protocol Buffers (protobuf) / gRPC**
  URL: https://protobuf.dev / https://grpc.io
  Binary serialisation format and RPC framework used for high-throughput internal service communication (e.g., between a crawl pipeline, NLP enrichment service, and signal storage layer). Less relevant for public APIs but valuable for internal microservice architecture where JSON parsing overhead becomes a bottleneck at scale.

### Security & Authentication Standards

- **OAuth 2.0 (RFC 6749) and Bearer Token Usage (RFC 6750)**
  URL: https://datatracker.ietf.org/doc/html/rfc6749
  The standard authorisation framework used by virtually all modern SaaS APIs. All products surveyed (AlphaSense, Feedly, ZoomInfo, Similarweb) use OAuth 2.0 flows or API key + bearer token patterns compatible with OAuth 2.0 conventions. The aggregator's API must support OAuth 2.0 for third-party integration.

- **OpenID Connect 1.0 (OIDC)**
  URL: https://openid.net/specs/openid-connect-core-1_0.html
  Identity layer on top of OAuth 2.0; enables enterprise SSO (Single Sign-On) via identity providers (Okta, Azure AD, Google Workspace). Enterprise buyers require OIDC/SAML support for procurement approval. The aggregator should implement OIDC for authentication.

- **SCIM 2.0 — System for Cross-domain Identity Management (RFC 7643/7644)**
  URL: https://datatracker.ietf.org/doc/html/rfc7643
  Standard for automated user provisioning and de-provisioning across SaaS applications. Enterprise IT departments require SCIM support to manage user lifecycle without manual admin effort. With 72% of enterprises adopting multi-protocol SSO, SCIM compliance is a gate for enterprise sales.

- **GDPR (EU 2016/679) — General Data Protection Regulation**
  URL: https://gdpr-info.eu/
  Governs all processing of EU personal data, including data collected via web scraping (executive names, contact details, employer associations). Critical legal constraint: France's CNIL fined KASPR €240,000 for scraping LinkedIn contacts. Legitimate Interest must be documented with a three-part balancing test (legitimate purpose, necessity, proportionality); robots.txt compliance is now treated as a key positive factor. Data minimisation is mandatory.

- **CCPA/CPRA (California Consumer Privacy Act / Privacy Rights Act)**
  URL: https://oag.ca.gov/privacy/ccpa
  US analog to GDPR; applies to personal data of California residents scraped from public web content. Requires disclosure of data collection and opt-out mechanisms for sale of personal data. Any aggregator processing contact or executive data must implement CCPA compliance controls alongside GDPR.

- **OWASP API Security Top 10 (2023)**
  URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
  Authoritative checklist of the top API security vulnerabilities (broken object-level authorisation, authentication failures, excessive data exposure, etc.). Serves as the design and review checklist for the aggregator's API layer. Particularly relevant given that intelligence data has commercial sensitivity and could be a target for data exfiltration.

### Financial Disclosure Standards

- **SEC EDGAR XBRL / iXBRL APIs**
  URL: https://www.sec.gov/search-filings/edgar-application-programming-interfaces / https://data.sec.gov
  The US Securities and Exchange Commission provides free, public REST APIs over `data.sec.gov` for machine-readable XBRL financial data from all public company filings. The Company Facts API returns all XBRL disclosures for a given company as structured JSON, updated within minutes of filing. Authentication is not required but automated access must include a User-Agent header per SEC Fair Access guidelines. Essential for aggregating public company earnings signals.

---

## Similar Products — Developer Documentation & APIs

### AlphaSense

- **Description:** Enterprise AI-powered market and financial intelligence platform; aggregates filings, earnings transcripts, broker research, expert calls, and regulatory documents with NLP search and generative AI research agents.
- **API Documentation:** https://developer.alpha-sense.com/
- **SDKs/Libraries:** Python SDK (`alphasense-api-sdk`, via pip); JavaScript/TypeScript SDK (`@alphasense/sdk-create-app`, via npx). Both published at https://developer.alpha-sense.com/api/getting-started/sdk/
- **Developer Guide:** https://developer.alpha-sense.com/api/getting-started
- **Standards:** GraphQL interface for generative search; REST for ingestion API (OpenAPI / Swagger documented at https://developer.alpha-sense.com/api/ingestion/swagger)
- **Authentication:** Bearer token (OAuth 2.0-style access token flow)
- **Notes:** Ingestion API allows customers to push private documents into AlphaSense for unified search alongside public data. Sample use cases (including a Trending Information Dashboard) available at https://developer.alpha-sense.com/api/getting-started/sample-use-cases

### Crayon

- **Description:** Competitive intelligence platform tracking competitor website changes, job postings, press releases, review sites, and social media, with battlecard and sales enablement workflow.
- **API Documentation:** https://apidocs.crayon.com/
- **SDKs/Libraries:** None publicly documented; Content and Answers APIs are the primary integration surface.
- **Developer Guide:** https://www.crayon.co/blog/connecting-competitive-intel-to-enterprise-ai-with-api-mcp
- **MCP Server:** Crayon launched the first competitive intelligence MCP server (2025); blog post at https://www.crayon.co/blog/crayon-launches-first-competitive-intelligence-mcp-server
- **Standards:** REST/JSON
- **Authentication:** API key

### Klue

- **Description:** Competitive intelligence and battlecard platform focused on win/loss analysis, card distribution to sales teams, and AI-summarised intel delivery.
- **API Documentation:** Available in-app under Apps & Integrations (admin access required); Content API for pulling published cards into external tools. OpenAI specification published within the Content API documentation.
- **SDKs/Libraries:** None public; integration documented via Klue's admin portal.
- **Developer Guide:** https://klue.com/product/integrations
- **MCP Server:** Klue MCP Server (2025); tested with OpenAI Agent Builder, Copilot Studio, Claude Desktop, and enterprise LLM stacks. Announcement: https://klue.com/blog/introducing-klue-mcp-server
- **Standards:** REST/JSON; OpenAI-compatible function specification for Content API
- **Authentication:** API key (generated in Apps & Integrations admin section)

### Feedly

- **Description:** AI-powered news aggregator and research platform; Leo AI assistant filters signals, tracks named entities, and summarises topics for teams.
- **API Documentation:** https://developers.feedly.com/reference/introduction
- **SDKs/Libraries:** Python client: https://github.com/feedly/python-api-client
- **Developer Guide:** https://developers.feedly.com (primary developer hub)
- **Standards:** REST/JSON; supports RSS 2.0 and Atom 1.0 feed ingestion natively; OPML import/export
- **Authentication:** OAuth 2.0 Bearer Token (self-service token generation from Feedly account settings)
- **Notes:** All API requests go to `api.feedly.com` over HTTPS. Feedly's API is positioned for automating intelligence sharing workflows with leadership and operations teams.

### Owler

- **Description:** Company news aggregator providing AI-summarised daily digests; tracks competitor funding rounds, leadership changes, product announcements, and revenue signals for millions of companies.
- **API Documentation:** https://developers.owler.com/ and https://developers.owler.com/docs
- **SDKs/Libraries:** Ballerina connector available (`ballerinax/owler`); community integrations via n8n HTTP nodes.
- **Developer Guide:** https://corp.owler.com/owler-api-for-ai-guide
- **Standards:** REST/JSON
- **Authentication:** API key (via Owler representative)
- **Notes:** Owler's API is positioned for AI agent integration, enabling real-time company lookups by name or website URL, returning company data, news, blog posts, and competitor lists.

### Contify

- **Description:** AI + NLP market and competitive intelligence platform covering 700,000+ companies and 1M+ editorially curated sources; delivers structured GenAI-enriched news data via API.
- **API Documentation:** https://developer.contify.com / https://help.contify.com/contify-apis-enterprise-standard
- **SDKs/Libraries:** None publicly documented; REST API with JSON responses.
- **Developer Guide:** https://www.contify.com/news-api/
- **Standards:** REST/JSON; RSS feeds and webhooks also supported for data delivery
- **Authentication:** API key + App ID as Bearer token in request header
- **Notes:** Contify's News Data Feeds API provides GenAI-enriched, structured news data on companies, industries, and business topics, ready for direct integration into applications and internal tools.

### Similarweb

- **Description:** Digital market intelligence platform providing web traffic analytics, audience overlap, keyword rankings, app performance, and technographic data for competitive research.
- **API Documentation:** https://developers.similarweb.com/docs/similarweb-web-traffic-api and https://docs.similarweb.com/api-v5
- **SDKs/Libraries:** PHP client library (community): https://github.com/thunderer/SimilarWebApi
- **Developer Guide:** https://developers.similarweb.com/docs/getting-started
- **Standards:** REST/JSON (V5); Batch API available alongside REST for high-volume queries. V5 brings standardised response formats across endpoints and full MCP + AI support.
- **Authentication:** API key (via Similarweb Account Manager)
- **Notes:** Similarweb API V5 (current) supports multi-metric requests in a single call and provides access to billions of data points across websites, mobile apps, keywords, and companies.

### ZoomInfo

- **Description:** B2B intelligence platform covering company firmographics, contact data, intent signals, and technographics; enterprise CI features are secondary to its core data product.
- **API Documentation:** https://docs.zoominfo.com (current) and https://api-docs.zoominfo.com/ (legacy, being deprecated)
- **SDKs/Libraries:** None publicly documented; REST API with interactive examples available in docs.
- **Developer Guide:** https://help.zoominfo.com/s/article/Overview-of-ZoomInfo-Enterprise-API
- **Standards:** REST/JSON; JSON Web Token (JWT) authentication with 60-minute token expiry.
- **Authentication:** JWT (requires active ZoomInfo subscription; API credentials from account management)
- **Notes:** Access to premium APIs is gated by subscription tier. The legacy Enterprise API is being deprecated; new integrations should use `docs.zoominfo.com`.

### SEC EDGAR

- **Description:** US Securities and Exchange Commission's public electronic filing system; provides free access to all public company filings (10-K, 10-Q, 8-K, earnings, proxy statements) in structured XBRL/iXBRL format.
- **API Documentation:** https://www.sec.gov/search-filings/edgar-application-programming-interfaces
- **SDKs/Libraries:** `sec-edgar-api` Python wrapper (https://sec-edgar-api.readthedocs.io/); `sec-api` third-party Python library (https://pypi.org/project/sec-api/); XBRL US API (https://github.com/xbrlus/xbrl-api)
- **Developer Guide:** https://www.sec.gov/about/developer-resources; data available at https://data.sec.gov
- **Standards:** REST/JSON (Company Facts API, Submissions API, Frames API); XBRL taxonomy for financial data; iXBRL for inline filing documents
- **Authentication:** None required; must include `User-Agent` header with company name and email per SEC Fair Access policy. Bulk downloads available as nightly ZIP files (`companyfacts.zip`).

### GDELT Project

- **Description:** Global open-access news event database monitoring coverage in 65 languages; provides free APIs for querying news events, entity data, domain profiles, and sentiment signals globally.
- **API Documentation:** https://docs.gdeltcloud.com/developers/api-keys and https://gdeltcloud.com/api-docs
- **SDKs/Libraries:** Python client: https://github.com/alex9smith/gdelt-doc-api; REST API accessible via HTTP GET.
- **Developer Guide:** https://blog.gdeltproject.org/gdelt-doc-2-0-api-debuts/
- **MCP Server:** GDELT Cloud now exposes an MCP server for AI agent integration.
- **Standards:** REST/JSON; natural language query interface via GDELT Cloud
- **Authentication:** GDELT Cloud API key (free tier available); core GDELT 2.0 Doc API is free and unauthenticated.
- **Notes:** GDELT covers a rolling 3-month window of full-text searchable news in 65 languages, machine-translated to English. Useful as a free baseline signal source for news monitoring and geopolitical event detection.

---

## Notes

### Emerging Patterns

**MCP as the emerging integration standard for CI:** Both Crayon and Klue launched MCP servers in 2025, and Similarweb V5 explicitly claims "full MCP + AI support." The Model Context Protocol is becoming the defacto integration layer between AI agents and commercial intelligence data sources. An open-source aggregator that ships an MCP server out of the box will be positioned as a first-class data source for any agentic workflow — without requiring custom API integration work from each consumer.

**Standard Webhooks gaining traction:** The Standard Webhooks specification is being adopted as the baseline for consistent, verifiable outbound webhook payloads. Using this for alert delivery reduces the per-integration burden on receiving systems.

**Regulatory risk is asymmetric for personal data:** GDPR enforcement against web scrapers (KASPR €240,000 fine in France) has created a sharp distinction between scraping *organisational* signals (company news, filings, job postings at the firm level) and scraping *personal* data (executive names and contacts). An open-source aggregator that focuses on company-level signals and explicitly avoids personal contact data collection will face a substantially lower compliance burden than platforms like ZoomInfo or 6sense.

**Free public data sources reduce bootstrapping cost:** SEC EDGAR (all US public company filings, free REST API), GDELT (global news events, free), and RSS/Atom feeds (near-universal news coverage, free) together provide a strong foundation of structured signals without API spend — a meaningful differentiation from commercial platforms that charge for data access.

### Gaps

- No widely adopted open standard exists for competitive intelligence data interchange (analogous to OpenAPI for APIs or RSS for news). SCIP (Strategic and Competitive Intelligence Professionals) defines ethics and methodology frameworks but not a data format standard. This is an opportunity: an open-source aggregator could publish and promote a JSON Schema / OpenAPI data model for CI signal exchange as a community standard.
- Klue's Content API exposes an OpenAI-compatible function specification, but this is proprietary and not publicly documented outside the admin portal. Standardising battlecard and signal data models (company entity, signal event, intelligence card) would reduce integration friction for the ecosystem.

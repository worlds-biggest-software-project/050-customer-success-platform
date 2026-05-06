# Standards & API Reference

> Project: Customer Success Platform · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 20802-1:2016 — OData Core Protocol**
- URL: https://www.iso.org/standard/69208.html
- The Open Data Protocol (OData) v4 was submitted by OASIS to ISO/IEC JTC 1 in April 2015 and published as an international standard in December 2016. ChurnZero's REST API is an OData v4 implementation, making this standard directly relevant for implementing and consuming CS platform data APIs. Tools such as Excel, Tableau, and Power BI know how to consume OData natively, making it a strong choice for exposing portfolio health data to BI consumers.

**ISO/IEC 20802-2:2016 — OData JSON Format**
- URL: https://www.iso.org/standard/69209.html
- Companion standard to ISO/IEC 20802-1 defining the JSON serialisation format for OData v4 payloads. Relevant to any CS platform REST API that returns account, health score, or event data in JSON.

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- Requires organisations to establish processes for identifying and tracking customer requirements and analysing customer satisfaction. Provides a normative framework for the _business processes_ that CS platforms operationalise: customer feedback capture, satisfaction tracking, issue resolution, and continuous improvement loops.

**ISO/IEC 25012:2008 — Data Quality Model**
- URL: https://www.iso.org/standard/35736.html
- Defines data quality characteristics (accuracy, completeness, consistency, timeliness, accessibility) applicable to customer data ingested by CS platforms. Relevant to health score reliability: a health score is only as good as the data quality of its inputs (CRM, product analytics, support tickets).

**ISO/IEC 27001:2022 — Information Security Management**
- URL: https://www.iso.org/standard/27001
- The primary international standard for enterprise information security management systems. CS platforms that process customer behavioural data and receive CRM data containing PII should obtain ISO 27001 certification to meet enterprise procurement requirements. All surveyed CS platforms (Gainsight, ChurnZero, Vitally) advertise SOC 2 Type II compliance, which substantially overlaps with ISO 27001 requirements.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational standard for API access delegation. All major CS platform integrations (Salesforce, HubSpot, Zendesk, Slack) use OAuth 2.0 for authorisation. A CS platform acting as an OAuth client must implement the authorization code flow (for user-consented integrations) and client credentials flow (for server-to-server CRM and product analytics sync).

**RFC 6750 — The OAuth 2.0 Bearer Token Usage**
- URL: https://datatracker.ietf.org/doc/html/rfc6750
- Specifies how bearer tokens obtained via OAuth 2.0 must be transmitted in API requests (Authorization header). Mandatory for all REST API calls to CRM, product analytics, and support system integrations.

**RFC 8693 — OAuth 2.0 Token Exchange**
- URL: https://datatracker.ietf.org/doc/html/rfc8693
- Defines a Security Token Service pattern for exchanging one token for another — useful in multi-tenant CS platform architectures where the platform acts on behalf of different customer accounts across multiple downstream integrations simultaneously.

**RFC 8414 — OAuth 2.0 Authorization Server Metadata**
- URL: https://datatracker.ietf.org/doc/html/rfc8414
- Enables clients to discover OAuth endpoints programmatically from a well-known metadata document at `/.well-known/oauth-authorization-server`. Relevant for implementing a CS platform that exposes its own OAuth-protected API to third-party integrators.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer built on top of OAuth 2.0, answering "who is this user?" in addition to "what can this application do?" Relevant for CS platform SSO (Single Sign-On) integration with enterprise identity providers (Okta, Azure AD, Google Workspace) — a table-stakes requirement for enterprise deployments.

**RFC 7807 — Problem Details for HTTP APIs**
- URL: https://datatracker.ietf.org/doc/html/rfc7807
- Defines a standard JSON error response format for REST APIs. Adopting this in a CS platform API ensures consistent, machine-readable error reporting that integrators can handle programmatically.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Standardises the `Link` header for pagination, used alongside cursor-based pagination in CS platform REST APIs (Vitally uses cursor-based pagination; this RFC provides the standard anchoring).

---

### Data Model & API Specifications

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The industry standard for describing REST API contracts. Version 3.1 (released 2021) aligns with JSON Schema Draft 2020-12 and adds first-class support for webhooks in the `webhooks` object at the root of the specification. A CS platform should publish an OpenAPI 3.1 spec covering health score endpoints, account CRUD, CTA management, and webhook event schemas so that integrators can generate SDKs automatically.

**OData v4.01 Protocol (OASIS)**
- URL: https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part1-protocol.html
- The OASIS-maintained OData 4.01 specification. ChurnZero's REST API is an OData v4 implementation, giving it native compatibility with BI tools. An open-source CS platform could adopt OData for its reporting endpoints to enable plug-and-play connections to Power BI, Tableau, and Excel without custom connectors.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification.html
- The standard for validating JSON document structure. All CS platform data models (account health score payload, CTA event payload, playbook trigger payload) should be defined as JSON Schema documents to enable validation, documentation generation, and SDK generation.

**CloudEvents v1.0 (CNCF)**
- URL: https://cloudevents.io/
- A specification for describing event data in a common, portable format. Relevant to CS platform webhook delivery: wrapping health score change events, CTA creation events, and playbook trigger events in CloudEvents format enables interoperability with event brokers (AWS EventBridge, Kafka, Google Eventarc) and makes integrations more portable.

**Segment Spec (Twilio Segment)**
- URL: https://segment.com/docs/connections/spec/
- De facto industry specification for customer event tracking across six API call types: Identify, Track, Page, Screen, Group, and Alias. CS platforms ingesting product analytics data should understand and normalise incoming event data against the Segment Spec, as it is the most widely adopted behavioural data vocabulary in SaaS product analytics.

---

### Security & Compliance Standards

**SOC 2 Type II (AICPA Trust Services Criteria)**
- URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-ございませんfoundations/system-and-organization-controls
- The primary compliance certification required by enterprise customers before deploying a CS platform that will receive CRM data, customer PII, and product usage telemetry. SOC 2 Type II evaluates controls across five trust principles: Security, Availability, Processing Integrity, Confidentiality, and Privacy — evaluated over a minimum 6-month observation period. All surveyed platforms hold SOC 2 Type II reports.

**GDPR — General Data Protection Regulation (EU 2016/679)**
- URL: https://gdpr.eu/
- Customer behavioural data processed for health scoring is regulated under GDPR when the end customers are EU residents. A CS platform must: (a) sign Data Processing Agreements with enterprise customers that classify the platform as a data processor; (b) support right-to-erasure (right to be forgotten) for individual end-user records; (c) maintain a Record of Processing Activities. The EDPB's 2025 Coordinated Enforcement Framework specifically emphasised the Right to Erasure obligations for all sub-processors in a data chain.

**CCPA / CPRA — California Consumer Privacy Act**
- URL: https://oag.ca.gov/privacy/ccpa
- The California equivalent of GDPR, applicable to CS platforms serving US SaaS companies with California-resident customers. Unlike GDPR's opt-in model, CCPA provides opt-out rights for consumers. An AI CS platform training churn prediction models on behavioural data must ensure appropriate consent provisions and data minimisation practices.

**EU AI Act (Regulation (EU) 2024/1689)**
- URL: https://artificialintelligenceact.eu/
- Entered into force August 2024; provisions for high-risk AI systems began applying in August 2026. AI churn prediction models that inform decisions affecting customers (e.g. account cancellation, escalation routing) may qualify as AI systems subject to transparency, accuracy, and human oversight requirements. An AI-native CS platform should assess which model components fall within the Act's scope.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- Industry reference for API security vulnerabilities. Particularly relevant for a CS platform exposing health score data, playbook triggers, and CRM-synced account data via REST API. Key risks to mitigate: Broken Object Level Authorization (each tenant must only access their own account data), Broken Authentication, Excessive Data Exposure in API responses, and Injection in query filter parameters.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- URL: https://modelcontextprotocol.io/
- An open protocol for connecting AI models to external data sources and tools. Directly relevant to an AI-native CS platform: an MCP server exposing customer health scores, account history, and playbook execution capabilities would enable LLM agents (Claude, GPT-4o) to reason over live CS data and take actions (trigger plays, draft QBR content, escalate accounts) without requiring the AI to have direct API credentials. This aligns with the agentic playbook execution and autonomous QBR generation opportunities identified in the feature survey.

---

## Similar Products — Developer Documentation & APIs

### Gainsight CS

- **Description:** The market-leading enterprise CS platform with health scoring, journey orchestration, CTA management, and product analytics integration.
- **API Documentation:** https://support.gainsight.com/gainsight_nxt/API_and_Developer_Docs/About/API_Documentation_Overview
- **Key APIs:**
  - Company Object API: https://support.gainsight.com/gainsight_nxt/API_and_Developer_Docs/Company_and_Relationship_API/Company_API_Documentation
  - Custom Object API: https://support.gainsight.com/gainsight_nxt/API_and_Developer_Docs/Custom_Object_API/Gainsight_Custom_Object_API_Documentation
  - Bulk REST APIs: https://support.gainsight.com/gainsight_nxt/API_and_Developer_Docs/Bulk_API/Gainsight_Bulk_REST_APIs
  - User Management APIs: https://support.gainsight.com/gainsight_nxt/API_and_Developer_Docs/User_Management_APIs/User_Management_APIs
  - PX Developer API (product analytics): https://support.gainsight.com/PX/API_for_Developers
- **Standards:** REST/JSON; proprietary endpoint design; no published OpenAPI spec
- **Authentication:** API key (header-based); tenant-scoped access tokens
- **Rate Limits:** Synchronous: 100 calls/min, 50,000 calls/day; Async (Bulk): 10 calls/hour, 100 calls/day
- **2026 Update:** Copilot queries can now be executed via REST API, returning AI-generated natural language responses and structured tabular data in a single call

### ChurnZero

- **Description:** Mid-market CS platform with real-time churn detection, playbook automation, and in-app messaging; REST API is an OData v4 implementation.
- **API Documentation:** https://churnzero.com/features/rest-api/ and https://app.churnzero.net/developers
- **SDKs/Libraries:**
  - .NET Standard client: https://github.com/Talentech/ChurnZeroApiClient
  - Ruby wrapper: https://github.com/customerlobby/churn_zero
- **Developer Guide:** Login-protected at https://app.churnzero.net/developers
- **Standards:** OData v4 (OASIS/ISO/IEC 20802-1); dynamic API endpoints that adjust as ChurnZero instance customisations are made
- **Authentication:** API key; Developer Tools user permission required; Bulk Read Permissions must be explicitly granted
- **Integration:** Segment Destination documented at https://segment.com/docs/connections/destinations/catalog/churnzero/; Fivetran connector available at https://fivetran.com/docs/connectors/applications/churnzero/setup-guide

### Vitally

- **Description:** Modern mid-market CS platform with AI-powered health scoring, CSM task management, and the fastest deployment time in its segment (2–4 weeks).
- **API Documentation:** https://docs.vitally.io/en/articles/9880649-rest-api-overview
- **Key Endpoint Docs:**
  - REST API collection: https://docs.vitally.io/en/collections/10410457-rest-api
  - Projects API: https://docs.vitally.io/en/articles/9880676-rest-api-projects
  - Project Templates API: https://docs.vitally.io/en/articles/9880869-rest-api-project-templates
- **Postman Collection:** https://www.postman.com/solution-architects-4700/vitally-io-apis/documentation/div86y5/vitally-io-rest-api
- **Standards:** REST/JSON; cursor-based pagination (ordered by `updatedAt` or `createdAt`)
- **Authentication:** API key via header (`Authorization: Bearer <token>`); keys managed under Settings > Integrations > Vitally REST API
- **Rate Limits:** 60 requests per minute per API key
- **Supported Objects:** Users, Accounts, Conversations, Tasks, Notes, NPS Responses (create, update, retrieve, list)

### Totango

- **Description:** Enterprise CS platform with account hierarchy management, health scoring, and SuccessPlay automation; merged with Catalyst in 2024; Unison AI layer launched 2025.
- **API Documentation:** https://support.totango.com/hc/en-us/sections/360005893212-Totango-API
- **Key APIs:**
  - HTTP API Overview: https://support.totango.com/hc/en-us/articles/203639605-Totango-HTTP-API-Overview
  - Customer Data Hub API: https://support.totango.com/hc/en-us/articles/360043441152-Customer-Data-Hub-API
  - Touchpoints API: https://support.totango.com/hc/en-us/articles/115000597266-Touchpoints-API
  - Integration Hub: https://int-hub.totango.com/
- **SDKs:** iOS SDK available at https://github.com/totango/totango-ios-api; npm module `totango-tracker` for Node.js
- **Standards:** REST/JSON; HTTP API can be used alongside JavaScript event collection
- **Authentication:** API token; tenant-scoped

### Custify

- **Description:** SMB/mid-market CS platform with transparent pricing (from $399/month), health scoring, playbooks, and in-app messaging.
- **API Documentation:** https://docs.custify.com/
- **API Key Access:** https://app.custify.com/settings/developer/api-key
- **Standards:** REST/JSON; JavaScript snippet available for client-side event tracking
- **Authentication:** API key generated from Settings > Developer > API Access
- **Supported Objects via API:** People, Companies, Notes, Tags, Tickets, Tasks, Events, Requests, Comments, Deals, Invoices, Subscriptions, Files, Custom Data Objects
- **Integration:** Segment Destination documented at https://segment.com/docs/connections/destinations/catalog/custify/; RudderStack destination at https://www.rudderstack.com/docs/destinations/streaming-destinations/custify/

### Salesforce CRM

- **Description:** The dominant enterprise CRM platform; bidirectional Salesforce sync is a table-stakes requirement for all CS platforms — health scores, CTAs, and renewal forecasts must be written back to Salesforce account records.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/
- **REST API Developer Guide (PDF, Spring '26, v66.0):** https://resources.docs.salesforce.com/latest/latest/en-us/sfdc/pdf/api_rest.pdf
- **Authentication Guide:** https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/intro_oauth_and_connected_apps.htm
- **Standards:** REST/JSON; OAuth 2.0 (Authorization Code flow for user-consented access; Client Credentials flow for server-to-server sync)
- **Authentication:** OAuth 2.0 via Connected Apps; access tokens required for all REST API calls; token exchange via Salesforce Identity service

### HubSpot CRM

- **Description:** The leading SMB and mid-market CRM; HubSpot bidirectional sync is a table-stakes integration alongside Salesforce for CS platforms targeting smaller SaaS companies.
- **API Documentation:** https://developers.hubspot.com/docs
- **Developer Portal:** https://developers.hubspot.com/
- **Node.js SDK:** https://github.com/HubSpot/hubspot-api-nodejs
- **Standards:** REST/JSON; standard HTTP verbs; all responses in JSON; base domain `https://api.hubapi.com`
- **Authentication:** OAuth 2.0 only (API key authentication sunset); OAuth app required for all third-party integrations
- **Supported Objects:** Contacts, Companies, Deals, Tickets, Activities (Notes, Tasks, Emails, Meetings, Calls); each object has get, get-all, and batch-get endpoints

### Segment (Twilio)

- **Description:** The dominant customer data platform (CDP) and de facto event tracking standard; CS platforms ingest product analytics via Segment to populate behavioural health signals.
- **API Documentation:** https://segment.com/docs/
- **Spec Documentation:** https://segment.com/docs/connections/spec/
- **HTTP API Source:** https://segment.com/docs/connections/sources/catalog/libraries/server/http-api/
- **Config API Reference:** https://reference.segmentapis.com/
- **Standards:** Segment Spec defines six semantic event types: Identify, Track, Page, Screen, Group, Alias — the de facto vocabulary for SaaS product analytics events
- **Authentication:** Write keys (source-specific) for event ingestion; OAuth 2.0 for Config API management

### Amplitude

- **Description:** Product analytics platform; widely integrated by CS platforms as a source of behavioural health signals (feature adoption, login frequency, usage depth).
- **API Documentation:** https://amplitude.com/docs/apis
- **Analytics APIs:** https://amplitude.com/docs/apis/analytics
- **Dashboard REST API:** https://amplitude.com/docs/apis/analytics/dashboard-rest
- **HTTP API Integration:** https://amplitude.com/integrations/http-sdk
- **SDKs:** https://amplitude.com/docs/sdks (JavaScript, Python, iOS, Android, and others)
- **Standards:** REST/JSON; Dashboard REST API uses HTTP Basic Auth (API key + secret key); event ingestion accepts batches of up to 2,000 events and 10MB uncompressed per request
- **Authentication:** API key + secret key (Basic Auth) for Dashboard REST API; write key for event ingestion

### Mixpanel

- **Description:** Product analytics platform; alternative to Amplitude as a behavioural data source for CS health scoring; provides event ingestion API and reporting aggregation API.
- **API Documentation:** https://developer.mixpanel.com/reference/overview
- **Ingestion API:** https://developer.mixpanel.com/reference/ingestion-api
- **Import Events:** https://developer.mixpanel.com/reference/import-events
- **SDKs:** JavaScript, Python, PHP, and others at https://docs.mixpanel.com/docs/tracking-methods/sdks/
- **Standards:** REST/JSON; event import accepts batches of up to 2,000 events; each event requires `event` name, `time`, `distinct_id`, and `$insert_id` (for deduplication and safe retry)
- **Authentication:** Project API key and secret; Basic Auth

### Pendo

- **Description:** Product analytics and in-app guidance platform; particularly relevant to CS platforms because Pendo tracks feature-level adoption per user, making it a high-signal input for health scoring.
- **API Documentation:** https://engageapi.pendo.io/ and https://www.pendo.io/developers/
- **Developer Documentation Directory:** https://support.pendo.io/hc/en-us/articles/38099922926875-Pendo-developer-documentation
- **Key APIs:**
  - Engage API: query core subscription data, manage guides, user segments, and metadata
  - Agent API: control the Pendo JS agent (fine-tune auto-event capture, trigger guides programmatically)
  - REST Aggregations API v1: query language for accessing Pendo data through aggregations
  - Feedback API: retrieve and manage user feedback requests (legacy)
- **Standards:** REST/JSON; integration key for authentication
- **Authentication:** Integration key created and managed by admins within the Pendo subscription

### Zendesk

- **Description:** The dominant customer support platform; CS platforms ingest Zendesk ticket data (volume, resolution time, CSAT, sentiment) as a health score input and alert signal.
- **API Documentation:** https://developer.zendesk.com/api-reference/
- **Webhooks Reference:** https://developer.zendesk.com/api-reference/webhooks/webhooks-api/webhooks/
- **Webhooks Guide:** https://developer.zendesk.com/documentation/webhooks/
- **Ticket Events:** https://developer.zendesk.com/api-reference/webhooks/event-types/ticket-events/
- **Standards:** REST/JSON; webhooks support both event subscription (user/organisation/help center activity) and trigger/automation connection (ticket activity); webhook payloads follow consistent JSON schema per event type
- **Authentication:** OAuth 2.0 and API token; webhook delivery authenticated via signing secret

### Slack

- **Description:** The dominant workplace messaging platform; CS platforms deliver real-time churn alerts, CTA notifications, and account health updates to CSM Slack channels.
- **API Documentation:** https://docs.slack.dev/
- **Events API:** https://docs.slack.dev/apis/events-api/
- **Incoming Webhooks:** https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks/
- **Standards:** REST/JSON; Events API replaces legacy outgoing webhooks; incoming webhooks send JSON payloads to a unique per-channel URL; Block Kit JSON format for structured message cards
- **Authentication:** OAuth 2.0 app authorisation; webhook URLs scoped to specific channels; signing secrets for verifying Events API delivery

---

## Notes

**No universal health score standard exists.** The Customer Health Score is the core data model of the CS platform category, yet no ISO, W3C, or industry body standard governs its structure, dimensions, or calculation methodology. Each platform (Gainsight, ChurnZero, Vitally) defines proprietary weighted scoring models. An open-source CS platform has the opportunity to publish a reference health score schema as an open standard — potentially accelerating adoption.

**OData v4 is underutilised in the CS category.** ChurnZero is the only surveyed platform that implements OData v4, giving its API native BI tool compatibility (Power BI, Tableau, Excel) without custom connectors. Gainsight and Vitally use proprietary REST endpoints. An open-source platform adopting OData for reporting endpoints would offer a significant integration advantage for revenue operations and analytics teams.

**The EU AI Act's applicability to churn prediction models is still being clarified.** As of May 2026, the Act's provisions for high-risk AI systems are beginning to apply. AI churn prediction models that inform customer-affecting decisions (escalation routing, cancellation risk flagging) may fall within scope, requiring transparency documentation, accuracy monitoring, and human oversight mechanisms. This is an evolving area and should be reviewed with legal counsel before deploying ML-based churn scoring in EU markets.

**CloudEvents adoption is emerging.** The CNCF CloudEvents v1.0 specification for portable event payloads is not yet adopted by any surveyed CS platform but is gaining traction in adjacent SaaS tooling. Adopting CloudEvents for webhook payloads in an open-source CS platform would improve interoperability with modern event-driven infrastructure (AWS EventBridge, Kafka, Knative).

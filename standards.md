# Standards & API Reference

> Project: Meeting Management Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022** — Information security management; governs the security of meeting recordings, transcripts, and agendas which often contain confidential business information; Annex A A.8.3 (Information access restriction) and A.8.11 (Data masking) are applicable. URL: https://www.iso.org/standard/82875.html

- **ISO/IEC 27018:2019 — Protection of PII in Public Clouds** — Governs processing of participant personal data (names, email addresses, voice recordings, biometric voiceprints) in cloud-hosted meeting management services; requires retention transparency and data minimisation. URL: https://www.iso.org/standard/76559.html

### W3C & IETF Standards

- **RFC 5545 — iCalendar (Internet Calendaring and Scheduling Core Object Specification)** — The foundational IETF standard for calendar data exchange (2009); defines VCALENDAR, VEVENT, VTODO, and VJOURNAL components; the universal format for meeting invitations (.ics files), calendar imports/exports, and scheduling interoperability across all meeting platforms. URL: https://datatracker.ietf.org/doc/html/rfc5545

- **RFC 7986 — iCalendar New Properties** — Extends RFC 5545 with new properties for conferencing systems (VIDEO/AUDIO/PHONE conference URIs), calendar name, description, and refresh interval; directly relevant to meeting management platform calendar invites including video conferencing links. URL: https://datatracker.ietf.org/doc/html/rfc7986

- **RFC 4791 — CalDAV: Calendaring Extensions to WebDAV** — IETF standard (2007) for real-time bidirectional calendar synchronisation between clients and servers; used by meeting management platforms for two-way sync with Google Calendar, Apple Calendar, and Exchange/Outlook calendar servers. URL: https://datatracker.ietf.org/doc/html/rfc4791

- **RFC 6638 — CalDAV Scheduling Extensions** — Extends CalDAV to support scheduling operations (organiser/attendee free/busy queries, meeting invitations, and acceptances) using iCalendar-based components; enables automated scheduling directly against CalDAV servers. URL: https://datatracker.ietf.org/doc/html/rfc6638

- **RFC 4918 — WebDAV (HTTP Extensions for Web Distributed Authoring)** — Foundation for CalDAV; defines PROPFIND, PROPPATCH, MKCOL, COPY, MOVE, LOCK, UNLOCK HTTP methods used in CalDAV calendar collection management. URL: https://datatracker.ietf.org/doc/html/rfc4918

- **RFC 6749 — OAuth 2.0** — Authorization framework used by all major calendar and meeting platforms (Google Calendar, Microsoft Graph, Calendly, Cronofy) for third-party access to calendar data and meeting scheduling on behalf of users. URL: https://datatracker.ietf.org/doc/html/rfc6749

- **RFC 7519 — JSON Web Token (JWT)** — Standard token format used in meeting management API authentication and calendar integration authorization flows. URL: https://datatracker.ietf.org/doc/html/rfc7519

- **RFC 2445 — iCalendar (Original)** — Original 1998 iCalendar specification (superseded by RFC 5545); still referenced in legacy enterprise systems; meeting management platforms must support both for backward compatibility. URL: https://datatracker.ietf.org/doc/html/rfc2445

### Data Model & API Specifications

- **OpenAPI 3.1** — Used by Calendly, Cronofy, Fireflies, and Fellow to document their REST APIs; enables SDK code generation and automated integration testing. URL: https://spec.openapis.org/oas/latest.html

- **Google Calendar API v3** — REST/JSON API for creating, reading, updating, and deleting calendar events; supports free/busy queries, attendee management, recurring events, and push notifications (webhooks); the most widely integrated calendar API for scheduling automation. URL: https://developers.google.com/workspace/calendar/api/v3/reference

- **Microsoft Graph API — Calendar Resources** — REST/JSON API replacing Exchange Web Services (EWS, deprecated October 1, 2026); provides full calendar event management, free/busy queries, meeting rooms, and online meeting creation (Teams, Skype) via Microsoft Graph; also supports Google Calendar interoperability via Graph API (2025). URL: https://learn.microsoft.com/en-us/graph/api/resources/calendar-overview

- **GraphQL** — Used by some meeting intelligence platforms for flexible querying of meeting transcripts, action items, and participant data. URL: https://graphql.org/

- **Fireflies GraphQL API** — Fireflies' primary API protocol for querying transcripts, meeting metadata, speaker information, and summaries; complemented by their MCP server integration. URL: https://docs.fireflies.ai/

### Security & Authentication Standards

- **GDPR Articles 6, 9, 32 — Lawful Processing, Biometric Data, and Security** — Meeting recordings and AI transcription require explicit consent; speaker recognition voiceprints are biometric data under GDPR Article 9 requiring explicit opt-in (March 2026 legal analysis); data residency in EU/EEA required for EU deployments; EDPB 2026 Coordinated Enforcement focuses on transparency obligations (Articles 12-14). URL: https://gdpr-info.eu/art-9-gdpr/

- **EU AI Act (2024/2847)** — AI transcription, automated summarisation, and action item extraction in meeting platforms are subject to transparency requirements (GPAI model obligations from August 2025); platforms must disclose AI use and provide opt-out options to participants. URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32024R2847

- **HIPAA — §164.312 — Technical Safeguards** — Meeting management platforms used for telehealth patient discussions must implement transmission security and access controls; requires BAAs with platform providers before processing PHI in meetings. URL: https://www.hhs.gov/hipaa/for-professionals/security/

- **SOC 2 Type II** — Required enterprise compliance certification for meeting management SaaS platforms; Calendly, Cronofy, and Fireflies maintain SOC 2 Type II reports for enterprise procurement. URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2

- **ISO 27001 / ISO 27017 / ISO 27701** — Cronofy holds ISO 27001, 27701, and 27018 certifications and is HIPAA and SOC 2 attested; represents the multi-framework compliance standard for enterprise calendar API providers. URL: https://www.cronofy.com/developer

- **SAML 2.0 / OIDC** — Required for enterprise SSO integration in meeting management platforms; ensures meeting access control is tied to enterprise identity lifecycle (provisioning, deprovisioning). URL: https://docs.oasis-open.org/security/saml/v2.0/

- **OWASP API Security Top 10 (2023)** — Governs API design for meeting management platforms; API1 (Broken Object Level Authorization) is critical for ensuring users can only access meetings they are authorized to view. URL: https://owasp.org/API-Security/

### MCP Server Specifications

Meeting management and AI meeting assistants have a rich and rapidly growing MCP ecosystem as of 2026:

- **Fireflies MCP Server (Official, Beta)** — Enables AI tools to connect directly to meeting transcript data; provides `fireflies_get_transcripts` (filter by date range) and `fireflies_get_transcript_details` (full content, speakers, metadata) tools; OAuth-based connection for Claude/ChatGPT or JSON config for Claude Desktop. URL: https://docs.fireflies.ai/getting-started/mcp-configuration

- **Fellow MCP Server (Official)** — Connects Fellow meeting notes to AI agents; 50+ native integrations; admin governance layer controlling which workspace members can connect; accesses only data user already has permission to view. URL: https://fellow.ai/

- **Otter.ai MCP Server (Bidirectional)** — Bidirectional MCP implementation: as a server, Otter exposes meeting history to AI tools (Claude, ChatGPT); as a client, Otter pulls context from external tools (Gmail, Google Drive, Notion, Salesforce) into AI chat. URL: https://otter.ai/

- **Granola MCP Server (February 2026)** — Enables AI agents to query Granola meeting notes via MCP in real time; Granola raised $125M Series C at $1.5B valuation in March 2026. URL: https://doolpa.com/news/granola-125m-series-c-1-5b-valuation-enterprise-march-2026

---

## Similar Products — Developer Documentation & APIs

### Google Calendar API

- **Description:** REST API for Google Calendar; enables creating, querying, and managing calendar events, free/busy lookups, resource calendars, and push notification webhooks; widely used as the primary calendar integration for meeting management platforms.
- **API Documentation:** https://developers.google.com/workspace/calendar/api/v3/reference
- **SDKs/Libraries:** Google API client libraries (Python, Java, .NET, Go, JavaScript, PHP, Ruby); google-auth libraries for OAuth 2.0
- **Developer Guide:** https://developers.google.com/workspace/calendar/api/guides/overview
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0 (Google Identity), iCalendar (RFC 5545) for event export
- **Authentication:** OAuth 2.0 (user authorization); Service accounts for domain-wide delegation; API keys (read-only public calendar data)

### Microsoft Graph API — Calendar

- **Description:** Microsoft Graph REST API for Outlook/Exchange calendar management; replaces Exchange Web Services (EWS, deprecated October 1, 2026); supports event management, free/busy queries, rooms/resources, and Teams online meeting creation; interoperability with Google Calendar via Graph API (2025).
- **API Documentation:** https://learn.microsoft.com/en-us/graph/api/resources/calendar-overview
- **SDKs/Libraries:** Microsoft Graph SDKs (Python, .NET, Go, Java, JavaScript/TypeScript)
- **Developer Guide:** https://learn.microsoft.com/en-us/graph/teams-concept-overview
- **Standards:** REST/JSON, OpenAPI 3.1, OAuth 2.0 (Azure AD), iCalendar (RFC 5545) for import/export
- **Authentication:** Azure AD OAuth 2.0 (delegated or application permissions); managed identity for Azure workloads

### Calendly

- **Description:** Leading meeting scheduling platform; eliminates scheduling back-and-forth by sharing availability pages; Scheduling API (2025) enables embedding scheduling flows without redirects; webhooks for booking events; 700+ integrations via Zapier and native connectors.
- **API Documentation:** https://developer.calendly.com/api-docs
- **Scheduling API Reference:** https://developer.calendly.com/api-docs/d7755e2f9e5fe-calendly-api
- **SDKs/Libraries:** REST API (JSON); Postman collection; community Python SDK (calendly-api-python)
- **Developer Guide:** https://developer.calendly.com/getting-started
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, Webhooks, iCalendar (RFC 5545) for calendar invites
- **Authentication:** OAuth 2.0 (personal access tokens for personal use; OAuth apps for integrations)

### Cal.com (Open Source)

- **Description:** Open-source (AGPL-3.0) scheduling platform with hosted cloud and self-hosted options; REST API for managing availability, event types, and bookings; Atom-based API design; strong in developer communities for embedding scheduling.
- **API Documentation:** https://cal.com/docs/api-reference/v2/introduction
- **SDKs/Libraries:** @calcom/sdk (TypeScript/JavaScript); REST API (v2 current); Cal Atoms (embeddable UI components)
- **Developer Guide:** https://cal.com/docs/
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, iCalendar (RFC 5545), CalDAV
- **Authentication:** API key; OAuth 2.0 for user-managed app integrations

### Cronofy

- **Description:** Enterprise-grade calendar synchronisation and scheduling API; unified API layer across Google Calendar, Microsoft 365/Exchange, Apple Calendar, and Zoom; ISO 27001/27701/27018 certified, HIPAA-compliant, SOC 2 attested; six regional data centres for data residency control.
- **API Documentation:** https://www.cronofy.com/developer/calendar-api
- **SDKs/Libraries:** cronofy-ruby; cronofy-python; cronofy-node; cronofy-java; cronofy-dotnet; cronofy-php
- **Developer Guide:** https://www.cronofy.com/developer
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, iCalendar (RFC 5545), CalDAV, CalConnect standards
- **Authentication:** OAuth 2.0 (per-user calendar access tokens); service account for enterprise

### Fireflies.ai

- **Description:** AI meeting assistant with transcription, summarisation, and CRM sync; 40+ native integrations; official MCP server (beta); GraphQL API for querying transcripts and meeting metadata; widely used for sales and customer success teams.
- **API Documentation:** https://docs.fireflies.ai/
- **MCP Configuration:** https://docs.fireflies.ai/getting-started/mcp-configuration
- **SDKs/Libraries:** GraphQL API; REST webhook events; community MCP server (props-labs/fireflies-mcp)
- **Developer Guide:** https://fireflies.ai/api
- **Standards:** GraphQL, REST/JSON (webhooks), OAuth 2.0
- **Authentication:** API key (Authorization Bearer); OAuth for MCP server connection

### Fellow.ai

- **Description:** Meeting management and AI meeting assistant platform focused on structured meeting agendas, notes, and action items; native integrations with Salesforce, HubSpot, Jira, Asana, Linear, Slack, Notion, and 50+ tools; official MCP server with admin governance.
- **API Documentation:** https://fellow.ai/developers/ (requires account)
- **SDKs/Libraries:** REST API; Webhooks; official MCP server
- **Developer Guide:** Fellow developer portal (account required)
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, iCalendar (calendar sync)
- **Authentication:** API token; OAuth 2.0 for third-party integrations

### Otter.ai

- **Description:** AI-powered meeting transcription and notes; OtterPilot auto-joins Zoom, Meet, and Teams from calendar; bidirectional MCP (server: exposes meeting history; client: pulls Gmail/Drive/Notion/Salesforce context into AI chat); live transcript sharing and collaborative notes.
- **API Documentation:** https://otter.ai/api (requires account)
- **SDKs/Libraries:** REST API; MCP server (bidirectional)
- **Developer Guide:** Otter developer portal (account required)
- **Standards:** REST/JSON, OAuth 2.0, iCalendar (calendar integration)
- **Authentication:** OAuth 2.0; API key for enterprise integrations

### Nylas Calendar API

- **Description:** Unified calendar API aggregating Google Calendar, Microsoft 365/Exchange, and Apple Calendar; REST API for events, availability, and scheduling; also provides email and contacts APIs in the same unified platform.
- **API Documentation:** https://developer.nylas.com/docs/calendar/
- **SDKs/Libraries:** nylas-python; nylas-node; nylas-java; nylas-ruby; nylas-go
- **Developer Guide:** https://developer.nylas.com/
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, iCalendar (RFC 5545)
- **Authentication:** OAuth 2.0 (per-user calendar access); API key for management

---

## Notes

- **EWS deprecation (October 1, 2026)**: Microsoft Exchange Web Services is being retired on October 1, 2026; all meeting management integrations with Exchange/Outlook must migrate to Microsoft Graph API before this date — a major forcing function for platform upgrades.

- **iCalendar (RFC 5545) as the universal format**: All calendar systems (Google, Microsoft, Apple) produce and consume RFC 5545-compliant .ics files; meeting management platforms must implement full iCalendar support including VTIMEZONE, VALARM, and attendee RSVP handling.

- **AI meeting data as biometric data (2026)**: GDPR Article 9 applies to speaker recognition features; meeting platforms with AI speaker ID must implement explicit opt-in consent, compliant DPAs, and data retention limits. This is an active regulatory risk area.

- **MCP as the meeting intelligence integration layer (2026)**: Fireflies, Fellow, Otter, and Granola all have official MCP servers; meeting transcript data is becoming a primary knowledge source for AI agents. The bidirectional Otter pattern (server + client) is an emerging architecture for contextually-aware AI meeting assistants.

- **Open-source landscape**: Cal.com (AGPL-3.0) is the leading open-source scheduling platform; there are no dominant open-source AI meeting assistant alternatives comparable to Fireflies or Otter; self-hosted transcription can be built using OpenAI Whisper (MIT) for the audio processing layer.

- **CalConnect standards**: CalConnect (the Calendaring and Scheduling Consortium) publishes additional specifications and profiles beyond the core IETF standards; relevant for interoperability testing and enterprise calendar integration compliance.

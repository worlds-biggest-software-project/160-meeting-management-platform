# Meeting Management Platform

> Candidate #160 · Researched: 2026-05-02

## Existing Products and Software Packages

| Tool | Description | Type | Pricing | Strengths / Weaknesses |
|------|-------------|------|---------|------------------------|
| Fellow | Full meeting lifecycle tool covering agendas, note-taking, action items, follow-ups, analytics, and calendar integration | SaaS | Free; Pro USD 9/user/month; Business USD 14; Enterprise custom | Most complete end-to-end meeting OS; strong manager/team workflow; less strong for ad-hoc meetings |
| Fireflies.ai | AI note-taker with broad CRM integrations (HubSpot, Salesforce, Pipedrive, Zoho); crossed USD 1B valuation in 2026 | SaaS | Free tier; Pro USD 10/user/month; Business USD 19 | Best-in-class CRM sync; strong search over past meetings; limited pre-meeting agenda tooling |
| Otter.ai | Real-time transcription and meeting notes with Otter Meeting Agent for autonomous in-meeting actions | SaaS | Free; Pro USD 16.99/user/month; Business USD 30 | Pioneer in real-time transcription; AI agent can respond in meetings autonomously; accuracy varies |
| Fathom | Free AI note-taker with unlimited recording; integrates with Zoom, Teams, and Google Meet; 7-language support | SaaS/freemium | Free; paid plans from USD 20/month | Best free tier in the category; limited agenda and pre-meeting features |
| Jamie | Privacy-focused note-taker that works offline and without a meeting bot; syncs to HubSpot, Salesforce, Notion | SaaS | Free (limited); Pro ~USD 24/month | No-bot approach appeals to privacy-sensitive teams; offline processing unique in the market |
| Granola | Device-audio capture that works with any video conferencing tool; no bot joins the call | SaaS | Subscription (pricing varies) | Universal conferencing compatibility; limited integrations versus competitors |
| Notion Calendar | Calendar-native meeting management with linked Notion docs, agenda templates, and context pull-in | SaaS | Bundled with Notion | Tight Notion integration; weak standalone note-taking and AI features |

## Relevant Industry Standards or Protocols

- **iCalendar (RFC 5545) / CalDAV** — standards for calendar data interchange and synchronisation, foundational for agenda builder and calendar sync features
- **OAuth 2.0** — authorisation standard enabling secure calendar (Google, Microsoft) and CRM (Salesforce, HubSpot) integrations without credential sharing
- **WebVTT / SRT (SubRip)** — subtitle and transcript format standards for interoperable export of meeting transcriptions and captions
- **GDPR / CCPA compliance frameworks** — data residency and consent requirements for recording, transcription, and storage of meeting content involving EU and California participants
- **WebRTC** — underlies in-browser meeting capture capabilities for tools that process audio natively rather than via a bot participant

## Available Research Materials

1. Reclaim AI (2026). *Top 18 AI Meeting Assistants & Note Takers of 2026*. https://reclaim.ai/blog/ai-meeting-assistants
2. Zapier (2026). *The 10 Best AI Meeting Assistants in 2026*. https://zapier.com/blog/best-ai-meeting-assistant/
3. Fellow.ai (2026). *The 22 Best AI Meeting Assistants & AI Notetakers for 2026: Ultimate Guide*. https://fellow.ai/blog/ai-meeting-assistants-ultimate-guide/
4. AssemblyAI (2026). *Top 10 AI Notetakers in 2026: Compare Features, Pricing, and Accuracy*. https://www.assemblyai.com/blog/top-ai-notetakers
5. Lindy.ai (2026). *Top 12 Meeting Minutes Apps: Tested & Reviewed for 2026*. https://www.lindy.ai/blog/best-meeting-minutes-app
6. Outdoo.ai (2026). *Fireflies AI Review 2026: Pros, Cons, Pricing & Top Alternatives*. https://www.outdoo.ai/blog/fireflies-ai-review
7. Cirrus Insight (2026). *Best AI Meeting Note Takers in 2026: Top Tools Compared*. https://www.cirrusinsight.com/blog/ai-meeting-note-takers

## Market Research

**Market Size:** The meeting management and AI note-taker segment is a fast-growing sub-market within the broader USD 12 billion video conferencing and collaboration market. Dedicated market sizing figures are not uniformly reported; however Fireflies.ai crossed a USD 1 billion valuation in 2026, indicating significant market confidence in the standalone AI note-taker category.

**Funding:** Fireflies.ai reached a USD 1 billion valuation in 2026. Otter.ai raised USD 20 million in a 2021 Series B. Fellow raised USD 25 million in a 2022 Series B. Fathom operates as a capital-efficient freemium business with undisclosed funding.

**Pricing Landscape:** Free tiers are common (Fathom, Fireflies free, Otter free) as user acquisition tools. Paid individual plans cluster around USD 10–20/user/month. Business tiers with CRM integrations, admin controls, and extended storage run USD 19–30/user/month. Enterprise contracts are custom. The "unlimited free recording" approach pioneered by Fathom is applying pricing pressure across the category.

**Key Buyer Personas:** Sales representatives and account executives seeking automatic CRM logging of customer calls; engineering and product managers tracking decisions and action items across recurring standups and planning meetings; HR and L&D teams recording and indexing training sessions; executives wanting searchable records of board and leadership meetings.

**Notable Trends:** The AI note-taker market reached mainstream adoption in 2026, with most knowledge workers having access to at least one such tool. The next competitive frontier is coordination — AI assistants that handle real calendar tradeoffs (priorities, focus blocks, time zones, prep time) rather than only documenting meetings. Privacy-first approaches (no bot, offline processing) are gaining traction with legal, healthcare, and government buyers. CRM auto-logging is the most commercially valuable integration for sales-motion buyers.

## AI-Native Opportunity

- Autonomous meeting agent that joins calls, takes notes, answers questions on behalf of participants, and follows up with action items without human prompting
- AI agenda builder that pulls context from previous meeting notes, open action items, and linked project tasks to suggest a structured agenda before each recurring meeting
- Longitudinal decision tracking: AI surfaces past decisions and commitments relevant to current agenda items, preventing repeated discussions of already-resolved questions
- Sentiment and engagement analysis during meetings, providing managers with post-call coaching insights on participation balance and discussion quality
- Cross-meeting knowledge synthesis: AI generates a weekly summary of key decisions, action items, and emerging themes across all meetings a person or team attended

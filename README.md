# Meeting Management Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native meeting operating system that combines agenda building, automatic note-taking, action item tracking, and calendar sync into a single platform.

The Meeting Management Platform is an end-to-end tool for the full meeting lifecycle — pre-meeting preparation, in-meeting capture, and post-meeting follow-up. It is aimed at engineering and product teams, sales organisations, managers, and privacy-sensitive industries who currently stitch together transcription tools, calendar apps, and CRMs to keep meetings productive. The core problem it solves is fragmentation: today's incumbents each address part of the workflow but leave coordination, longitudinal decision tracking, and privacy-sensitive deployment as unmet needs.

---

## Why Meeting Management Platform?

- **No strong open-source alternative exists.** The category is dominated by proprietary SaaS — Fellow, Fireflies.ai, Otter.ai, Fathom, Jamie, Granola, Notion Calendar — leaving organisations with no auditable, self-hostable option.
- **Pricing pressure on incumbents is real, but locked behind paid tiers.** Paid individual plans cluster around USD 10–20/user/month and business tiers run USD 19–30/user/month; Fathom's "unlimited free recording" approach has begun applying pressure across the category.
- **Most tools are note-takers, not meeting platforms.** Fireflies, Otter, Fathom, Jamie, and Granola focus on transcription and summaries; pre-meeting agenda tooling and longitudinal decision tracking remain weak.
- **Privacy-sensitive sectors are underserved.** Legal, healthcare, and government buyers need air-gapped or offline-first deployment; only Jamie's no-bot approach partially addresses this, and it lacks broader analytics.
- **Coordination — not documentation — is the next frontier.** The 2026 trend is AI assistants that handle real calendar tradeoffs (priorities, focus blocks, time zones, prep time) rather than only documenting meetings.

---

## Key Features

### Capture & Transcription

- Automatic transcription with target 90%+ accuracy and support for 10+ languages
- Recording support with audio/video capture and access controls
- Multi-platform integration across Zoom, Google Meet, Microsoft Teams, and Slack
- Bot-optional recording mode for privacy-sensitive environments

### Notes, Summaries & Action Items

- AI-generated meeting summaries (overview, bullet points, action items)
- Automatic action item identification, listing, and tracking
- Action item assignment and follow-up to named team members
- Full-text search across meeting history with timestamps

### Pre-Meeting Preparation

- Agenda builder for creating and sharing agendas before meetings
- Context pull-in from previous meeting notes, open action items, and linked project tasks
- Calendar integration with Google Calendar and Outlook for scheduling context

### Integrations & Sync

- CRM auto-logging into Salesforce and HubSpot
- Project management sync with Asana, Jira, and Monday.com
- Conversational AI chat interface to ask questions about meeting content
- Speaker analytics and sentiment analysis across recurring meetings

### Privacy, Coaching & Knowledge Synthesis (Backlog)

- Privacy-first deployment with offline processing and bot-optional recording
- Meeting quality coaching using engagement and participation analysis
- Longitudinal decision tracking surfacing past decisions relevant to current items
- Weekly LLM-generated synthesis of decisions and emerging themes across meetings
- Specialised vertical agents for sales (SDR qualification) and recruiting (candidate assessment)

---

## AI-Native Advantage

Where incumbents focus on transcribing and summarising one meeting at a time, this project treats meetings as a connected knowledge graph. An autonomous meeting agent can join calls, take notes, answer questions, and handle follow-ups without human prompting. An AI agenda builder pulls context from previous meeting notes, open action items, and linked project tasks to suggest a structured agenda before each recurring meeting. Longitudinal decision tracking surfaces past decisions and commitments relevant to current discussion, preventing re-litigation of resolved questions, and cross-meeting knowledge synthesis generates weekly summaries of key decisions and emerging themes across everything a person or team attended.

---

## Tech Stack & Deployment

The platform is designed around open standards: **iCalendar (RFC 5545) / CalDAV** for calendar interchange, **OAuth 2.0** for calendar and CRM authorisation, **WebVTT / SRT** for interoperable transcript export, and **WebRTC** for in-browser meeting capture without a bot participant. Compliance with **GDPR / CCPA** frameworks is treated as a first-class requirement for recording, transcription, and storage of meeting content. Deployment modes target both cloud and self-hosted use, including offline-first / air-gapped configurations for privacy-maximalist buyers. Integrations with Zoom, Google Meet, Microsoft Teams, Slack, Salesforce, HubSpot, Asana, Jira, Monday.com, and Notion are first-class, with public API and Zapier-style automation paths exposed for custom workflows.

---

## Market Context

The meeting management and AI note-taker segment is a fast-growing sub-market within the broader USD 12 billion video conferencing and collaboration market; Fireflies.ai crossed a USD 1 billion valuation in 2026, indicating significant market confidence in the standalone AI note-taker category. Incumbent pricing clusters at USD 10–20/user/month for individual paid plans and USD 19–30/user/month for business tiers with CRM integrations and admin controls. Primary buyers include sales representatives and account executives seeking automatic CRM logging, engineering and product managers tracking decisions and action items across recurring meetings, HR and L&D teams indexing training sessions, and executives wanting searchable records of board and leadership meetings.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.

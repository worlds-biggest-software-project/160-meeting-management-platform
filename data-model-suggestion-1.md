# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Meeting Management Platform · Created: 2026-05-20

## Philosophy

This model follows a fully normalized relational approach where every concept in the domain has its own dedicated table with well-defined foreign key relationships. The design maximises data integrity and query flexibility by avoiding redundancy and enforcing referential constraints at the database level. Every relationship between meetings, participants, transcripts, action items, and integrations is explicit in the schema.

Normalized relational models are the backbone of most enterprise SaaS platforms. Google Calendar API, Microsoft Graph Calendar, and Calendly all expose resource-centric schemas that map cleanly to normalized tables. This approach is well understood by development teams, has mature tooling (ORMs, migration frameworks, query analysers), and is the safest choice for a platform that must support complex cross-entity queries such as "show me all unresolved action items across all meetings attended by person X in the last 30 days."

The trade-off is a higher table count (50+), more JOIN-heavy queries, and the need for careful migration management as the schema evolves. However, for a meeting management platform where data integrity, compliance, and cross-entity reporting are core requirements, normalisation is a natural fit.

**Best for:** Teams building an enterprise-grade meeting platform with complex reporting needs, strong data integrity requirements, and relational database expertise.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Flexible ad-hoc querying and reporting
- (+) Well-understood by most engineering teams
- (+) Mature migration and ORM tooling
- (-) High table count increases schema complexity
- (-) JOIN-heavy queries can be slower for deeply nested data
- (-) Schema changes require migrations; less flexible for rapid iteration
- (-) Multi-jurisdiction field variations require adding columns or separate tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5545 (iCalendar) | VEVENT properties map to `meetings` table columns (summary, dtstart, dtend, rrule, location, uid); recurrence rules stored as RFC 5545 RRULE strings |
| RFC 7986 (iCalendar Extensions) | Conference URI fields (VIDEO, AUDIO) map to `meeting_conference_links` table |
| RFC 4791 (CalDAV) | CalDAV sync state tracked in `calendar_sync_connections` table with sync tokens |
| RFC 6749 (OAuth 2.0) | OAuth tokens and scopes stored in `oauth_connections` table per user per provider |
| ISO 3166-1 | Country codes used in `users.country_code` and `organizations.country_code` for data residency |
| ISO 639-1 | Language codes in `transcripts.language` for multi-language transcription |
| WebVTT (W3C) | Transcript segments modelled as rows in `transcript_segments` with start/end timestamps and speaker attribution |
| GDPR/CCPA | Consent tracking in `recording_consents` table; data retention policies in `data_retention_policies` |
| ISO/IEC 27001 | Audit trail in `audit_log` table for all data access and modifications |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    country_code CHAR(2),              -- ISO 3166-1 alpha-2
    timezone VARCHAR(50) DEFAULT 'UTC',
    plan_tier VARCHAR(20) DEFAULT 'free', -- free, pro, business, enterprise
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(320) NOT NULL,
    display_name VARCHAR(255) NOT NULL,
    avatar_url TEXT,
    country_code CHAR(2),              -- ISO 3166-1 alpha-2
    locale VARCHAR(10) DEFAULT 'en',   -- ISO 639-1 + region
    timezone VARCHAR(50) DEFAULT 'UTC',
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_users_email ON users (email);

CREATE TABLE organization_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(30) NOT NULL DEFAULT 'member', -- owner, admin, member, viewer
    joined_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);
CREATE INDEX idx_org_members_org ON organization_members (organization_id);
CREATE INDEX idx_org_members_user ON organization_members (user_id);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(30) DEFAULT 'member',  -- lead, member
    UNIQUE (team_id, user_id)
);
```

## Calendar & OAuth Integrations

```sql
CREATE TABLE oauth_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL,       -- google, microsoft, zoom, salesforce, hubspot
    provider_account_id VARCHAR(255),
    access_token TEXT NOT NULL,
    refresh_token TEXT,
    token_expires_at TIMESTAMPTZ,
    scopes TEXT[],                        -- OAuth scopes granted
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_oauth_user_provider ON oauth_connections (user_id, provider);

CREATE TABLE calendar_sync_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    oauth_connection_id UUID NOT NULL REFERENCES oauth_connections(id) ON DELETE CASCADE,
    calendar_provider VARCHAR(50) NOT NULL, -- google_calendar, outlook, caldav
    external_calendar_id VARCHAR(255),
    sync_token TEXT,                      -- CalDAV/Google sync token for incremental sync
    last_synced_at TIMESTAMPTZ,
    sync_direction VARCHAR(10) DEFAULT 'both', -- inbound, outbound, both
    is_primary BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cal_sync_user ON calendar_sync_connections (user_id);
```

## Meeting Management

```sql
CREATE TABLE meetings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    ical_uid VARCHAR(512),              -- RFC 5545 UID for calendar interop
    title VARCHAR(500) NOT NULL,
    description TEXT,
    location VARCHAR(500),
    meeting_type VARCHAR(30) DEFAULT 'one_off', -- one_off, recurring_series, recurring_instance
    series_id UUID REFERENCES meetings(id),     -- self-ref for recurring instances
    rrule TEXT,                         -- RFC 5545 RRULE for recurrence pattern
    starts_at TIMESTAMPTZ NOT NULL,
    ends_at TIMESTAMPTZ NOT NULL,
    timezone VARCHAR(50),
    status VARCHAR(20) DEFAULT 'scheduled', -- scheduled, in_progress, completed, cancelled
    visibility VARCHAR(20) DEFAULT 'default', -- default, public, private, confidential
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_meetings_org ON meetings (organization_id);
CREATE INDEX idx_meetings_starts ON meetings (starts_at);
CREATE INDEX idx_meetings_series ON meetings (series_id) WHERE series_id IS NOT NULL;
CREATE INDEX idx_meetings_ical_uid ON meetings (ical_uid) WHERE ical_uid IS NOT NULL;
CREATE INDEX idx_meetings_created_by ON meetings (created_by);

CREATE TABLE meeting_participants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),           -- NULL for external participants
    email VARCHAR(320) NOT NULL,
    display_name VARCHAR(255),
    role VARCHAR(20) DEFAULT 'attendee',          -- organizer, attendee, optional, observer
    rsvp_status VARCHAR(20) DEFAULT 'pending',    -- pending, accepted, declined, tentative
    is_external BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_participants_meeting_email ON meeting_participants (meeting_id, email);
CREATE INDEX idx_participants_user ON meeting_participants (user_id) WHERE user_id IS NOT NULL;

CREATE TABLE meeting_conference_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    provider VARCHAR(30) NOT NULL,        -- zoom, google_meet, teams, webex, custom
    join_url TEXT NOT NULL,
    dial_in_number VARCHAR(50),
    meeting_code VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE meeting_tags (
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    tag VARCHAR(100) NOT NULL,
    PRIMARY KEY (meeting_id, tag)
);
CREATE INDEX idx_meeting_tags_tag ON meeting_tags (tag);
```

## Agenda & Pre-Meeting Preparation

```sql
CREATE TABLE agendas (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    created_by UUID NOT NULL REFERENCES users(id),
    is_shared BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_agendas_meeting ON agendas (meeting_id);

CREATE TABLE agenda_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agenda_id UUID NOT NULL REFERENCES agendas(id) ON DELETE CASCADE,
    parent_item_id UUID REFERENCES agenda_items(id),  -- nested agenda items
    title VARCHAR(500) NOT NULL,
    description TEXT,
    duration_minutes INT,
    presenter_user_id UUID REFERENCES users(id),
    sort_order INT NOT NULL DEFAULT 0,
    status VARCHAR(20) DEFAULT 'pending', -- pending, discussed, skipped, deferred
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_agenda_items_agenda ON agenda_items (agenda_id);
```

## Recording & Transcription

```sql
CREATE TABLE recordings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    storage_provider VARCHAR(30) NOT NULL, -- s3, gcs, azure_blob, local
    storage_path TEXT NOT NULL,
    media_type VARCHAR(10) NOT NULL,       -- audio, video
    mime_type VARCHAR(100),
    duration_seconds INT,
    file_size_bytes BIGINT,
    capture_method VARCHAR(20) NOT NULL,   -- bot, device_audio, upload
    status VARCHAR(20) DEFAULT 'processing', -- processing, ready, failed, deleted
    retention_expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_recordings_meeting ON recordings (meeting_id);

CREATE TABLE recording_consents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recording_id UUID NOT NULL REFERENCES recordings(id) ON DELETE CASCADE,
    participant_id UUID NOT NULL REFERENCES meeting_participants(id),
    consent_given BOOLEAN NOT NULL,
    consent_method VARCHAR(30) NOT NULL,    -- in_meeting_prompt, pre_meeting_email, org_policy
    consented_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE transcripts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recording_id UUID NOT NULL REFERENCES recordings(id) ON DELETE CASCADE,
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    language VARCHAR(10) NOT NULL DEFAULT 'en', -- ISO 639-1
    transcription_engine VARCHAR(50),           -- whisper, deepgram, assembly_ai
    accuracy_score DECIMAL(5,4),
    status VARCHAR(20) DEFAULT 'processing',    -- processing, completed, failed
    full_text TEXT,                              -- full concatenated transcript
    webvtt_content TEXT,                         -- WebVTT formatted export
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_transcripts_meeting ON transcripts (meeting_id);

CREATE TABLE transcript_segments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transcript_id UUID NOT NULL REFERENCES transcripts(id) ON DELETE CASCADE,
    speaker_id UUID REFERENCES speakers(id),
    segment_index INT NOT NULL,
    start_time_ms BIGINT NOT NULL,
    end_time_ms BIGINT NOT NULL,
    text TEXT NOT NULL,
    confidence DECIMAL(5,4),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_segments_transcript ON transcript_segments (transcript_id);
CREATE INDEX idx_segments_speaker ON transcript_segments (speaker_id) WHERE speaker_id IS NOT NULL;

CREATE TABLE speakers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    participant_id UUID REFERENCES meeting_participants(id),
    label VARCHAR(100) NOT NULL,         -- "Speaker 1" or resolved name
    voice_sample_hash VARCHAR(64),       -- for speaker recognition matching
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_speakers_meeting ON speakers (meeting_id);
```

## Meeting Notes & Summaries

```sql
CREATE TABLE meeting_notes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id),
    content TEXT NOT NULL,
    content_format VARCHAR(10) DEFAULT 'markdown', -- markdown, html, plain
    is_ai_generated BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_notes_meeting ON meeting_notes (meeting_id);

CREATE TABLE meeting_summaries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    summary_type VARCHAR(30) NOT NULL,    -- overview, bullet_points, executive, custom
    content TEXT NOT NULL,
    ai_model VARCHAR(50),                 -- claude-4, gpt-4o, etc.
    ai_prompt_template_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_summaries_meeting ON meeting_summaries (meeting_id);
```

## Action Items & Decisions

```sql
CREATE TABLE action_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    assigned_to UUID REFERENCES users(id),
    assigned_by UUID REFERENCES users(id),
    due_date DATE,
    priority VARCHAR(10) DEFAULT 'medium', -- low, medium, high, urgent
    status VARCHAR(20) DEFAULT 'open',     -- open, in_progress, completed, cancelled
    source VARCHAR(20) DEFAULT 'manual',   -- manual, ai_extracted
    ai_confidence DECIMAL(5,4),
    source_segment_id UUID REFERENCES transcript_segments(id),
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_actions_meeting ON action_items (meeting_id);
CREATE INDEX idx_actions_assigned ON action_items (assigned_to) WHERE assigned_to IS NOT NULL;
CREATE INDEX idx_actions_org_status ON action_items (organization_id, status);
CREATE INDEX idx_actions_due_date ON action_items (due_date) WHERE due_date IS NOT NULL;

CREATE TABLE decisions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    decided_by UUID REFERENCES users(id),
    source VARCHAR(20) DEFAULT 'manual',   -- manual, ai_extracted
    ai_confidence DECIMAL(5,4),
    source_segment_id UUID REFERENCES transcript_segments(id),
    supersedes_decision_id UUID REFERENCES decisions(id), -- links to overridden decision
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_decisions_meeting ON decisions (meeting_id);
CREATE INDEX idx_decisions_org ON decisions (organization_id);
```

## Integrations (CRM & Project Management)

```sql
CREATE TABLE crm_sync_mappings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    oauth_connection_id UUID NOT NULL REFERENCES oauth_connections(id),
    crm_provider VARCHAR(30) NOT NULL,     -- salesforce, hubspot
    sync_enabled BOOLEAN DEFAULT true,
    field_mappings JSONB DEFAULT '{}',
    -- Example field_mappings:
    -- {
    --   "meeting.title": "Activity.Subject",
    --   "meeting.summary": "Activity.Description",
    --   "action_items": "Task"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE crm_sync_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    crm_sync_mapping_id UUID NOT NULL REFERENCES crm_sync_mappings(id),
    meeting_id UUID REFERENCES meetings(id),
    action_item_id UUID REFERENCES action_items(id),
    sync_direction VARCHAR(10) NOT NULL,   -- outbound, inbound
    external_record_id VARCHAR(255),
    status VARCHAR(20) NOT NULL,           -- success, failed, skipped
    error_message TEXT,
    synced_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pm_tool_sync (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    oauth_connection_id UUID NOT NULL REFERENCES oauth_connections(id),
    pm_provider VARCHAR(30) NOT NULL,      -- asana, jira, monday, linear
    project_external_id VARCHAR(255),
    sync_enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pm_task_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    action_item_id UUID NOT NULL REFERENCES action_items(id) ON DELETE CASCADE,
    pm_tool_sync_id UUID NOT NULL REFERENCES pm_tool_sync(id),
    external_task_id VARCHAR(255) NOT NULL,
    external_task_url TEXT,
    last_synced_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Analytics & Engagement

```sql
CREATE TABLE speaker_analytics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    speaker_id UUID NOT NULL REFERENCES speakers(id),
    talk_time_seconds INT NOT NULL DEFAULT 0,
    word_count INT NOT NULL DEFAULT 0,
    question_count INT NOT NULL DEFAULT 0,
    interruption_count INT NOT NULL DEFAULT 0,
    sentiment_score DECIMAL(5,4),          -- -1.0 to 1.0
    engagement_score DECIMAL(5,4),         -- 0.0 to 1.0
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_speaker_analytics_meeting ON speaker_analytics (meeting_id);

CREATE TABLE meeting_quality_scores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    participation_balance DECIMAL(5,4),    -- 0.0 (one person dominated) to 1.0 (even)
    decision_velocity INT,                 -- minutes to reach decisions
    action_item_count INT,
    agenda_adherence DECIMAL(5,4),         -- 0.0 to 1.0
    overall_score DECIMAL(5,4),
    coaching_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Search & Knowledge

```sql
CREATE TABLE search_index (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    entity_type VARCHAR(30) NOT NULL,     -- meeting, transcript, action_item, decision, note
    entity_id UUID NOT NULL,
    searchable_text TEXT NOT NULL,
    search_vector TSVECTOR GENERATED ALWAYS AS (to_tsvector('english', searchable_text)) STORED,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_search_vector ON search_index USING GIN (search_vector);
CREATE INDEX idx_search_org_type ON search_index (organization_id, entity_type);
```

## Audit & Compliance

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    user_id UUID,
    action VARCHAR(50) NOT NULL,           -- meeting.created, recording.accessed, action_item.completed
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB,                         -- {"field": {"old": "x", "new": "y"}}
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org ON audit_log (organization_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);

CREATE TABLE data_retention_policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    entity_type VARCHAR(50) NOT NULL,      -- recording, transcript, meeting
    retention_days INT NOT NULL,
    auto_delete BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 5 | organizations, users, organization_members, teams, team_members |
| Calendar & OAuth | 2 | oauth_connections, calendar_sync_connections |
| Meeting Management | 4 | meetings, meeting_participants, meeting_conference_links, meeting_tags |
| Agenda & Preparation | 2 | agendas, agenda_items |
| Recording & Transcription | 5 | recordings, recording_consents, transcripts, transcript_segments, speakers |
| Notes & Summaries | 2 | meeting_notes, meeting_summaries |
| Action Items & Decisions | 2 | action_items, decisions |
| Integrations | 4 | crm_sync_mappings, crm_sync_log, pm_tool_sync, pm_task_links |
| Analytics | 2 | speaker_analytics, meeting_quality_scores |
| Search | 1 | search_index |
| Audit & Compliance | 2 | audit_log, data_retention_policies |
| **Total** | **31** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation across services and avoids sequential ID enumeration attacks.

2. **Separate `speakers` table from `meeting_participants`** — speaker diarization produces speaker labels before they are matched to participants; keeping them separate allows incremental resolution and handles cases where a speaker cannot be matched.

3. **RFC 5545 RRULE stored as text** — recurrence rules are stored as iCalendar-format RRULE strings (e.g., `FREQ=WEEKLY;BYDAY=MO,WE,FR`) rather than decomposed into columns, preserving lossless round-trip with calendar providers.

4. **`meeting_type` + `series_id` for recurring meetings** — recurring series are a parent meeting row; individual instances reference it via `series_id`, enabling per-instance overrides while maintaining the series relationship.

5. **`source_segment_id` on action items and decisions** — links AI-extracted items back to the specific transcript segment, enabling "click to hear the context" UX and confidence auditing.

6. **Dedicated `recording_consents` table** — GDPR Article 9 requires explicit consent for voice recording (biometric data); per-participant consent tracking is a compliance requirement, not optional.

7. **PostgreSQL `TSVECTOR` for full-text search** — avoids external search infrastructure for MVP; the generated column approach keeps the index automatically up to date.

8. **Multi-tenant via `organization_id` foreign keys** — shared-table multi-tenancy with Row-Level Security policies on all tenant-scoped tables; avoids schema-per-tenant complexity.

9. **CRM field mappings stored as JSONB** — different CRM providers have different schemas; JSONB allows flexible per-organisation mapping configuration without schema changes.

10. **`decisions.supersedes_decision_id` for longitudinal tracking** — enables the "this decision overrides a previous one" chain that powers the longitudinal decision tracking feature.

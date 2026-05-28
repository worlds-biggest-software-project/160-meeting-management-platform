# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Meeting Management Platform · Created: 2026-05-20

## Philosophy

This model uses a pragmatic hybrid approach: stable, well-understood fields are stored as typed relational columns with proper constraints and indexes, while variable, integration-specific, and rapidly-evolving fields are stored in JSONB columns. The result is a schema that is both structured enough for reliable querying and flexible enough to accommodate the wide variation across calendar providers, CRM systems, conferencing platforms, and jurisdiction-specific requirements without constant migrations.

This pattern is common in modern SaaS platforms that integrate with many external systems. Stripe stores payment method details in JSONB alongside typed core fields. Shopify uses JSONB for metafields that vary by merchant. The approach is particularly well-suited to a meeting management platform because meetings sourced from Google Calendar have different metadata than those from Microsoft Graph or CalDAV, CRM field mappings vary between Salesforce and HubSpot, and AI-extracted insights may have evolving structures as models improve.

The key principle is: if you query it, filter by it, or join on it, make it a column. If it varies by provider, is display-only, or changes shape frequently, put it in JSONB. This gives you the best of both worlds at the cost of careful discipline about what goes where.

**Best for:** Rapid MVP development with multi-provider integrations, teams that need schema flexibility without sacrificing query performance on core fields, and platforms serving diverse organisations with varying requirements.

**Trade-offs:**
- (+) Fewer tables than fully normalised (lower cognitive overhead)
- (+) JSONB fields absorb provider-specific variation without migrations
- (+) Core fields remain typed, indexed, and queryable with SQL
- (+) Faster iteration: new metadata fields require no schema changes
- (+) GIN indexes on JSONB enable efficient containment queries
- (-) JSONB fields lack database-level type enforcement (rely on application validation)
- (-) JSONB contents are less discoverable than typed columns
- (-) Complex JSONB queries can be slower than equivalent relational JOINs
- (-) Risk of "JSONB junk drawer" if discipline is not maintained
- (-) Reporting tools may struggle with deeply nested JSONB

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5545 (iCalendar) | Core VEVENT properties (summary, dtstart, dtend, rrule) are typed columns; extended properties stored in `meetings.ical_properties` JSONB |
| RFC 7986 (iCalendar Extensions) | Conference links from different providers stored in `meetings.conference_data` JSONB |
| RFC 6749 (OAuth 2.0) | Provider-specific OAuth metadata stored in `integrations.provider_config` JSONB |
| ISO 3166-1 | Typed `country_code` column on organizations; jurisdiction-specific settings in `organizations.settings` JSONB |
| ISO 639-1 | Typed `language` column on transcripts; per-segment language detection in segment JSONB |
| WebVTT (W3C) | Transcript segments stored relationally; WebVTT voice tags and styling metadata in JSONB |
| GDPR/CCPA | Consent data stored relationally for queryability; jurisdiction-specific consent metadata in JSONB |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    country_code CHAR(2),
    timezone VARCHAR(50) DEFAULT 'UTC',
    plan_tier VARCHAR(20) DEFAULT 'free',
    settings JSONB DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_recording_method": "device_audio",
    --   "auto_transcribe": true,
    --   "transcript_language": "en",
    --   "retention_days": {"recordings": 90, "transcripts": 365, "meetings": -1},
    --   "ai_features": {"auto_summary": true, "action_item_extraction": true, "sentiment": false},
    --   "branding": {"logo_url": "...", "primary_color": "#1a73e8"},
    --   "compliance": {"require_recording_consent": true, "gdpr_mode": true}
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(320) UNIQUE NOT NULL,
    display_name VARCHAR(255) NOT NULL,
    avatar_url TEXT,
    locale VARCHAR(10) DEFAULT 'en',
    timezone VARCHAR(50) DEFAULT 'UTC',
    preferences JSONB DEFAULT '{}',
    -- preferences example:
    -- {
    --   "notification_channels": ["email", "slack"],
    --   "auto_join_recordings": true,
    --   "default_summary_type": "bullet_points",
    --   "calendar_view": "week",
    --   "quiet_hours": {"start": "18:00", "end": "09:00", "timezone": "America/New_York"}
    -- }
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(30) NOT NULL DEFAULT 'member',
    permissions JSONB DEFAULT '{}',
    -- permissions example (overrides for role):
    -- {
    --   "can_delete_recordings": true,
    --   "can_export_transcripts": true,
    --   "can_manage_integrations": false,
    --   "max_recording_hours_month": 100
    -- }
    joined_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);
CREATE INDEX idx_org_members_org ON organization_members (organization_id);
CREATE INDEX idx_org_members_user ON organization_members (user_id);
```

## Integrations (Calendar, CRM, PM Tools)

```sql
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL,
    provider_type VARCHAR(20) NOT NULL,    -- calendar, crm, pm_tool, communication
    provider_account_id VARCHAR(255),
    access_token_encrypted TEXT,
    refresh_token_encrypted TEXT,
    token_expires_at TIMESTAMPTZ,
    scopes TEXT[],
    is_active BOOLEAN DEFAULT true,
    provider_config JSONB DEFAULT '{}',
    -- provider_config example (Google Calendar):
    -- {
    --   "calendar_ids": ["primary", "team-meetings@group.calendar.google.com"],
    --   "sync_token": "CPDEgJ...",
    --   "webhook_channel_id": "uuid",
    --   "webhook_expiration": "2026-06-20T00:00:00Z",
    --   "sync_direction": "both",
    --   "color_mapping": {"#1a73e8": "external", "#34a853": "internal"}
    -- }
    -- provider_config example (Salesforce CRM):
    -- {
    --   "instance_url": "https://acme.my.salesforce.com",
    --   "api_version": "v59.0",
    --   "sync_objects": ["Activity", "Task", "Event"],
    --   "field_mapping": {
    --     "meeting.title": "Activity.Subject",
    --     "meeting.summary": "Activity.Description",
    --     "action_items[].title": "Task.Subject"
    --   },
    --   "auto_sync": true
    -- }
    -- provider_config example (Jira PM):
    -- {
    --   "site_url": "https://acme.atlassian.net",
    --   "project_key": "ENG",
    --   "issue_type_for_action_items": "Task",
    --   "auto_create_issues": false
    -- }
    last_synced_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_integrations_user ON integrations (user_id, provider_type);
CREATE INDEX idx_integrations_org ON integrations (organization_id, provider_type);
```

## Meetings

```sql
CREATE TABLE meetings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    -- Core fields (always queried, always present)
    title VARCHAR(500) NOT NULL,
    starts_at TIMESTAMPTZ NOT NULL,
    ends_at TIMESTAMPTZ NOT NULL,
    timezone VARCHAR(50),
    status VARCHAR(20) DEFAULT 'scheduled',
    meeting_type VARCHAR(30) DEFAULT 'one_off',
    series_id UUID REFERENCES meetings(id),
    rrule TEXT,
    created_by UUID NOT NULL REFERENCES users(id),

    -- Calendar interop
    ical_uid VARCHAR(512),
    external_calendar_event_id VARCHAR(255),
    source_integration_id UUID REFERENCES integrations(id),

    -- Flexible fields
    conference_data JSONB DEFAULT '{}',
    -- conference_data example:
    -- {
    --   "provider": "zoom",
    --   "join_url": "https://zoom.us/j/123456",
    --   "meeting_id": "123456",
    --   "passcode": "abc",
    --   "dial_in": ["+1-669-900-6833"],
    --   "host_key": "456789"
    -- }
    -- OR for Google Meet:
    -- {
    --   "provider": "google_meet",
    --   "join_url": "https://meet.google.com/abc-defg-hij",
    --   "conference_id": "abc-defg-hij"
    -- }

    ical_properties JSONB DEFAULT '{}',
    -- ical_properties: extended RFC 5545 properties not worth separate columns
    -- {
    --   "categories": ["planning", "sprint"],
    --   "priority": 5,
    --   "geo": {"lat": 37.7749, "lon": -122.4194},
    --   "x-custom-fields": {"department": "Engineering"}
    -- }

    metadata JSONB DEFAULT '{}',
    -- metadata: provider-specific data that varies widely
    -- {
    --   "google": {"hangout_link": "...", "color_id": "9", "etag": "..."},
    --   "microsoft": {"is_online_meeting": true, "online_meeting_provider": "teamsForBusiness"},
    --   "tags": ["weekly", "standup", "engineering"]
    -- }

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_meetings_org_starts ON meetings (organization_id, starts_at);
CREATE INDEX idx_meetings_status ON meetings (status, starts_at);
CREATE INDEX idx_meetings_series ON meetings (series_id) WHERE series_id IS NOT NULL;
CREATE INDEX idx_meetings_ical_uid ON meetings (ical_uid) WHERE ical_uid IS NOT NULL;
CREATE INDEX idx_meetings_tags ON meetings USING GIN ((metadata->'tags'));

CREATE TABLE meeting_participants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),
    email VARCHAR(320) NOT NULL,
    display_name VARCHAR(255),
    role VARCHAR(20) DEFAULT 'attendee',
    rsvp_status VARCHAR(20) DEFAULT 'pending',
    is_external BOOLEAN DEFAULT false,
    participation_data JSONB DEFAULT '{}',
    -- participation_data example:
    -- {
    --   "joined_at": "2026-05-20T09:01:00Z",
    --   "left_at": "2026-05-20T09:27:00Z",
    --   "talk_time_seconds": 420,
    --   "word_count": 850,
    --   "sentiment_score": 0.72,
    --   "interruption_count": 2,
    --   "questions_asked": 5
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (meeting_id, email)
);
CREATE INDEX idx_participants_meeting ON meeting_participants (meeting_id);
CREATE INDEX idx_participants_user ON meeting_participants (user_id) WHERE user_id IS NOT NULL;
```

## Agenda

```sql
CREATE TABLE agenda_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    parent_item_id UUID REFERENCES agenda_items(id),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    presenter_user_id UUID REFERENCES users(id),
    duration_minutes INT,
    sort_order INT NOT NULL DEFAULT 0,
    status VARCHAR(20) DEFAULT 'pending',
    extras JSONB DEFAULT '{}',
    -- extras example:
    -- {
    --   "linked_action_items": ["uuid1", "uuid2"],
    --   "linked_documents": [{"url": "https://...", "title": "Sprint Plan"}],
    --   "ai_suggested": true,
    --   "ai_context": "Based on 3 open action items from last week's meeting"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_agenda_items_meeting ON agenda_items (meeting_id, sort_order);
```

## Recording, Transcription & Notes

```sql
CREATE TABLE recordings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    capture_method VARCHAR(20) NOT NULL,
    media_type VARCHAR(10) NOT NULL,
    status VARCHAR(20) DEFAULT 'processing',
    duration_seconds INT,
    file_size_bytes BIGINT,
    storage_data JSONB NOT NULL,
    -- storage_data example:
    -- {
    --   "provider": "s3",
    --   "bucket": "meeting-recordings-us-east-1",
    --   "key": "org/uuid/meeting/uuid/recording.webm",
    --   "region": "us-east-1",
    --   "encryption": "AES-256",
    --   "content_type": "video/webm"
    -- }
    consent_status JSONB DEFAULT '{}',
    -- consent_status example:
    -- {
    --   "all_consented": false,
    --   "consents": [
    --     {"email": "jane@acme.com", "consented": true, "method": "in_meeting_prompt", "at": "2026-05-20T09:00:30Z"},
    --     {"email": "bob@acme.com", "consented": true, "method": "org_policy", "at": "2026-05-20T09:00:00Z"}
    --   ],
    --   "pending": ["external@partner.com"]
    -- }
    retention_expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_recordings_meeting ON recordings (meeting_id);

CREATE TABLE transcripts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    recording_id UUID REFERENCES recordings(id),
    language VARCHAR(10) NOT NULL DEFAULT 'en',
    engine VARCHAR(50),
    status VARCHAR(20) DEFAULT 'processing',
    accuracy_score DECIMAL(5,4),
    segment_count INT DEFAULT 0,
    full_text TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_transcripts_meeting ON transcripts (meeting_id);

CREATE TABLE transcript_segments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transcript_id UUID NOT NULL REFERENCES transcripts(id) ON DELETE CASCADE,
    segment_index INT NOT NULL,
    speaker_label VARCHAR(100),
    speaker_email VARCHAR(320),
    start_time_ms BIGINT NOT NULL,
    end_time_ms BIGINT NOT NULL,
    text TEXT NOT NULL,
    confidence DECIMAL(5,4),
    extras JSONB DEFAULT '{}',
    -- extras example:
    -- {
    --   "language_detected": "en",
    --   "words": [
    --     {"word": "I", "start_ms": 120000, "end_ms": 120200, "confidence": 0.99},
    --     {"word": "think", "start_ms": 120200, "end_ms": 120500, "confidence": 0.97}
    --   ],
    --   "is_question": true,
    --   "sentiment": "positive"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_segments_transcript ON transcript_segments (transcript_id, segment_index);
CREATE INDEX idx_segments_speaker ON transcript_segments (speaker_email) WHERE speaker_email IS NOT NULL;

CREATE TABLE meeting_notes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    author_id UUID REFERENCES users(id),
    content TEXT NOT NULL,
    content_format VARCHAR(10) DEFAULT 'markdown',
    is_ai_generated BOOLEAN DEFAULT false,
    ai_metadata JSONB DEFAULT '{}',
    -- ai_metadata example:
    -- {
    --   "model": "claude-4",
    --   "summary_type": "overview",
    --   "prompt_template": "meeting-summary-v3",
    --   "input_tokens": 4500,
    --   "output_tokens": 800,
    --   "generation_time_ms": 2100
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_notes_meeting ON meeting_notes (meeting_id);
```

## Action Items & Decisions

```sql
CREATE TABLE action_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    assigned_to_user_id UUID REFERENCES users(id),
    assigned_to_email VARCHAR(320),
    due_date DATE,
    priority VARCHAR(10) DEFAULT 'medium',
    status VARCHAR(20) DEFAULT 'open',
    source VARCHAR(20) DEFAULT 'manual',
    ai_confidence DECIMAL(5,4),
    source_segment_index INT,
    external_sync JSONB DEFAULT '{}',
    -- external_sync example:
    -- {
    --   "jira": {"issue_key": "ENG-1234", "issue_url": "https://acme.atlassian.net/browse/ENG-1234", "synced_at": "..."},
    --   "asana": {"task_id": "12345", "task_url": "https://app.asana.com/0/...", "synced_at": "..."},
    --   "salesforce": {"task_id": "00T...", "synced_at": "..."}
    -- }
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_actions_org_status ON action_items (organization_id, status);
CREATE INDEX idx_actions_assigned ON action_items (assigned_to_user_id, status) WHERE assigned_to_user_id IS NOT NULL;
CREATE INDEX idx_actions_due ON action_items (due_date) WHERE status != 'completed' AND due_date IS NOT NULL;

CREATE TABLE decisions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    decided_by_user_id UUID REFERENCES users(id),
    source VARCHAR(20) DEFAULT 'manual',
    ai_confidence DECIMAL(5,4),
    supersedes_decision_id UUID REFERENCES decisions(id),
    context JSONB DEFAULT '{}',
    -- context example:
    -- {
    --   "source_segment_index": 42,
    --   "related_decisions": ["uuid1", "uuid2"],
    --   "stakeholders": ["jane@acme.com", "bob@acme.com"],
    --   "category": "architecture",
    --   "tags": ["database", "postgresql"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_decisions_org ON decisions (organization_id);
CREATE INDEX idx_decisions_context_tags ON decisions USING GIN ((context->'tags'));
```

## Search & Analytics

```sql
CREATE TABLE search_index (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    entity_type VARCHAR(30) NOT NULL,
    entity_id UUID NOT NULL,
    meeting_id UUID,
    searchable_text TEXT NOT NULL,
    search_vector TSVECTOR GENERATED ALWAYS AS (to_tsvector('english', searchable_text)) STORED,
    facets JSONB DEFAULT '{}',
    -- facets example:
    -- {
    --   "date": "2026-05-20",
    --   "participants": ["jane@acme.com", "bob@acme.com"],
    --   "tags": ["standup", "engineering"],
    --   "has_recording": true,
    --   "has_action_items": true
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_search_vector ON search_index USING GIN (search_vector);
CREATE INDEX idx_search_org ON search_index (organization_id, entity_type);
CREATE INDEX idx_search_facets ON search_index USING GIN (facets);

CREATE TABLE meeting_analytics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    participation_balance DECIMAL(5,4),
    decision_count INT DEFAULT 0,
    action_item_count INT DEFAULT 0,
    total_duration_seconds INT,
    agenda_adherence DECIMAL(5,4),
    overall_quality_score DECIMAL(5,4),
    detailed_metrics JSONB DEFAULT '{}',
    -- detailed_metrics example:
    -- {
    --   "speaker_distribution": {"jane@acme.com": 0.35, "bob@acme.com": 0.40, "others": 0.25},
    --   "topic_distribution": [{"topic": "sprint planning", "pct": 0.6}, {"topic": "blockers", "pct": 0.3}],
    --   "engagement_over_time": [{"minute": 5, "score": 0.8}, {"minute": 10, "score": 0.6}],
    --   "coaching_suggestions": [
    --     "Bob spoke 40% of the time; consider encouraging others",
    --     "3 agenda items were skipped; consider shorter agendas"
    --   ]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_analytics_meeting ON meeting_analytics (meeting_id);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    user_id UUID,
    action VARCHAR(50) NOT NULL,
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB,
    request_context JSONB DEFAULT '{}',
    -- request_context example:
    -- {
    --   "ip_address": "192.168.1.1",
    --   "user_agent": "Mozilla/5.0...",
    --   "source": "web_app",
    --   "api_key_id": "uuid"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org_time ON audit_log (organization_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
```

## Example Queries

```sql
-- Find all meetings with Zoom conference links for an org
SELECT id, title, starts_at, conference_data->>'join_url' AS zoom_url
FROM meetings
WHERE organization_id = $1
  AND conference_data->>'provider' = 'zoom'
  AND starts_at > now()
ORDER BY starts_at;

-- Find all action items synced to Jira
SELECT ai.title, ai.status, ai.external_sync->'jira'->>'issue_key' AS jira_key
FROM action_items ai
WHERE ai.organization_id = $1
  AND ai.external_sync ? 'jira'
  AND ai.status != 'completed';

-- Find meetings tagged with "sprint" via JSONB containment
SELECT id, title, starts_at
FROM meetings
WHERE organization_id = $1
  AND metadata->'tags' @> '["sprint"]'::jsonb
ORDER BY starts_at DESC
LIMIT 20;

-- Find decisions by category
SELECT d.title, d.description, m.title AS meeting_title
FROM decisions d
JOIN meetings m ON m.id = d.meeting_id
WHERE d.organization_id = $1
  AND d.context->>'category' = 'architecture'
ORDER BY d.created_at DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organizations, users, organization_members |
| Integrations | 1 | Unified integrations table with JSONB provider_config |
| Meetings | 2 | meetings, meeting_participants (conference data in JSONB) |
| Agenda | 1 | agenda_items (flat with parent_item_id) |
| Recording & Transcription | 3 | recordings, transcripts, transcript_segments |
| Notes | 1 | meeting_notes (covers both human and AI-generated) |
| Action Items & Decisions | 2 | action_items, decisions |
| Search & Analytics | 2 | search_index, meeting_analytics |
| Audit | 1 | audit_log |
| **Total** | **16** | Significantly fewer than normalized model |

---

## Key Design Decisions

1. **Unified `integrations` table** -- instead of separate tables for calendar connections, CRM connections, and PM tool connections, a single table with `provider_type` and `provider_config` JSONB handles all providers. This dramatically reduces table count and makes adding new providers trivial.

2. **`conference_data` JSONB on meetings** -- Zoom, Google Meet, Teams, and Webex all have different conference metadata. JSONB absorbs this variation without a polymorphic table or separate conference link tables.

3. **`participation_data` JSONB on participants** -- analytics fields like talk_time, sentiment, and word_count are computed post-meeting and vary by what analytics features are enabled. JSONB avoids nullable columns on the participants table.

4. **`external_sync` JSONB on action items** -- an action item may be synced to zero, one, or multiple external tools (Jira AND Salesforce). JSONB with provider keys handles this cleanly without junction tables.

5. **`consent_status` JSONB on recordings** -- consent tracking is per-participant per-recording with method and timestamp; JSONB captures the variable-length consent array without a separate consent table.

6. **Word-level timestamps in segment `extras`** -- word-level timing data from transcription engines is voluminous and only needed for precise highlight/playback features. Storing in JSONB avoids a massive `transcript_words` table.

7. **`facets` JSONB on search_index** -- enables faceted search (filter by date, participants, tags) using GIN indexes on JSONB, avoiding separate facet tables or denormalised columns.

8. **Settings and preferences as JSONB** -- organization settings and user preferences are hierarchical and frequently extended. JSONB with application-layer validation is far more practical than a key-value settings table.

9. **Fewer tables, same query capability** -- 16 tables vs. 31 in the normalised model. The JSONB fields replace approximately 15 tables (conference_links, recording_consents, meeting_tags, crm_sync_mappings, crm_sync_log, pm_tool_sync, pm_task_links, speaker_analytics, meeting_quality_scores, etc.).

10. **JSONB discipline rule** -- the schema enforces a clear rule: if a field is used in WHERE clauses, JOINs, or aggregations, it must be a typed column. JSONB is for display, configuration, and provider-specific metadata only. The `metadata->'tags'` GIN index is the one intentional exception, where JSONB querying adds genuine value.

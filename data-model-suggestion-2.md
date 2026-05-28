# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Meeting Management Platform · Created: 2026-05-20

## Philosophy

This model treats the event store as the single source of truth. Every change to every entity -- a meeting being created, a participant joining, an action item being assigned, a transcript segment being added -- is recorded as an immutable event. Read-optimised materialised views (projections) are rebuilt from the event stream to serve queries. The Command Query Responsibility Segregation (CQRS) pattern separates the write path (commands that produce events) from the read path (projections that serve queries).

Event sourcing is used by systems where a complete audit trail is non-negotiable and temporal queries ("what was the state of this meeting on March 3rd?") are important. Financial trading platforms, healthcare record systems, and collaboration tools like Notion and Linear use event-sourcing principles internally. For a meeting management platform that handles GDPR-regulated recordings, needs to track who changed what and when, and wants to power AI analytics over change patterns, event sourcing provides a natural foundation.

The trade-off is increased system complexity: you need event serialisation, projection builders, eventual consistency handling, and a snapshot strategy for large aggregate streams. However, the benefits -- full audit trail for free, temporal queries, replay capability, and natural fit for AI analytics over meeting lifecycle events -- make this a strong choice for privacy-sensitive and enterprise-grade deployments.

**Best for:** Enterprise deployments requiring bulletproof audit trails, GDPR compliance evidence, temporal queries ("what decisions were active on date X?"), and AI-powered analytics over meeting lifecycle patterns.

**Trade-offs:**
- (+) Complete, immutable audit trail by design -- not bolted on
- (+) Temporal queries: reconstruct any entity's state at any point in time
- (+) Natural fit for AI analytics over change patterns and meeting lifecycle
- (+) Easy to add new read models without changing write logic
- (+) Event replay enables data migration, bug fixes, and new feature backfills
- (-) Higher system complexity: event store, projections, eventual consistency
- (-) Read model lag: projections are eventually consistent, not immediately consistent
- (-) Larger storage requirements: events are never deleted
- (-) Steeper learning curve for development teams unfamiliar with CQRS
- (-) Debugging requires understanding both events and projections

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5545 (iCalendar) | Calendar events serialised as event payloads containing iCalendar properties; RRULE stored in `MeetingSeriesCreated` events |
| RFC 6749 (OAuth 2.0) | OAuth token lifecycle tracked via `OAuthTokenGranted`, `OAuthTokenRefreshed`, `OAuthTokenRevoked` events |
| WebVTT (W3C) | Transcript segments emitted as `TranscriptSegmentAdded` events; WebVTT export rebuilt from projection |
| GDPR Articles 6, 9, 32 | Consent events (`RecordingConsentGranted`, `RecordingConsentRevoked`) are immutable evidence; right-to-erasure handled via crypto-shredding |
| ISO/IEC 27001 | The event store IS the audit log -- every action is an event with actor, timestamp, and payload |
| CloudEvents 1.0 | Event envelope follows CloudEvents specification for interoperability with external systems |

---

## Event Store

```sql
-- The single source of truth: all domain events
CREATE TABLE events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(50) NOT NULL,    -- Meeting, User, ActionItem, Transcript, etc.
    aggregate_id UUID NOT NULL,             -- ID of the entity this event belongs to
    event_type VARCHAR(100) NOT NULL,       -- MeetingCreated, ParticipantAdded, etc.
    event_version INT NOT NULL,             -- version within the aggregate stream
    payload JSONB NOT NULL,                 -- event-specific data
    metadata JSONB NOT NULL DEFAULT '{}',   -- actor_id, ip_address, correlation_id, causation_id
    -- metadata example:
    -- {
    --   "actor_id": "uuid",
    --   "actor_type": "user",          -- user, system, ai_agent
    --   "ip_address": "192.168.1.1",
    --   "correlation_id": "uuid",      -- traces a user action across multiple events
    --   "causation_id": "uuid",        -- the event that caused this event
    --   "organization_id": "uuid",
    --   "source": "web_app"            -- web_app, api, calendar_sync, ai_agent
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_type, aggregate_id, event_version)
);
CREATE INDEX idx_events_aggregate ON events (aggregate_type, aggregate_id, event_version);
CREATE INDEX idx_events_type ON events (event_type, created_at);
CREATE INDEX idx_events_org ON events ((metadata->>'organization_id'), created_at);
CREATE INDEX idx_events_created ON events (created_at);

-- Snapshots for aggregates with long event streams
CREATE TABLE snapshots (
    aggregate_type VARCHAR(50) NOT NULL,
    aggregate_id UUID NOT NULL,
    snapshot_version INT NOT NULL,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id, snapshot_version)
);
```

## Event Type Catalogue

Below are the key domain events. Each event's `payload` contains the data specific to that event.

```sql
-- ============================================================
-- MEETING LIFECYCLE EVENTS
-- ============================================================

-- MeetingCreated
-- payload: {
--   "title": "Weekly Standup",
--   "description": "...",
--   "starts_at": "2026-05-20T09:00:00Z",
--   "ends_at": "2026-05-20T09:30:00Z",
--   "timezone": "America/New_York",
--   "ical_uid": "abc123@google.com",
--   "rrule": "FREQ=WEEKLY;BYDAY=MO",
--   "location": "Room 4B",
--   "meeting_type": "recurring_series",
--   "visibility": "default"
-- }

-- MeetingUpdated
-- payload: { "changes": {"title": {"old": "Standup", "new": "Weekly Standup"}} }

-- MeetingCancelled
-- payload: { "reason": "Organizer unavailable", "notify_participants": true }

-- MeetingStarted
-- payload: { "actual_start": "2026-05-20T09:02:00Z" }

-- MeetingEnded
-- payload: { "actual_end": "2026-05-20T09:28:00Z", "duration_seconds": 1560 }

-- ============================================================
-- PARTICIPANT EVENTS
-- ============================================================

-- ParticipantInvited
-- payload: { "email": "jane@acme.com", "display_name": "Jane Doe", "role": "attendee" }

-- ParticipantRSVPChanged
-- payload: { "email": "jane@acme.com", "old_status": "pending", "new_status": "accepted" }

-- ParticipantJoinedLive
-- payload: { "email": "jane@acme.com", "joined_at": "2026-05-20T09:01:00Z" }

-- ParticipantLeftLive
-- payload: { "email": "jane@acme.com", "left_at": "2026-05-20T09:27:00Z" }

-- ============================================================
-- AGENDA EVENTS
-- ============================================================

-- AgendaCreated
-- payload: { "agenda_id": "uuid" }

-- AgendaItemAdded
-- payload: { "item_id": "uuid", "title": "Review sprint goals", "duration_minutes": 10, "sort_order": 1 }

-- AgendaItemStatusChanged
-- payload: { "item_id": "uuid", "old_status": "pending", "new_status": "discussed" }

-- ============================================================
-- RECORDING & CONSENT EVENTS
-- ============================================================

-- RecordingStarted
-- payload: { "recording_id": "uuid", "capture_method": "device_audio", "media_type": "audio" }

-- RecordingCompleted
-- payload: { "recording_id": "uuid", "duration_seconds": 1560, "file_size_bytes": 15000000, "storage_path": "s3://..." }

-- RecordingConsentGranted
-- payload: { "participant_email": "jane@acme.com", "consent_method": "in_meeting_prompt" }

-- RecordingConsentRevoked
-- payload: { "participant_email": "jane@acme.com", "reason": "participant_request" }

-- ============================================================
-- TRANSCRIPTION EVENTS
-- ============================================================

-- TranscriptionStarted
-- payload: { "transcript_id": "uuid", "engine": "whisper", "language": "en" }

-- TranscriptSegmentAdded
-- payload: { "segment_index": 42, "speaker_label": "Speaker 1", "start_time_ms": 120000, "end_time_ms": 125000, "text": "I think we should...", "confidence": 0.94 }

-- SpeakerIdentified
-- payload: { "speaker_label": "Speaker 1", "resolved_email": "jane@acme.com", "resolution_method": "voice_match" }

-- TranscriptionCompleted
-- payload: { "transcript_id": "uuid", "segment_count": 312, "accuracy_score": 0.93 }

-- ============================================================
-- ACTION ITEM & DECISION EVENTS
-- ============================================================

-- ActionItemCreated
-- payload: { "title": "Update API docs", "assigned_to_email": "bob@acme.com", "due_date": "2026-05-25", "source": "ai_extracted", "ai_confidence": 0.91, "source_segment_index": 42 }

-- ActionItemStatusChanged
-- payload: { "old_status": "open", "new_status": "completed", "completed_at": "2026-05-24T15:00:00Z" }

-- DecisionRecorded
-- payload: { "title": "Use PostgreSQL for data layer", "description": "Team agreed on PG over MySQL", "decided_by_email": "jane@acme.com", "source": "ai_extracted", "supersedes_decision_id": null }

-- ============================================================
-- SUMMARY & NOTE EVENTS
-- ============================================================

-- MeetingSummaryGenerated
-- payload: { "summary_type": "overview", "content": "The team discussed...", "ai_model": "claude-4" }

-- MeetingNoteAdded
-- payload: { "author_id": "uuid", "content": "Key takeaway: ...", "content_format": "markdown" }

-- ============================================================
-- ANALYTICS EVENTS
-- ============================================================

-- SpeakerAnalyticsComputed
-- payload: { "speaker_label": "Speaker 1", "talk_time_seconds": 420, "word_count": 850, "sentiment_score": 0.72 }

-- MeetingQualityScored
-- payload: { "participation_balance": 0.78, "decision_velocity": 12, "action_item_count": 5, "overall_score": 0.82 }

-- ============================================================
-- INTEGRATION SYNC EVENTS
-- ============================================================

-- CRMSyncCompleted
-- payload: { "crm_provider": "salesforce", "external_record_id": "006xxx", "sync_direction": "outbound" }

-- CRMSyncFailed
-- payload: { "crm_provider": "hubspot", "error": "Rate limit exceeded", "retry_after": 60 }
```

## Read Model Projections

Projections are materialised views rebuilt from the event stream. They serve all read queries.

```sql
-- ============================================================
-- PROJECTION: Current Meeting State
-- ============================================================
CREATE TABLE proj_meetings (
    id UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    location VARCHAR(500),
    meeting_type VARCHAR(30),
    series_id UUID,
    rrule TEXT,
    starts_at TIMESTAMPTZ NOT NULL,
    ends_at TIMESTAMPTZ NOT NULL,
    timezone VARCHAR(50),
    status VARCHAR(20) NOT NULL,
    visibility VARCHAR(20),
    created_by UUID NOT NULL,
    participant_count INT DEFAULT 0,
    action_item_count INT DEFAULT 0,
    decision_count INT DEFAULT 0,
    has_recording BOOLEAN DEFAULT false,
    has_transcript BOOLEAN DEFAULT false,
    quality_score DECIMAL(5,4),
    last_event_version INT NOT NULL,       -- for optimistic concurrency
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_meetings_org ON proj_meetings (organization_id, starts_at);
CREATE INDEX idx_proj_meetings_status ON proj_meetings (status, starts_at);

-- ============================================================
-- PROJECTION: Current Participants
-- ============================================================
CREATE TABLE proj_meeting_participants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES proj_meetings(id),
    user_id UUID,
    email VARCHAR(320) NOT NULL,
    display_name VARCHAR(255),
    role VARCHAR(20),
    rsvp_status VARCHAR(20),
    joined_live_at TIMESTAMPTZ,
    left_live_at TIMESTAMPTZ,
    talk_time_seconds INT,
    UNIQUE (meeting_id, email)
);
CREATE INDEX idx_proj_participants_meeting ON proj_meeting_participants (meeting_id);

-- ============================================================
-- PROJECTION: Action Items (cross-meeting view)
-- ============================================================
CREATE TABLE proj_action_items (
    id UUID PRIMARY KEY,
    meeting_id UUID NOT NULL,
    organization_id UUID NOT NULL,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    assigned_to_email VARCHAR(320),
    assigned_to_user_id UUID,
    due_date DATE,
    priority VARCHAR(10),
    status VARCHAR(20) NOT NULL,
    source VARCHAR(20),
    ai_confidence DECIMAL(5,4),
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_actions_assigned ON proj_action_items (assigned_to_user_id, status);
CREATE INDEX idx_proj_actions_org ON proj_action_items (organization_id, status, due_date);

-- ============================================================
-- PROJECTION: Decisions (longitudinal tracking)
-- ============================================================
CREATE TABLE proj_decisions (
    id UUID PRIMARY KEY,
    meeting_id UUID NOT NULL,
    organization_id UUID NOT NULL,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    decided_by_email VARCHAR(320),
    source VARCHAR(20),
    supersedes_decision_id UUID,
    is_current BOOLEAN DEFAULT true,     -- false when superseded
    created_at TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_decisions_org ON proj_decisions (organization_id, is_current);

-- ============================================================
-- PROJECTION: Transcript (assembled from segments)
-- ============================================================
CREATE TABLE proj_transcripts (
    id UUID PRIMARY KEY,
    meeting_id UUID NOT NULL,
    language VARCHAR(10),
    engine VARCHAR(50),
    segment_count INT DEFAULT 0,
    accuracy_score DECIMAL(5,4),
    status VARCHAR(20),
    full_text TEXT,
    created_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE proj_transcript_segments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transcript_id UUID NOT NULL REFERENCES proj_transcripts(id),
    segment_index INT NOT NULL,
    speaker_label VARCHAR(100),
    speaker_email VARCHAR(320),           -- resolved speaker
    start_time_ms BIGINT NOT NULL,
    end_time_ms BIGINT NOT NULL,
    text TEXT NOT NULL,
    confidence DECIMAL(5,4)
);
CREATE INDEX idx_proj_segments_transcript ON proj_transcript_segments (transcript_id, segment_index);

-- ============================================================
-- PROJECTION: Full-Text Search
-- ============================================================
CREATE TABLE proj_search (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    entity_type VARCHAR(30) NOT NULL,
    entity_id UUID NOT NULL,
    meeting_id UUID,
    searchable_text TEXT NOT NULL,
    search_vector TSVECTOR GENERATED ALWAYS AS (to_tsvector('english', searchable_text)) STORED,
    created_at TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_search_vector ON proj_search USING GIN (search_vector);
CREATE INDEX idx_proj_search_org ON proj_search (organization_id, entity_type);

-- ============================================================
-- PROJECTION: User's Meeting Timeline
-- ============================================================
CREATE TABLE proj_user_timeline (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    organization_id UUID NOT NULL,
    meeting_id UUID NOT NULL,
    meeting_title VARCHAR(500),
    starts_at TIMESTAMPTZ NOT NULL,
    role VARCHAR(20),
    action_items_assigned INT DEFAULT 0,
    action_items_completed INT DEFAULT 0
);
CREATE INDEX idx_proj_timeline_user ON proj_user_timeline (user_id, starts_at DESC);
```

## Projection Rebuilder (Pseudocode)

```sql
-- Example: rebuilding the proj_meetings projection from events
-- This runs as a background process, consuming events in order

-- Fetch events for a meeting aggregate:
SELECT event_type, payload, metadata, event_version, created_at
FROM events
WHERE aggregate_type = 'Meeting'
  AND aggregate_id = '550e8400-e29b-41d4-a716-446655440000'
ORDER BY event_version ASC;

-- The projection builder applies each event in sequence:
-- MeetingCreated    -> INSERT INTO proj_meetings
-- MeetingUpdated    -> UPDATE proj_meetings SET title = ...
-- ParticipantInvited -> UPDATE proj_meetings SET participant_count = participant_count + 1
-- RecordingCompleted -> UPDATE proj_meetings SET has_recording = true
-- etc.
```

## Temporal Query Examples

```sql
-- "What decisions were active on March 15, 2026?"
SELECT DISTINCT ON (d.aggregate_id) d.payload->>'title' as decision_title
FROM events d
WHERE d.event_type = 'DecisionRecorded'
  AND d.created_at <= '2026-03-15T23:59:59Z'
  AND d.aggregate_id NOT IN (
    SELECT payload->>'supersedes_decision_id'
    FROM events
    WHERE event_type = 'DecisionRecorded'
      AND created_at <= '2026-03-15T23:59:59Z'
      AND payload->>'supersedes_decision_id' IS NOT NULL
  )
ORDER BY d.aggregate_id, d.event_version DESC;

-- "What was the participant list for meeting X at 9:05 AM?"
SELECT payload->>'email' as participant_email, payload->>'role' as role
FROM events
WHERE aggregate_type = 'Meeting'
  AND aggregate_id = '550e8400-...'
  AND event_type = 'ParticipantInvited'
  AND created_at <= '2026-05-20T09:05:00Z';
```

## Crypto-Shredding for GDPR Right to Erasure

```sql
-- Each user's event payloads are encrypted with a per-user key
CREATE TABLE user_encryption_keys (
    user_id UUID PRIMARY KEY,
    encryption_key BYTEA NOT NULL,        -- AES-256 key, itself encrypted at rest
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- To "erase" a user: delete their encryption key
-- All their event payloads become unreadable, but the event structure remains
-- for aggregate integrity. Projections are rebuilt, omitting unreadable payloads.
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events, snapshots |
| Crypto-Shredding | 1 | user_encryption_keys |
| Projection: Meetings | 2 | proj_meetings, proj_meeting_participants |
| Projection: Action Items | 1 | proj_action_items |
| Projection: Decisions | 1 | proj_decisions |
| Projection: Transcripts | 2 | proj_transcripts, proj_transcript_segments |
| Projection: Search | 1 | proj_search |
| Projection: User Timeline | 1 | proj_user_timeline |
| **Total** | **11** | Plus the event store which replaces ~20 operational tables |

---

## Key Design Decisions

1. **Single `events` table instead of per-aggregate tables** -- simplifies the infrastructure and enables cross-aggregate queries; the `(aggregate_type, aggregate_id, event_version)` unique constraint ensures ordering within each aggregate stream.

2. **JSONB payloads over typed columns** -- each event type has a different structure; JSONB allows schema evolution without migrations. Event schemas are enforced at the application layer via validation.

3. **CloudEvents-inspired `metadata` envelope** -- every event carries actor, correlation, causation, and source metadata, enabling distributed tracing and causal reasoning across event chains.

4. **Projections are disposable and rebuildable** -- if a projection becomes corrupted or a new read model is needed, it can be rebuilt from scratch by replaying the event stream. This is a fundamental advantage over mutable-state systems.

5. **Crypto-shredding for GDPR** -- rather than deleting events (which would break aggregate integrity), user data is encrypted with per-user keys. Deleting the key makes the data unrecoverable while preserving the event structure.

6. **Snapshot strategy for performance** -- aggregates with long event streams (e.g., a recurring meeting series with 200+ instances) use snapshots to avoid replaying hundreds of events on every load.

7. **Temporal queries are first-class** -- the event store naturally supports "point-in-time" queries that are expensive or impossible in a mutable-state database.

8. **The event store IS the audit log** -- no separate audit table is needed; every mutation is already an immutable event with full context. This saves significant development effort for compliance features.

9. **Eventually consistent read models** -- the trade-off of CQRS; projections may lag the event store by milliseconds to seconds. The application must handle this (e.g., "your changes are being processed" UX).

10. **Event type catalogue as a domain language** -- the event types serve as a shared vocabulary between engineering, product, and compliance teams. "When does `RecordingConsentGranted` get emitted?" is a precise, answerable question.

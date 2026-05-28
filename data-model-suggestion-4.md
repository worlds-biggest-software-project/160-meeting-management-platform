# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Meeting Management Platform · Created: 2026-05-20

## Philosophy

This model combines a relational foundation for operational CRUD with a property graph layer for relationship-heavy queries. The core meeting management entities (meetings, users, recordings, transcripts) live in standard relational tables. A parallel graph structure -- implemented as `graph_nodes` and `graph_edges` tables in PostgreSQL -- captures the rich web of relationships between people, meetings, decisions, action items, and topics. This enables queries that are extremely expensive in pure relational models: "show me the chain of decisions that led to this one", "who has the most influence over this topic area?", "find all meetings connected to this project through shared participants, action items, or referenced decisions."

The graph-relational hybrid is used by LinkedIn (social graph + relational profile data), GitHub (contribution graph + repository CRUD), and knowledge management tools like Roam Research and Obsidian. For a meeting management platform, the graph layer powers the cross-meeting knowledge graph described in the README: understanding who committed to what, when; surfacing conflicts and gaps; and tracking the longitudinal chain of decisions across dozens of meetings over months.

The relational tables handle the "system of record" concerns (scheduling, CRUD, integrations), while the graph handles the "system of intelligence" concerns (knowledge discovery, relationship traversal, AI-powered insights). This separation keeps operational queries fast and predictable while enabling the graph-based analytics that differentiate the platform from transcription-only competitors.

**Best for:** Platforms where cross-meeting knowledge synthesis, decision chain tracking, participant influence analysis, and topic clustering are core differentiators -- not just nice-to-have features.

**Trade-offs:**
- (+) Enables relationship queries that are impractical in pure relational models
- (+) Graph traversal finds hidden connections across meetings, people, and decisions
- (+) Natural fit for AI features: knowledge graph as context for LLM queries
- (+) Graph layer can be added incrementally without changing relational tables
- (+) Recursive CTE queries on edge tables work in standard PostgreSQL (no graph DB needed)
- (-) Dual-write complexity: changes must update both relational and graph layers
- (-) Graph consistency must be maintained via application logic or triggers
- (-) Graph queries (multi-hop traversals) can be expensive without careful indexing
- (-) Team must understand both relational and graph query patterns
- (-) If the product never builds knowledge-graph features, the graph layer is dead weight

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RFC 5545 (iCalendar) | Core meeting properties in relational tables; RFC 5545 RELATED-TO property modelled as graph edges between meeting nodes |
| RFC 7986 (iCalendar Extensions) | Conference URIs stored on relational meeting table |
| RFC 6749 (OAuth 2.0) | OAuth connections in relational tables; integration relationships in graph |
| ISO 3166-1 | Jurisdiction nodes in graph enable geographic clustering of meetings and participants |
| ISO 639-1 | Language codes on transcript records; language nodes in graph for multilingual analytics |
| WebVTT (W3C) | Transcript segments in relational tables; topic and keyword nodes extracted into graph |
| GDPR/CCPA | Consent and data access tracked relationally; graph edges enable "who accessed whose data" traversal for privacy audits |

---

## Relational Layer: Operational Tables

```sql
-- ============================================================
-- IDENTITY & MULTI-TENANCY
-- ============================================================
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    country_code CHAR(2),
    timezone VARCHAR(50) DEFAULT 'UTC',
    plan_tier VARCHAR(20) DEFAULT 'free',
    settings JSONB DEFAULT '{}',
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
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(30) NOT NULL DEFAULT 'member',
    joined_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);
CREATE INDEX idx_org_members_org ON organization_members (organization_id);

-- ============================================================
-- MEETINGS (operational CRUD)
-- ============================================================
CREATE TABLE meetings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    location VARCHAR(500),
    meeting_type VARCHAR(30) DEFAULT 'one_off',
    series_id UUID REFERENCES meetings(id),
    rrule TEXT,
    starts_at TIMESTAMPTZ NOT NULL,
    ends_at TIMESTAMPTZ NOT NULL,
    timezone VARCHAR(50),
    status VARCHAR(20) DEFAULT 'scheduled',
    conference_data JSONB DEFAULT '{}',
    ical_uid VARCHAR(512),
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_meetings_org_starts ON meetings (organization_id, starts_at);
CREATE INDEX idx_meetings_series ON meetings (series_id) WHERE series_id IS NOT NULL;

CREATE TABLE meeting_participants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),
    email VARCHAR(320) NOT NULL,
    display_name VARCHAR(255),
    role VARCHAR(20) DEFAULT 'attendee',
    rsvp_status VARCHAR(20) DEFAULT 'pending',
    UNIQUE (meeting_id, email)
);
CREATE INDEX idx_participants_meeting ON meeting_participants (meeting_id);
CREATE INDEX idx_participants_user ON meeting_participants (user_id) WHERE user_id IS NOT NULL;

-- ============================================================
-- RECORDINGS & TRANSCRIPTS (operational)
-- ============================================================
CREATE TABLE recordings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    capture_method VARCHAR(20) NOT NULL,
    media_type VARCHAR(10) NOT NULL,
    status VARCHAR(20) DEFAULT 'processing',
    duration_seconds INT,
    file_size_bytes BIGINT,
    storage_path TEXT,
    storage_provider VARCHAR(30),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_recordings_meeting ON recordings (meeting_id);

CREATE TABLE transcripts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    recording_id UUID REFERENCES recordings(id),
    language VARCHAR(10) DEFAULT 'en',
    engine VARCHAR(50),
    status VARCHAR(20) DEFAULT 'processing',
    accuracy_score DECIMAL(5,4),
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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_segments_transcript ON transcript_segments (transcript_id, segment_index);

-- ============================================================
-- ACTION ITEMS & DECISIONS (operational)
-- ============================================================
CREATE TABLE action_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    assigned_to UUID REFERENCES users(id),
    due_date DATE,
    priority VARCHAR(10) DEFAULT 'medium',
    status VARCHAR(20) DEFAULT 'open',
    source VARCHAR(20) DEFAULT 'manual',
    ai_confidence DECIMAL(5,4),
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_actions_org_status ON action_items (organization_id, status);
CREATE INDEX idx_actions_assigned ON action_items (assigned_to, status) WHERE assigned_to IS NOT NULL;

CREATE TABLE decisions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    decided_by UUID REFERENCES users(id),
    source VARCHAR(20) DEFAULT 'manual',
    ai_confidence DECIMAL(5,4),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_decisions_org ON decisions (organization_id);

-- ============================================================
-- INTEGRATIONS & AUDIT
-- ============================================================
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    provider VARCHAR(50) NOT NULL,
    provider_type VARCHAR(20) NOT NULL,
    access_token_encrypted TEXT,
    refresh_token_encrypted TEXT,
    token_expires_at TIMESTAMPTZ,
    provider_config JSONB DEFAULT '{}',
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_integrations_user ON integrations (user_id, provider_type);

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    user_id UUID,
    action VARCHAR(50) NOT NULL,
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB,
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org_time ON audit_log (organization_id, created_at);
```

## Graph Layer: Knowledge Graph

```sql
-- ============================================================
-- GRAPH NODE TABLE
-- ============================================================
-- Every entity that participates in the knowledge graph gets a node.
-- The node references back to the relational table for full data.
CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    node_type VARCHAR(50) NOT NULL,
    -- Node types:
    -- 'person'       -> users table
    -- 'meeting'      -> meetings table
    -- 'action_item'  -> action_items table
    -- 'decision'     -> decisions table
    -- 'topic'        -> extracted topic (no relational backing)
    -- 'keyword'      -> extracted keyword
    -- 'project'      -> external project reference
    -- 'document'     -> linked document URL
    -- 'team'         -> teams
    entity_id UUID,                        -- FK to relational table (NULL for graph-only nodes)
    label VARCHAR(500) NOT NULL,           -- display label
    properties JSONB DEFAULT '{}',         -- node-specific properties
    -- properties example (topic node):
    -- {
    --   "canonical_name": "database migration",
    --   "aliases": ["db migration", "schema migration"],
    --   "first_mentioned": "2026-01-15T10:00:00Z",
    --   "mention_count": 47,
    --   "category": "engineering"
    -- }
    -- properties example (person node):
    -- {
    --   "email": "jane@acme.com",
    --   "department": "Engineering",
    --   "expertise_areas": ["databases", "api-design"],
    --   "meeting_count": 156
    -- }
    embedding VECTOR(1536),               -- for semantic similarity search (pgvector)
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_graph_nodes_org_type ON graph_nodes (organization_id, node_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes (entity_id) WHERE entity_id IS NOT NULL;
CREATE INDEX idx_graph_nodes_label ON graph_nodes USING GIN (to_tsvector('english', label));
-- Requires pgvector extension:
-- CREATE INDEX idx_graph_nodes_embedding ON graph_nodes USING ivfflat (embedding vector_cosine_ops);

-- ============================================================
-- GRAPH EDGE TABLE
-- ============================================================
-- Edges capture typed, weighted, temporal relationships between nodes.
CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    source_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type VARCHAR(50) NOT NULL,
    -- Edge types:
    -- 'attended'          -> person -> meeting
    -- 'organised'         -> person -> meeting
    -- 'assigned_to'       -> action_item -> person
    -- 'created_in'        -> action_item -> meeting
    -- 'decided_in'        -> decision -> meeting
    -- 'decided_by'        -> decision -> person
    -- 'supersedes'        -> decision -> decision
    -- 'discussed'         -> meeting -> topic
    -- 'mentioned'         -> person -> topic (in a meeting)
    -- 'follows_up'        -> meeting -> meeting (recurring series link)
    -- 'references'        -> decision -> document
    -- 'collaborates_with' -> person -> person (computed from co-attendance)
    -- 'expert_in'         -> person -> topic (computed from speaking patterns)
    -- 'blocks'            -> action_item -> action_item
    -- 'related_to'        -> decision -> action_item
    weight DECIMAL(5,4) DEFAULT 1.0,      -- relationship strength (0.0 to 1.0)
    properties JSONB DEFAULT '{}',
    -- properties example (attended edge):
    -- {
    --   "talk_time_seconds": 420,
    --   "sentiment_score": 0.72,
    --   "role": "presenter"
    -- }
    -- properties example (discussed edge):
    -- {
    --   "segment_indices": [12, 45, 67],
    --   "duration_seconds": 300,
    --   "depth": "deep"               -- surface, moderate, deep
    -- }
    -- properties example (collaborates_with edge):
    -- {
    --   "meeting_count": 42,
    --   "shared_action_items": 15,
    --   "last_interaction": "2026-05-20T10:00:00Z"
    -- }
    valid_from TIMESTAMPTZ DEFAULT now(), -- temporal: when this relationship started
    valid_to TIMESTAMPTZ,                 -- NULL means currently active
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_graph_edges_source ON graph_edges (source_node_id, edge_type);
CREATE INDEX idx_graph_edges_target ON graph_edges (target_node_id, edge_type);
CREATE INDEX idx_graph_edges_org_type ON graph_edges (organization_id, edge_type);
CREATE INDEX idx_graph_edges_temporal ON graph_edges (valid_from, valid_to)
    WHERE valid_to IS NULL;  -- index active edges
```

## Graph Query Examples

```sql
-- ============================================================
-- 1. Decision chain: what decisions led to this one?
-- ============================================================
WITH RECURSIVE decision_chain AS (
    -- Start from the current decision
    SELECT
        gn.id AS node_id,
        gn.label AS decision_title,
        gn.entity_id AS decision_id,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.entity_id = $1  -- starting decision UUID
      AND gn.node_type = 'decision'

    UNION ALL

    -- Follow 'supersedes' edges backward
    SELECT
        prev.id,
        prev.label,
        prev.entity_id,
        dc.depth + 1,
        dc.path || prev.id
    FROM decision_chain dc
    JOIN graph_edges ge ON ge.source_node_id = dc.node_id AND ge.edge_type = 'supersedes'
    JOIN graph_nodes prev ON prev.id = ge.target_node_id
    WHERE prev.id != ALL(dc.path)  -- prevent cycles
      AND dc.depth < 10
)
SELECT depth, decision_title, decision_id
FROM decision_chain
ORDER BY depth ASC;

-- ============================================================
-- 2. Who are the top experts on a topic?
-- ============================================================
SELECT
    person_node.label AS person_name,
    person_node.properties->>'email' AS email,
    e.weight AS expertise_score,
    e.properties->>'meeting_count' AS meetings_on_topic
FROM graph_nodes topic_node
JOIN graph_edges e ON e.target_node_id = topic_node.id AND e.edge_type = 'expert_in'
JOIN graph_nodes person_node ON person_node.id = e.source_node_id
WHERE topic_node.node_type = 'topic'
  AND topic_node.label ILIKE '%database migration%'
  AND topic_node.organization_id = $1
ORDER BY e.weight DESC
LIMIT 10;

-- ============================================================
-- 3. Find meetings connected to a project through any path
--    (participants, action items, decisions, or topics)
-- ============================================================
WITH RECURSIVE connected AS (
    SELECT
        ge.target_node_id AS node_id,
        ge.edge_type,
        1 AS hops,
        ARRAY[ge.source_node_id, ge.target_node_id] AS path
    FROM graph_nodes project_node
    JOIN graph_edges ge ON ge.source_node_id = project_node.id
    WHERE project_node.entity_id = $1  -- project UUID
      AND project_node.node_type = 'project'

    UNION ALL

    SELECT
        ge2.target_node_id,
        ge2.edge_type,
        c.hops + 1,
        c.path || ge2.target_node_id
    FROM connected c
    JOIN graph_edges ge2 ON ge2.source_node_id = c.node_id
    WHERE ge2.target_node_id != ALL(c.path)
      AND c.hops < 3  -- max 3 hops
)
SELECT DISTINCT gn.entity_id AS meeting_id, gn.label AS meeting_title
FROM connected c
JOIN graph_nodes gn ON gn.id = c.node_id AND gn.node_type = 'meeting';

-- ============================================================
-- 4. Collaboration strength between two people
-- ============================================================
SELECT
    e.weight AS collaboration_score,
    e.properties->>'meeting_count' AS shared_meetings,
    e.properties->>'shared_action_items' AS shared_actions,
    e.properties->>'last_interaction' AS last_interaction
FROM graph_nodes p1
JOIN graph_edges e ON e.source_node_id = p1.id AND e.edge_type = 'collaborates_with'
JOIN graph_nodes p2 ON p2.id = e.target_node_id
WHERE p1.properties->>'email' = 'jane@acme.com'
  AND p2.properties->>'email' = 'bob@acme.com'
  AND p1.organization_id = $1;

-- ============================================================
-- 5. Semantic similarity: find meetings related to a query
--    (using pgvector cosine similarity)
-- ============================================================
-- SELECT gn.entity_id AS meeting_id, gn.label, 1 - (gn.embedding <=> $query_embedding) AS similarity
-- FROM graph_nodes gn
-- WHERE gn.organization_id = $1
--   AND gn.node_type = 'meeting'
-- ORDER BY gn.embedding <=> $query_embedding
-- LIMIT 10;
```

## Graph Maintenance: Sync Triggers

```sql
-- Trigger to create/update graph nodes when relational data changes
-- (Shown as pseudocode; implement as PostgreSQL trigger functions)

-- After INSERT on meetings:
--   INSERT INTO graph_nodes (organization_id, node_type, entity_id, label)
--   VALUES (NEW.organization_id, 'meeting', NEW.id, NEW.title);

-- After INSERT on meeting_participants:
--   -- Ensure person node exists
--   INSERT INTO graph_nodes (...) ON CONFLICT (entity_id) DO NOTHING;
--   -- Create 'attended' edge
--   INSERT INTO graph_edges (organization_id, source_node_id, target_node_id, edge_type)
--   SELECT mp.user_id, person_node.id, meeting_node.id, 'attended'
--   FROM ...;

-- After INSERT on action_items:
--   -- Create action_item node
--   -- Create 'created_in' edge to meeting
--   -- Create 'assigned_to' edge to person

-- After INSERT on decisions:
--   -- Create decision node
--   -- Create 'decided_in' edge to meeting
--   -- Create 'decided_by' edge to person

-- BATCH JOB: After transcript processing
--   -- AI extracts topics from transcript
--   -- Create/update topic nodes
--   -- Create 'discussed' edges from meeting to topics
--   -- Compute 'expert_in' edges from person speaking time on topics
--   -- Compute 'collaborates_with' edges from co-attendance patterns
--   -- Generate embeddings for meeting and topic nodes
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organizations, users, organization_members |
| Meetings (relational) | 2 | meetings, meeting_participants |
| Recording & Transcription | 3 | recordings, transcripts, transcript_segments |
| Action Items & Decisions | 2 | action_items, decisions |
| Integrations | 1 | integrations |
| Audit | 1 | audit_log |
| **Graph Layer** | **2** | **graph_nodes, graph_edges** |
| **Total** | **14** | Relational for CRUD, graph for intelligence |

---

## Key Design Decisions

1. **Graph in PostgreSQL, not a separate graph database** -- using `graph_nodes` and `graph_edges` tables in the same PostgreSQL instance avoids operational complexity of running Neo4j or Amazon Neptune alongside the primary database. Recursive CTEs handle multi-hop traversals adequately for organisations with up to millions of edges.

2. **`entity_id` on graph nodes links back to relational tables** -- the graph layer is not a replacement for relational data; it augments it. Full entity data lives in the relational table; the graph node carries only the label, type, and relationship-relevant properties.

3. **Temporal edges with `valid_from` / `valid_to`** -- relationships change over time. A person may leave a team, a decision may be superseded. Temporal edges preserve the historical graph for point-in-time analysis without deleting edges.

4. **Weight on edges enables ranking** -- not all relationships are equally strong. A person who attended 50 meetings on a topic is a stronger expert than someone who attended 2. The `weight` field enables PageRank-style influence scoring.

5. **`embedding` column on graph nodes (pgvector)** -- enables semantic similarity search. Meeting nodes and topic nodes get embeddings from their content, allowing "find meetings similar to this one" queries using cosine similarity.

6. **Graph-only nodes (topics, keywords, projects)** -- some graph nodes have no relational backing table. Topics are extracted by AI from transcripts and exist only in the graph. This allows the knowledge graph to grow organically as AI processing discovers new concepts.

7. **Computed edges via batch jobs** -- edges like `collaborates_with` and `expert_in` are not created by user actions but computed periodically from attendance patterns, speaking time, and action item co-ownership. These are the edges that power the "cross-meeting knowledge graph" feature.

8. **Lean relational layer** -- the relational tables in this model are deliberately simpler than in Suggestion 1. Agenda items, meeting notes, analytics, and search are not separate relational tables; they exist as graph nodes/edges or are deferred. The relational layer focuses on CRUD operations; the graph layer handles analytics and knowledge discovery.

9. **Graph as AI context provider** -- when the AI agenda builder or conversational AI needs to understand context ("what happened in previous meetings about topic X?"), it queries the graph for connected nodes rather than doing expensive full-text searches across all transcript tables.

10. **Incremental graph enrichment** -- the graph can start empty and be populated incrementally as meetings are transcribed and AI processing extracts topics, keywords, and relationships. The relational layer works independently from day one; the graph layer becomes more valuable over time.

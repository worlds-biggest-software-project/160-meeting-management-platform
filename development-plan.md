# Meeting Management Platform — Phased Development Plan

> Project: 160-meeting-management-platform · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js) | Full-stack unification: same language for API server, real-time WebSocket handling, and React frontend; strong async I/O for concurrent calendar sync, transcription callbacks, and CRM webhooks; mature ecosystem for every integration target (Google, Microsoft, Salesforce, HubSpot) |
| API framework | Fastify with @fastify/swagger | Highest-performance Node.js framework; native OpenAPI 3.1 schema generation via @fastify/swagger; JSON Schema validation built-in; plugin architecture matches the phased build approach |
| Database | PostgreSQL 16 with pgvector | Hybrid relational + JSONB model (Data Model Suggestion 3) fits the multi-provider integration surface; TSVECTOR for full-text search; pgvector for semantic similarity on meeting embeddings; Row-Level Security for multi-tenant isolation |
| ORM / query builder | Drizzle ORM | Type-safe SQL with zero runtime overhead; generates migration files; supports JSONB columns and PostgreSQL-specific features; lighter than Prisma, more type-safe than Knex |
| Task queue | BullMQ (Redis-backed) | Handles async workloads: transcription jobs, AI summarisation, CRM sync, calendar sync; supports delayed jobs, retries, rate limiting, and priority queues; Redis also serves as cache and pub/sub layer |
| Cache / pub-sub | Redis 7 | Session store, API response cache, real-time event broadcast (pub/sub for WebSocket), and BullMQ backing store — one dependency for three concerns |
| Real-time | Socket.IO on Fastify | Live transcript streaming, collaborative agenda editing, meeting status updates; Socket.IO handles reconnection and room management; namespace isolation per organisation |
| Frontend | Next.js 15 (App Router) | Server components for dashboard rendering; client components for real-time meeting UI; built-in API route proxying; Vercel deployment path for SaaS; self-host via Docker |
| UI component library | shadcn/ui + Tailwind CSS 4 | Unstyled, composable components; copy-paste ownership (no dependency lock-in); Radix primitives for accessibility; Tailwind for utility-first styling |
| Transcription engine | OpenAI Whisper (self-hosted) + Deepgram (cloud fallback) | Whisper (MIT license) for self-hosted/offline deployments; Deepgram for cloud tier with real-time streaming; abstracted behind a provider interface |
| AI / LLM provider | Anthropic Claude API (primary), OpenAI (fallback) | Claude for summarisation, action item extraction, agenda suggestion, conversational AI; provider interface allows swapping; prompt caching for cost efficiency |
| Object storage | S3-compatible (AWS S3, MinIO for self-hosted) | Recordings and transcript exports stored in S3; MinIO provides self-hosted S3 API compatibility; presigned URLs for secure browser access |
| Authentication | NextAuth.js v5 with OIDC/SAML | Supports Google, Microsoft, and custom OIDC providers; SAML for enterprise SSO; session management with JWT + database sessions |
| Containerisation | Docker + Docker Compose | Multi-container deployment: API, worker, frontend, PostgreSQL, Redis, MinIO; single `docker compose up` for self-hosted; individual images for cloud deployment |
| Testing framework | Vitest + Playwright | Vitest for unit and integration tests (Jest-compatible, faster); Playwright for E2E browser tests; testcontainers for PostgreSQL/Redis integration tests |
| Code quality | ESLint 9 (flat config) + Prettier + TypeScript strict mode | Flat ESLint config; Prettier for formatting; `strict: true` in tsconfig; lint-staged + Husky for pre-commit hooks |
| Package manager | pnpm | Efficient disk usage via content-addressable storage; workspace support for monorepo (API, frontend, shared types); lockfile determinism |
| Monorepo structure | pnpm workspaces + Turborepo | Shared types package between API and frontend; Turborepo for parallel builds and caching; single repo, multiple deployable packages |

### Data Model Selection

This plan adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB)** as the primary schema, with selected elements from **Suggestion 4 (Graph-Relational)** added in Phase 10 for cross-meeting knowledge synthesis. Rationale:

- 16 tables vs. 31 (Suggestion 1) reduces initial complexity
- JSONB columns absorb provider-specific variation (conference data, CRM field mappings, analytics metrics) without migration churn
- Core queryable fields remain typed columns with proper indexes
- Graph layer (graph_nodes, graph_edges) added later for the knowledge graph differentiator
- Event sourcing (Suggestion 2) deferred — the audit_log table provides compliance tracking without CQRS complexity at MVP stage

### Project Structure

```
meeting-management-platform/
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── .env.example
├── packages/
│   └── shared/                          # Shared types, constants, validation
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
│           ├── types/
│           │   ├── meeting.ts
│           │   ├── user.ts
│           │   ├── action-item.ts
│           │   ├── decision.ts
│           │   ├── transcript.ts
│           │   ├── integration.ts
│           │   └── api.ts               # Request/response schemas
│           ├── constants/
│           │   ├── meeting-status.ts
│           │   ├── providers.ts
│           │   └── roles.ts
│           └── validation/
│               ├── meeting.ts
│               └── user.ts
├── apps/
│   ├── api/                             # Fastify API server
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── server.ts                # Fastify setup, plugin registration
│   │       ├── config.ts                # Environment config with defaults
│   │       ├── db/
│   │       │   ├── schema.ts            # Drizzle schema (all tables)
│   │       │   ├── migrations/          # Generated SQL migrations
│   │       │   └── seed.ts
│   │       ├── plugins/
│   │       │   ├── auth.ts              # Authentication plugin
│   │       │   ├── rbac.ts              # Role-based access control
│   │       │   └── tenant.ts            # Multi-tenant context
│   │       ├── routes/
│   │       │   ├── meetings/
│   │       │   ├── agendas/
│   │       │   ├── recordings/
│   │       │   ├── transcripts/
│   │       │   ├── action-items/
│   │       │   ├── decisions/
│   │       │   ├── integrations/
│   │       │   ├── search/
│   │       │   ├── analytics/
│   │       │   └── admin/
│   │       ├── services/
│   │       │   ├── meeting.service.ts
│   │       │   ├── transcript.service.ts
│   │       │   ├── ai.service.ts
│   │       │   ├── calendar-sync.service.ts
│   │       │   ├── crm-sync.service.ts
│   │       │   ├── recording.service.ts
│   │       │   └── search.service.ts
│   │       ├── providers/
│   │       │   ├── transcription/
│   │       │   │   ├── interface.ts
│   │       │   │   ├── whisper.ts
│   │       │   │   └── deepgram.ts
│   │       │   ├── ai/
│   │       │   │   ├── interface.ts
│   │       │   │   ├── claude.ts
│   │       │   │   └── openai.ts
│   │       │   ├── calendar/
│   │       │   │   ├── interface.ts
│   │       │   │   ├── google.ts
│   │       │   │   └── microsoft.ts
│   │       │   ├── crm/
│   │       │   │   ├── interface.ts
│   │       │   │   ├── salesforce.ts
│   │       │   │   └── hubspot.ts
│   │       │   └── storage/
│   │       │       ├── interface.ts
│   │       │       ├── s3.ts
│   │       │       └── local.ts
│   │       └── workers/
│   │           ├── transcription.worker.ts
│   │           ├── summarisation.worker.ts
│   │           ├── action-extraction.worker.ts
│   │           ├── calendar-sync.worker.ts
│   │           ├── crm-sync.worker.ts
│   │           └── analytics.worker.ts
│   ├── web/                             # Next.js frontend
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── next.config.ts
│   │   └── src/
│   │       ├── app/
│   │       │   ├── layout.tsx
│   │       │   ├── (auth)/
│   │       │   │   ├── login/
│   │       │   │   └── signup/
│   │       │   ├── (dashboard)/
│   │       │   │   ├── meetings/
│   │       │   │   ├── action-items/
│   │       │   │   ├── decisions/
│   │       │   │   ├── search/
│   │       │   │   ├── analytics/
│   │       │   │   └── settings/
│   │       │   └── (meeting)/
│   │       │       └── [meetingId]/
│   │       │           ├── agenda/
│   │       │           ├── notes/
│   │       │           ├── transcript/
│   │       │           └── recording/
│   │       ├── components/
│   │       │   ├── ui/                  # shadcn/ui components
│   │       │   ├── meeting/
│   │       │   ├── agenda/
│   │       │   ├── transcript/
│   │       │   ├── action-item/
│   │       │   └── analytics/
│   │       ├── hooks/
│   │       ├── lib/
│   │       │   ├── api-client.ts
│   │       │   └── socket.ts
│   │       └── stores/
│   └── worker/                          # BullMQ worker process
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
│           ├── index.ts
│           └── processors/
└── tests/
    ├── fixtures/
    │   ├── ical/                        # .ics test files
    │   ├── transcripts/                 # Sample transcript segments
    │   ├── audio/                       # Short audio samples for transcription tests
    │   └── webhooks/                    # Sample webhook payloads
    ├── integration/
    └── e2e/
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the monorepo structure, database schema, authentication system, and development tooling. After this phase, a developer can run the full stack locally with `docker compose up`, authenticate via OAuth, and hit health-check API endpoints. Every subsequent phase builds on this foundation without restructuring.

### Tasks

#### 1.1 — Monorepo Scaffolding & Tooling

**What**: Initialise the pnpm workspace with three packages (api, web, shared), configure Turborepo, ESLint, Prettier, TypeScript, Husky, and Docker Compose.

**Design**:

```typescript
// pnpm-workspace.yaml
packages:
  - 'packages/*'
  - 'apps/*'

// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": [".env"],
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "lint": {},
    "test": { "dependsOn": ["^build"] },
    "test:integration": { "dependsOn": ["^build"] },
    "db:migrate": { "cache": false }
  }
}
```

```typescript
// apps/api/src/config.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.coerce.number().default(3001),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url().default('redis://localhost:6379'),
  JWT_SECRET: z.string().min(32),
  CORS_ORIGINS: z.string().default('http://localhost:3000'),
  S3_ENDPOINT: z.string().url().optional(),
  S3_BUCKET: z.string().default('meeting-recordings'),
  S3_ACCESS_KEY: z.string().optional(),
  S3_SECRET_KEY: z.string().optional(),
  ANTHROPIC_API_KEY: z.string().optional(),
  OPENAI_API_KEY: z.string().optional(),
  WHISPER_ENDPOINT: z.string().url().optional(),
  DEEPGRAM_API_KEY: z.string().optional(),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

export type EnvConfig = z.infer<typeof envSchema>;
export const config = envSchema.parse(process.env);
```

```yaml
# docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: meeting_platform
      POSTGRES_USER: meeting_user
      POSTGRES_PASSWORD: meeting_pass
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]
    volumes: ["miniodata:/data"]
  api:
    build: { context: ., dockerfile: Dockerfile.api }
    ports: ["3001:3001"]
    depends_on: [postgres, redis, minio]
    env_file: .env
  web:
    build: { context: ., dockerfile: Dockerfile.web }
    ports: ["3000:3000"]
    depends_on: [api]
    env_file: .env
volumes:
  pgdata:
  miniodata:
```

**Testing**:
- `Unit: envSchema.parse with valid env → config object with correct types`
- `Unit: envSchema.parse with missing DATABASE_URL → ZodError listing required field`
- `Unit: envSchema.parse with defaults → PORT=3001, LOG_LEVEL=info, NODE_ENV=development`
- `Integration: docker compose up → all services healthy within 30 seconds`
- `Integration: API /health endpoint → { status: 'ok', version: '0.1.0' }`

---

#### 1.2 — Database Schema & Migrations

**What**: Implement the Hybrid Relational + JSONB schema (Data Model Suggestion 3) using Drizzle ORM with migration files.

**Design**:

```typescript
// apps/api/src/db/schema.ts
import { pgTable, uuid, varchar, text, timestamp, boolean, integer, bigint, decimal, jsonb, uniqueIndex, index } from 'drizzle-orm/pg-core';

export const organizations = pgTable('organizations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).unique().notNull(),
  countryCode: varchar('country_code', { length: 2 }),
  timezone: varchar('timezone', { length: 50 }).default('UTC'),
  planTier: varchar('plan_tier', { length: 20 }).default('free'),
  settings: jsonb('settings').default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
});

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 320 }).unique().notNull(),
  displayName: varchar('display_name', { length: 255 }).notNull(),
  avatarUrl: text('avatar_url'),
  locale: varchar('locale', { length: 10 }).default('en'),
  timezone: varchar('timezone', { length: 50 }).default('UTC'),
  preferences: jsonb('preferences').default({}),
  isActive: boolean('is_active').default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
});

export const organizationMembers = pgTable('organization_members', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  role: varchar('role', { length: 30 }).notNull().default('member'),
  permissions: jsonb('permissions').default({}),
  joinedAt: timestamp('joined_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  uniqueMember: uniqueIndex('idx_org_member_unique').on(table.organizationId, table.userId),
}));

export const integrations = pgTable('integrations', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  provider: varchar('provider', { length: 50 }).notNull(),
  providerType: varchar('provider_type', { length: 20 }).notNull(),
  providerAccountId: varchar('provider_account_id', { length: 255 }),
  accessTokenEncrypted: text('access_token_encrypted'),
  refreshTokenEncrypted: text('refresh_token_encrypted'),
  tokenExpiresAt: timestamp('token_expires_at', { withTimezone: true }),
  scopes: text('scopes').array(),
  isActive: boolean('is_active').default(true),
  providerConfig: jsonb('provider_config').default({}),
  lastSyncedAt: timestamp('last_synced_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
});

export const meetings = pgTable('meetings', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  title: varchar('title', { length: 500 }).notNull(),
  startsAt: timestamp('starts_at', { withTimezone: true }).notNull(),
  endsAt: timestamp('ends_at', { withTimezone: true }).notNull(),
  timezone: varchar('timezone', { length: 50 }),
  status: varchar('status', { length: 20 }).default('scheduled'),
  meetingType: varchar('meeting_type', { length: 30 }).default('one_off'),
  seriesId: uuid('series_id'),
  rrule: text('rrule'),
  createdBy: uuid('created_by').notNull().references(() => users.id),
  icalUid: varchar('ical_uid', { length: 512 }),
  externalCalendarEventId: varchar('external_calendar_event_id', { length: 255 }),
  sourceIntegrationId: uuid('source_integration_id').references(() => integrations.id),
  conferenceData: jsonb('conference_data').default({}),
  icalProperties: jsonb('ical_properties').default({}),
  metadata: jsonb('metadata').default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  orgStartsIdx: index('idx_meetings_org_starts').on(table.organizationId, table.startsAt),
  statusIdx: index('idx_meetings_status').on(table.status, table.startsAt),
}));

export const meetingParticipants = pgTable('meeting_participants', {
  id: uuid('id').primaryKey().defaultRandom(),
  meetingId: uuid('meeting_id').notNull().references(() => meetings.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').references(() => users.id),
  email: varchar('email', { length: 320 }).notNull(),
  displayName: varchar('display_name', { length: 255 }),
  role: varchar('role', { length: 20 }).default('attendee'),
  rsvpStatus: varchar('rsvp_status', { length: 20 }).default('pending'),
  isExternal: boolean('is_external').default(false),
  participationData: jsonb('participation_data').default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  meetingEmailUnique: uniqueIndex('idx_participants_meeting_email').on(table.meetingId, table.email),
}));

// Remaining tables: agenda_items, recordings, transcripts, transcript_segments,
// meeting_notes, action_items, decisions, search_index, meeting_analytics, audit_log
// follow the same pattern from Data Model Suggestion 3
```

Row-Level Security policies for multi-tenant isolation:

```sql
-- Applied via migration after table creation
ALTER TABLE meetings ENABLE ROW LEVEL SECURITY;

CREATE POLICY meetings_tenant_isolation ON meetings
  USING (organization_id = current_setting('app.current_org_id')::uuid);

-- Repeat for all tenant-scoped tables
```

**Testing**:
- `Unit: Drizzle schema compiles without TypeScript errors`
- `Integration: drizzle-kit generate → migration SQL files created`
- `Integration: drizzle-kit push to test database → all 16 tables created`
- `Integration: INSERT into organizations + users + organization_members → referential integrity holds`
- `Integration: INSERT meeting with invalid organization_id → foreign key violation`
- `Integration: RLS policy → query with wrong org context returns empty result set`
- `Fixture: seed.ts creates sample org, users, and meetings → SELECT counts match expected`

---

#### 1.3 — Authentication & Authorization

**What**: Implement OAuth 2.0 login (Google, Microsoft), session management, and RBAC middleware.

**Design**:

```typescript
// packages/shared/src/types/user.ts
export interface AuthUser {
  id: string;
  email: string;
  displayName: string;
  avatarUrl: string | null;
  organizationId: string;
  role: OrgRole;
}

export type OrgRole = 'owner' | 'admin' | 'member' | 'viewer';

export const ROLE_HIERARCHY: Record<OrgRole, number> = {
  owner: 40,
  admin: 30,
  member: 20,
  viewer: 10,
};

export interface Permission {
  resource: 'meeting' | 'recording' | 'transcript' | 'action_item' | 'decision' | 'integration' | 'member' | 'settings';
  action: 'create' | 'read' | 'update' | 'delete' | 'export';
}
```

```typescript
// apps/api/src/plugins/auth.ts
import fp from 'fastify-plugin';

declare module 'fastify' {
  interface FastifyRequest {
    user: AuthUser;
  }
}

export const authPlugin = fp(async (fastify) => {
  fastify.decorateRequest('user', null);

  fastify.addHook('preHandler', async (request, reply) => {
    // Skip auth for public routes
    if (request.routeOptions.config?.public) return;

    const token = request.headers.authorization?.replace('Bearer ', '');
    if (!token) return reply.code(401).send({ error: 'Unauthorized' });

    const session = await verifyToken(token);
    if (!session) return reply.code(401).send({ error: 'Invalid token' });

    request.user = session.user;

    // Set RLS context for PostgreSQL
    await fastify.db.execute(
      sql`SELECT set_config('app.current_org_id', ${session.user.organizationId}, true)`
    );
  });
});
```

```typescript
// apps/api/src/plugins/rbac.ts
export function requireRole(minRole: OrgRole) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    if (ROLE_HIERARCHY[request.user.role] < ROLE_HIERARCHY[minRole]) {
      return reply.code(403).send({ error: 'Insufficient permissions' });
    }
  };
}

export function requirePermission(permission: Permission) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const hasPermission = await checkPermission(request.user, permission);
    if (!hasPermission) {
      return reply.code(403).send({ error: 'Permission denied', required: permission });
    }
  };
}
```

OAuth callback flow:
1. Frontend redirects to `/api/auth/google` (or `/api/auth/microsoft`)
2. API redirects to provider OAuth consent screen
3. Provider redirects back to `/api/auth/callback/google` with auth code
4. API exchanges code for tokens, creates/updates user, creates session
5. API redirects to frontend with session cookie

**Testing**:
- `Unit: verifyToken with valid JWT → decoded AuthUser`
- `Unit: verifyToken with expired JWT → null`
- `Unit: requireRole('admin') with member user → 403`
- `Unit: requireRole('member') with admin user → passes (admin >= member)`
- `Integration (mocked OAuth): Google OAuth callback with valid code → user created, session returned`
- `Integration (mocked OAuth): Microsoft OAuth callback with existing user → user updated, session returned`
- `Integration: request without Authorization header → 401`
- `Integration: request with valid token → request.user populated, RLS context set`
- `E2E: login flow → redirect to Google → callback → dashboard loaded with user name`

---

#### 1.4 — Multi-Tenant Organization Management

**What**: CRUD endpoints for organizations and members; organization switching; invitation flow.

**Design**:

```typescript
// API Routes
// POST   /api/organizations                    → create org
// GET    /api/organizations/:orgId             → get org details
// PATCH  /api/organizations/:orgId             → update org settings
// GET    /api/organizations/:orgId/members     → list members
// POST   /api/organizations/:orgId/members     → invite member
// PATCH  /api/organizations/:orgId/members/:id → update member role
// DELETE /api/organizations/:orgId/members/:id → remove member

interface CreateOrganizationRequest {
  name: string;
  slug: string;
  countryCode?: string;
  timezone?: string;
}

interface CreateOrganizationResponse {
  id: string;
  name: string;
  slug: string;
  role: 'owner'; // creator is always owner
}

interface InviteMemberRequest {
  email: string;
  role: OrgRole;
}
```

**Testing**:
- `Unit: createOrganization with valid input → org created, creator is owner`
- `Unit: createOrganization with duplicate slug → ConflictError`
- `Unit: inviteMember as admin → invitation created`
- `Unit: inviteMember as viewer → 403`
- `Unit: removeMember (self) as owner when sole owner → 400 (cannot remove sole owner)`
- `Integration: create org → invite member → member accepts → member listed`
- `Integration: member with viewer role → cannot access admin endpoints`

---

## Phase 2: Meeting CRUD & Calendar Integration

### Purpose
Implement core meeting lifecycle (create, read, update, delete, list) and two-way calendar synchronization with Google Calendar and Microsoft Graph. After this phase, users can manage meetings natively and see them synced to/from their connected calendars. This is the operational backbone that all subsequent features build on.

### Tasks

#### 2.1 — Meeting CRUD API

**What**: Full REST API for meeting management with filtering, pagination, and recurring series support.

**Design**:

```typescript
// API Routes
// POST   /api/meetings                        → create meeting
// GET    /api/meetings                        → list meetings (with filters)
// GET    /api/meetings/:id                    → get meeting details
// PATCH  /api/meetings/:id                    → update meeting
// DELETE /api/meetings/:id                    → cancel meeting
// POST   /api/meetings/:id/participants       → add participant
// DELETE /api/meetings/:id/participants/:pid  → remove participant

interface CreateMeetingRequest {
  title: string;
  description?: string;
  startsAt: string;          // ISO 8601
  endsAt: string;            // ISO 8601
  timezone?: string;         // IANA timezone
  meetingType?: 'one_off' | 'recurring_series';
  rrule?: string;            // RFC 5545 RRULE (e.g., 'FREQ=WEEKLY;BYDAY=MO,WE,FR')
  conferenceData?: {
    provider: 'zoom' | 'google_meet' | 'teams' | 'webex' | 'custom';
    joinUrl: string;
    meetingId?: string;
    passcode?: string;
    dialIn?: string[];
  };
  participants?: Array<{
    email: string;
    displayName?: string;
    role?: 'organizer' | 'attendee' | 'optional';
  }>;
}

interface MeetingResponse {
  id: string;
  title: string;
  description: string | null;
  startsAt: string;
  endsAt: string;
  timezone: string;
  status: MeetingStatus;
  meetingType: 'one_off' | 'recurring_series' | 'recurring_instance';
  seriesId: string | null;
  rrule: string | null;
  conferenceData: ConferenceData;
  participants: ParticipantResponse[];
  actionItemCount: number;
  decisionCount: number;
  hasRecording: boolean;
  hasTranscript: boolean;
  createdBy: { id: string; displayName: string };
  createdAt: string;
  updatedAt: string;
}

type MeetingStatus = 'scheduled' | 'in_progress' | 'completed' | 'cancelled';

interface ListMeetingsQuery {
  startDate?: string;        // ISO 8601 date
  endDate?: string;          // ISO 8601 date
  status?: MeetingStatus;
  participantEmail?: string;
  search?: string;           // title search
  seriesId?: string;
  page?: number;
  limit?: number;            // default 20, max 100
  sort?: 'starts_at' | '-starts_at' | 'created_at' | '-created_at';
}
```

Recurring meeting expansion: when `meetingType` is `recurring_series`, the API stores the parent series row. A background job expands RRULE into individual `recurring_instance` rows up to 90 days ahead. Each instance references the parent via `seriesId`.

**Testing**:
- `Unit: createMeeting with valid input → meeting row inserted, 201 returned`
- `Unit: createMeeting with endTime before startTime → 400 validation error`
- `Unit: createMeeting with RRULE → series parent created, expansion job enqueued`
- `Unit: listMeetings with date range → only meetings in range returned`
- `Unit: listMeetings with participantEmail filter → only meetings with that participant`
- `Unit: getMeeting with participant counts → actionItemCount and decisionCount populated`
- `Unit: deleteMeeting → status set to 'cancelled', not physically deleted`
- `Unit: updateMeeting on recurring_instance → only that instance modified (not series)`
- `Integration: create meeting with 3 participants → all participants queryable`
- `Fixture: RRULE 'FREQ=WEEKLY;BYDAY=MO;COUNT=4' → 4 instances expanded`

---

#### 2.2 — Google Calendar Sync

**What**: Two-way synchronization with Google Calendar via the Calendar API v3, using incremental sync tokens and webhook push notifications.

**Design**:

```typescript
// apps/api/src/providers/calendar/interface.ts
export interface CalendarProvider {
  readonly providerName: string;

  getCalendars(integration: Integration): Promise<ExternalCalendar[]>;
  syncEvents(integration: Integration, syncToken?: string): Promise<CalendarSyncResult>;
  createEvent(integration: Integration, meeting: Meeting): Promise<ExternalEventRef>;
  updateEvent(integration: Integration, meeting: Meeting, externalId: string): Promise<void>;
  deleteEvent(integration: Integration, externalId: string): Promise<void>;
  setupWebhook(integration: Integration, callbackUrl: string): Promise<WebhookRegistration>;
}

export interface CalendarSyncResult {
  created: ExternalEvent[];
  updated: ExternalEvent[];
  deleted: string[];          // external event IDs
  nextSyncToken: string;
}

export interface ExternalEvent {
  externalId: string;
  icalUid: string;
  title: string;
  description?: string;
  startsAt: string;
  endsAt: string;
  timezone: string;
  rrule?: string;
  conferenceData?: ConferenceData;
  participants: Array<{ email: string; displayName?: string; rsvpStatus: string }>;
  providerMetadata: Record<string, unknown>;  // stored in meetings.metadata
}
```

```typescript
// apps/api/src/providers/calendar/google.ts
export class GoogleCalendarProvider implements CalendarProvider {
  readonly providerName = 'google_calendar';

  async syncEvents(integration: Integration, syncToken?: string): Promise<CalendarSyncResult> {
    const calendar = google.calendar({ version: 'v3', auth: this.getAuth(integration) });
    const params: calendar_v3.Params$Resource$Events$List = {
      calendarId: integration.providerConfig.calendarId || 'primary',
      singleEvents: true,
      syncToken: syncToken || undefined,
      timeMin: syncToken ? undefined : new Date().toISOString(),
      maxResults: 250,
    };

    const response = await calendar.events.list(params);
    // Map Google Calendar events to ExternalEvent format
    // Handle pagination via nextPageToken
    // Return new syncToken for incremental sync
  }

  async setupWebhook(integration: Integration, callbackUrl: string): Promise<WebhookRegistration> {
    // POST to Google Calendar API /watch endpoint
    // Returns channel ID and expiration (max 7 days)
    // Store in integration.providerConfig.webhookChannelId
  }
}
```

Sync flow:
1. Initial sync: fetch all future events, store sync token in `integrations.provider_config`
2. Incremental sync: use stored sync token to fetch only changes since last sync
3. Webhook: Google pushes notification to `/api/webhooks/google-calendar` when events change
4. Webhook handler enqueues a `calendar-sync` BullMQ job for the affected integration
5. Worker performs incremental sync, creating/updating/deleting local meetings as needed
6. Outbound sync: when a meeting is created/updated locally, push to Google Calendar

**Testing**:
- `Unit: mapGoogleEventToExternalEvent → correct field mapping for standard event`
- `Unit: mapGoogleEventToExternalEvent with conference data → conferenceData populated`
- `Unit: mapGoogleEventToExternalEvent with RRULE → rrule field set`
- `Integration (mocked Google API): initial sync → creates local meetings from Google events`
- `Integration (mocked Google API): incremental sync with sync token → only changed events processed`
- `Integration (mocked Google API): webhook notification → sync job enqueued`
- `Integration (mocked Google API): local meeting created → Google event created with correct fields`
- `Integration (mocked Google API): Google API returns 410 Gone → full resync triggered`
- `Fixture: Google API response with 5 events → 5 meetings created with correct mappings`

---

#### 2.3 — Microsoft Graph Calendar Sync

**What**: Two-way synchronization with Outlook/Exchange calendars via Microsoft Graph API, using delta queries and change notifications.

**Design**:

```typescript
// apps/api/src/providers/calendar/microsoft.ts
export class MicrosoftCalendarProvider implements CalendarProvider {
  readonly providerName = 'microsoft_graph';

  async syncEvents(integration: Integration, syncToken?: string): Promise<CalendarSyncResult> {
    const client = this.getGraphClient(integration);
    const endpoint = syncToken
      ? syncToken  // deltaLink IS the sync token for Graph API
      : `/me/calendarView/delta?startDateTime=${new Date().toISOString()}&endDateTime=${endDate}`;

    const response = await client.api(endpoint).get();
    // Map Graph API events to ExternalEvent format
    // Handle @odata.nextLink for pagination
    // Return @odata.deltaLink as nextSyncToken
  }

  async setupWebhook(integration: Integration, callbackUrl: string): Promise<WebhookRegistration> {
    // POST to /subscriptions with resource '/me/events'
    // Graph webhooks expire in max 3 days (calendar resources)
    // Webhook renewal job runs daily
  }
}
```

Key differences from Google Calendar sync:
- Microsoft uses delta queries (`/delta` endpoint) instead of sync tokens
- Change notifications require a validation handshake (lifecycle notification)
- Online meeting creation uses `/me/onlineMeetings` for Teams links
- EWS deprecation (October 1, 2026): this implementation uses Graph API exclusively

**Testing**:
- `Unit: mapGraphEventToExternalEvent → correct field mapping`
- `Unit: mapGraphEventToExternalEvent with Teams meeting → conferenceData includes Teams join URL`
- `Integration (mocked Graph API): delta query sync → local meetings created/updated`
- `Integration (mocked Graph API): change notification → sync job enqueued`
- `Integration (mocked Graph API): subscription renewal → new expiration date`
- `Integration (mocked Graph API): delta query returns @removed → local meeting cancelled`

---

## Phase 3: Recording, Transcription & Storage

### Purpose
Enable audio capture, cloud storage of recordings, and AI-powered transcription. After this phase, meetings can be recorded (via uploaded audio or device capture), stored securely in S3-compatible storage, transcribed with speaker diarization, and the resulting text is searchable. This is the foundation for all AI features.

### Tasks

#### 3.1 — Recording Upload & Storage

**What**: API for uploading meeting recordings, storing them in S3-compatible storage, and managing access via presigned URLs.

**Design**:

```typescript
// apps/api/src/providers/storage/interface.ts
export interface StorageProvider {
  upload(key: string, data: Buffer | Readable, contentType: string): Promise<StorageResult>;
  getPresignedUrl(key: string, expiresInSeconds?: number): Promise<string>;
  delete(key: string): Promise<void>;
  getMetadata(key: string): Promise<StorageMetadata>;
}

export interface StorageResult {
  key: string;
  bucket: string;
  size: number;
  etag: string;
}

// API Routes
// POST   /api/meetings/:id/recordings/upload-url  → get presigned upload URL
// POST   /api/meetings/:id/recordings              → register uploaded recording
// GET    /api/meetings/:id/recordings               → list recordings
// GET    /api/recordings/:id/play-url               → get presigned playback URL
// DELETE /api/recordings/:id                        → delete recording

interface RequestUploadUrlResponse {
  uploadUrl: string;         // presigned S3 PUT URL
  recordingId: string;       // pre-created recording ID
  expiresAt: string;         // upload URL expiration
}

interface RegisterRecordingRequest {
  recordingId: string;
  captureMethod: 'device_audio' | 'bot' | 'upload';
  mediaType: 'audio' | 'video';
  durationSeconds?: number;
}
```

Storage key format: `{orgId}/{meetingId}/{recordingId}.{ext}`

Consent tracking is stored in the recording's `consent_status` JSONB:

```typescript
interface ConsentStatus {
  allConsented: boolean;
  consents: Array<{
    email: string;
    consented: boolean;
    method: 'in_meeting_prompt' | 'pre_meeting_email' | 'org_policy';
    at: string;
  }>;
  pending: string[];  // emails that haven't consented
}
```

**Testing**:
- `Unit: generateStorageKey → correct format with org/meeting/recording path`
- `Unit: requestUploadUrl → presigned URL returned with correct expiration`
- `Integration (MinIO): upload file → file retrievable via presigned download URL`
- `Integration (MinIO): delete file → file no longer accessible`
- `Integration: upload recording for meeting without consent → consent_status.allConsented = false`
- `Integration: register recording with invalid meetingId → 404`
- `E2E: upload audio file via presigned URL → recording listed in meeting`

---

#### 3.2 — Transcription Pipeline

**What**: Async transcription of recordings using Whisper (self-hosted) or Deepgram (cloud), with speaker diarization and segment storage.

**Design**:

```typescript
// apps/api/src/providers/transcription/interface.ts
export interface TranscriptionProvider {
  readonly providerName: string;
  transcribe(audio: TranscriptionInput): Promise<TranscriptionResult>;
  getStatus(jobId: string): Promise<TranscriptionJobStatus>;
}

export interface TranscriptionInput {
  audioUrl: string;          // presigned URL to recording
  language?: string;         // ISO 639-1, or 'auto' for detection
  enableDiarization: boolean;
  maxSpeakers?: number;
}

export interface TranscriptionResult {
  segments: TranscriptSegment[];
  fullText: string;
  detectedLanguage: string;
  durationMs: number;
  accuracyScore?: number;
}

export interface TranscriptSegment {
  segmentIndex: number;
  speakerLabel: string;      // 'Speaker 1', 'Speaker 2', etc.
  startTimeMs: number;
  endTimeMs: number;
  text: string;
  confidence: number;
  words?: Array<{
    word: string;
    startMs: number;
    endMs: number;
    confidence: number;
  }>;
}
```

```typescript
// apps/api/src/workers/transcription.worker.ts
// BullMQ job processor
export async function processTranscription(job: Job<TranscriptionJobData>) {
  const { recordingId, meetingId, organizationId } = job.data;

  // 1. Get recording from DB, generate presigned URL for audio
  // 2. Select transcription provider based on org settings
  // 3. Submit to transcription provider
  // 4. Poll for completion (or receive webhook callback)
  // 5. Store transcript and segments in DB
  // 6. Update recording status to 'ready'
  // 7. Enqueue summarisation and action-extraction jobs
  // 8. Update search index with transcript text
}
```

Job flow: `recording.uploaded` → `transcription.worker` → `summarisation.worker` → `action-extraction.worker`

**Testing**:
- `Unit: WhisperProvider.transcribe with mock HTTP → segments parsed correctly`
- `Unit: DeepgramProvider.transcribe with mock HTTP → segments mapped to common format`
- `Unit: speaker diarization output → speakerLabel correctly assigned per segment`
- `Integration (mocked Whisper): upload recording → transcription job enqueued → segments stored`
- `Integration: transcription completion → summary and action-extraction jobs enqueued`
- `Fixture: 30-second audio sample → transcript with expected text (±10% word error rate)`
- `Fixture: multi-speaker audio → at least 2 distinct speaker labels`

---

#### 3.3 — Transcript API & WebVTT Export

**What**: REST endpoints for accessing transcripts, individual segments, and exporting in WebVTT/SRT formats per RFC standards.

**Design**:

```typescript
// API Routes
// GET    /api/meetings/:id/transcript          → full transcript with segments
// GET    /api/meetings/:id/transcript/export   → WebVTT or SRT export
// GET    /api/meetings/:id/transcript/search   → search within transcript

interface TranscriptResponse {
  id: string;
  meetingId: string;
  language: string;
  engine: string;
  status: 'processing' | 'completed' | 'failed';
  accuracyScore: number | null;
  segmentCount: number;
  segments: TranscriptSegmentResponse[];
  createdAt: string;
}

interface TranscriptExportQuery {
  format: 'webvtt' | 'srt' | 'txt' | 'json';
}

// WebVTT output format (W3C WebVTT standard):
// WEBVTT
//
// 00:00:01.200 --> 00:00:05.300
// <v Speaker 1>I think we should move forward with the PostgreSQL migration.
//
// 00:00:05.800 --> 00:00:09.100
// <v Speaker 2>Agreed. What's the timeline for that?
```

**Testing**:
- `Unit: formatAsWebVTT with 3 segments → valid WebVTT output with speaker tags`
- `Unit: formatAsSRT with 3 segments → valid SRT output with sequential numbering`
- `Unit: transcript search "migration" → segments containing that word returned with timestamps`
- `Integration: GET transcript for completed meeting → all segments returned in order`
- `Integration: GET transcript export as WebVTT → downloadable .vtt file`
- `Integration: GET transcript for meeting without recording → 404`
- `Fixture: known transcript → WebVTT output matches expected file exactly`

---

## Phase 4: AI-Powered Notes, Summaries & Action Items

### Purpose
Build the AI processing pipeline that transforms raw transcripts into structured meeting intelligence: summaries, action items, and decisions. This is the core value proposition that differentiates the platform from a simple recording tool. After this phase, every completed meeting automatically generates an overview summary, bullet-point highlights, extracted action items with assignees, and identified decisions.

### Tasks

#### 4.1 — AI Provider Abstraction & Prompt Management

**What**: Implement a pluggable AI provider interface with Claude as primary and OpenAI as fallback, plus a prompt template management system.

**Design**:

```typescript
// apps/api/src/providers/ai/interface.ts
export interface AIProvider {
  readonly providerName: string;
  complete(request: AICompletionRequest): Promise<AICompletionResponse>;
  completeStreaming(request: AICompletionRequest): AsyncIterable<string>;
}

export interface AICompletionRequest {
  systemPrompt: string;
  userPrompt: string;
  maxTokens?: number;
  temperature?: number;
  responseFormat?: 'text' | 'json';
  cacheable?: boolean;       // enable prompt caching for repeated system prompts
}

export interface AICompletionResponse {
  content: string;
  model: string;
  inputTokens: number;
  outputTokens: number;
  finishReason: 'end_turn' | 'max_tokens' | 'stop';
  durationMs: number;
}

// Prompt templates stored as typed objects
export interface PromptTemplate {
  id: string;
  name: string;
  version: number;
  systemPrompt: string;
  userPromptTemplate: string;  // uses {{variable}} interpolation
  responseSchema?: z.ZodSchema; // for JSON response validation
}
```

```typescript
// apps/api/src/providers/ai/claude.ts
import Anthropic from '@anthropic-ai/sdk';

export class ClaudeProvider implements AIProvider {
  readonly providerName = 'claude';
  private client: Anthropic;

  async complete(request: AICompletionRequest): Promise<AICompletionResponse> {
    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: request.maxTokens ?? 4096,
      system: request.systemPrompt,
      messages: [{ role: 'user', content: request.userPrompt }],
    });
    // Map response to AICompletionResponse
  }
}
```

Prompt caching strategy: system prompts for summarisation and action extraction are stable across invocations, making them excellent candidates for Anthropic prompt caching. The `cacheable: true` flag triggers cache-aware request formatting.

**Testing**:
- `Unit: ClaudeProvider.complete with mock API → response mapped correctly`
- `Unit: OpenAIProvider.complete with mock API → response mapped correctly`
- `Unit: prompt template interpolation → variables replaced, output valid`
- `Unit: JSON response validation against Zod schema → parsed or ValidationError`
- `Integration (mocked API): Claude rate limit → falls back to OpenAI`
- `Integration (mocked API): prompt caching header set when cacheable=true`

---

#### 4.2 — Meeting Summary Generation

**What**: Automatically generate multiple summary formats (overview, bullet points, executive) from transcript text.

**Design**:

```typescript
// Prompt template for meeting summaries
const SUMMARY_SYSTEM_PROMPT = `You are a meeting summarisation assistant. Given a meeting transcript, produce a structured summary. Output JSON matching the provided schema exactly. Focus on decisions made, action items identified, key discussion points, and any unresolved questions.`;

const SUMMARY_USER_TEMPLATE = `Meeting: {{meetingTitle}}
Date: {{meetingDate}}
Participants: {{participants}}
Duration: {{durationMinutes}} minutes

Transcript:
{{transcript}}

Generate a structured summary with the following sections:
1. overview: 2-3 sentence executive summary
2. keyPoints: array of bullet-point highlights (max 10)
3. decisionsIdentified: array of decisions with who decided
4. unresolvedQuestions: array of topics that need follow-up`;

const summaryResponseSchema = z.object({
  overview: z.string(),
  keyPoints: z.array(z.string()),
  decisionsIdentified: z.array(z.object({
    title: z.string(),
    description: z.string().optional(),
    decidedBy: z.string().optional(),
  })),
  unresolvedQuestions: z.array(z.string()),
});
```

```typescript
// apps/api/src/workers/summarisation.worker.ts
export async function processSummarisation(job: Job<SummarisationJobData>) {
  const { meetingId, transcriptId } = job.data;

  // 1. Load transcript text and meeting metadata
  // 2. If transcript > 100K tokens, chunk and summarise in stages
  // 3. Call AI provider with summary prompt
  // 4. Validate response against schema
  // 5. Store in meeting_notes table with is_ai_generated = true
  // 6. Store ai_metadata with model, tokens, generation time
}
```

Long transcript handling: for transcripts exceeding the context window, use a map-reduce approach — summarise each 30-minute chunk, then synthesise chunk summaries into a final summary.

**Testing**:
- `Unit: summary prompt interpolation → all variables replaced`
- `Unit: summary response parsing → valid SummaryResponse object`
- `Unit: summary response with missing fields → validation error, retry with corrected prompt`
- `Integration (mocked AI): transcript input → summary stored in meeting_notes`
- `Integration (mocked AI): long transcript (>100K tokens) → chunked summarisation`
- `Integration: summary generation → ai_metadata includes model name and token counts`
- `Fixture: known transcript → summary contains expected key points`

---

#### 4.3 — Action Item Extraction

**What**: AI-powered extraction of action items from transcripts with automatic assignee detection and due date inference.

**Design**:

```typescript
const ACTION_EXTRACTION_USER_TEMPLATE = `Meeting: {{meetingTitle}}
Participants: {{participants}}

Transcript:
{{transcript}}

Extract all action items mentioned during this meeting. For each action item, identify:
1. title: clear, actionable description
2. assignedTo: email of the person responsible (match from participant list)
3. dueDate: inferred date if mentioned (ISO 8601), or null
4. priority: low, medium, or high based on urgency language
5. sourceSegmentIndex: the transcript segment index where this action was discussed
6. confidence: your confidence in this extraction (0.0 to 1.0)`;

const actionItemsResponseSchema = z.object({
  actionItems: z.array(z.object({
    title: z.string(),
    assignedTo: z.string().email().nullable(),
    dueDate: z.string().nullable(),
    priority: z.enum(['low', 'medium', 'high']),
    sourceSegmentIndex: z.number().int().nullable(),
    confidence: z.number().min(0).max(1),
  })),
});
```

```typescript
// apps/api/src/workers/action-extraction.worker.ts
export async function processActionExtraction(job: Job<ActionExtractionJobData>) {
  const { meetingId, transcriptId, organizationId } = job.data;

  // 1. Load transcript and participant list
  // 2. Call AI provider with action extraction prompt
  // 3. Validate response
  // 4. For each extracted action item:
  //    a. Resolve assignedTo email to user_id (if internal user)
  //    b. Insert into action_items table with source='ai_extracted'
  //    c. Store ai_confidence and source_segment_index
  // 5. Update search index
}
```

**Testing**:
- `Unit: action extraction response parsing → valid ActionItemsResponse`
- `Unit: assignee resolution with known participant → user_id set`
- `Unit: assignee resolution with unknown email → assigned_to_email set, user_id null`
- `Integration (mocked AI): transcript with 3 action items → 3 rows in action_items table`
- `Integration (mocked AI): action item with source_segment_index → links to correct segment`
- `Fixture: transcript saying "Bob, please update the docs by Friday" → action item with assignedTo=bob@acme.com, dueDate=next Friday`

---

#### 4.4 — Action Item & Decision CRUD API

**What**: REST endpoints for managing action items and decisions, including manual creation, status updates, and cross-meeting listing.

**Design**:

```typescript
// Action Item Routes
// POST   /api/action-items                    → create manually
// GET    /api/action-items                    → list across all meetings (with filters)
// GET    /api/meetings/:id/action-items       → list for specific meeting
// PATCH  /api/action-items/:id                → update (status, assignee, due date)
// DELETE /api/action-items/:id                → delete

// Decision Routes
// POST   /api/decisions                       → record manually
// GET    /api/decisions                       → list across all meetings
// GET    /api/meetings/:id/decisions          → list for specific meeting
// PATCH  /api/decisions/:id                   → update
// POST   /api/decisions/:id/supersede         → mark as superseded by new decision

interface ListActionItemsQuery {
  status?: 'open' | 'in_progress' | 'completed' | 'cancelled';
  assignedTo?: string;       // user ID
  meetingId?: string;
  dueBefore?: string;        // ISO 8601 date
  dueAfter?: string;
  priority?: 'low' | 'medium' | 'high';
  source?: 'manual' | 'ai_extracted';
  page?: number;
  limit?: number;
}

interface UpdateActionItemRequest {
  title?: string;
  description?: string;
  assignedTo?: string;       // user ID
  dueDate?: string;
  priority?: 'low' | 'medium' | 'high';
  status?: 'open' | 'in_progress' | 'completed' | 'cancelled';
}

interface SupersedeDecisionRequest {
  newDecisionTitle: string;
  newDecisionDescription?: string;
  reason?: string;
}
```

**Testing**:
- `Unit: createActionItem manually → source='manual', ai_confidence=null`
- `Unit: listActionItems with status filter → only matching items returned`
- `Unit: listActionItems with assignedTo filter → only items for that user`
- `Unit: updateActionItem status to completed → completedAt timestamp set`
- `Unit: supersedeDecision → old decision's supersedes_decision_id set, new decision created`
- `Integration: list action items across meetings → items from multiple meetings returned`
- `Integration: action item due date filtering → overdue items queryable`
- `Integration: decision chain via supersedes → full chain retrievable`

---

## Phase 5: Agenda Builder & Pre-Meeting Preparation

### Purpose
Add pre-meeting workflow: collaborative agenda creation, AI-suggested agenda items based on previous meetings, and meeting preparation briefs. This addresses a key gap identified in competitor analysis — most tools focus on post-meeting notes, not pre-meeting readiness. After this phase, users can build agendas, receive AI-generated preparation materials, and have context from prior meetings surfaced automatically.

### Tasks

#### 5.1 — Agenda CRUD & Collaborative Editing

**What**: API and real-time collaborative editing for meeting agendas with nested items, time allocations, and presenter assignments.

**Design**:

```typescript
// API Routes
// POST   /api/meetings/:id/agenda             → create agenda
// GET    /api/meetings/:id/agenda             → get agenda with items
// POST   /api/meetings/:id/agenda/items       → add agenda item
// PATCH  /api/agenda-items/:id                → update item
// DELETE /api/agenda-items/:id                → remove item
// PATCH  /api/meetings/:id/agenda/reorder     → reorder items

interface CreateAgendaItemRequest {
  title: string;
  description?: string;
  durationMinutes?: number;
  presenterUserId?: string;
  parentItemId?: string;     // for nested items
  sortOrder?: number;
}

interface AgendaResponse {
  meetingId: string;
  items: AgendaItemResponse[];
  totalDurationMinutes: number;
  isShared: boolean;
}

interface AgendaItemResponse {
  id: string;
  title: string;
  description: string | null;
  durationMinutes: number | null;
  presenter: { id: string; displayName: string } | null;
  status: 'pending' | 'discussed' | 'skipped' | 'deferred';
  sortOrder: number;
  children: AgendaItemResponse[];  // nested items
  extras: {
    linkedActionItems?: string[];
    linkedDocuments?: Array<{ url: string; title: string }>;
    aiSuggested?: boolean;
    aiContext?: string;
  };
}
```

Real-time collaborative editing via Socket.IO:

```typescript
// Socket.IO events for agenda collaboration
// Client → Server:
// 'agenda:join' { meetingId }           → join the agenda editing room
// 'agenda:item:add' { ... }            → add item (broadcast to room)
// 'agenda:item:update' { id, changes } → update item (broadcast)
// 'agenda:item:reorder' { items }      → reorder (broadcast)

// Server → Client:
// 'agenda:updated' { agenda }          → full agenda state after change
// 'agenda:user:joined' { user }        → another user joined editing
// 'agenda:user:left' { user }          → another user left editing
```

**Testing**:
- `Unit: createAgendaItem with valid input → item inserted with correct sort order`
- `Unit: createAgendaItem with parentItemId → nested under parent`
- `Unit: reorderItems → sort_order updated for all affected items`
- `Unit: agenda totalDurationMinutes → sum of all item durations`
- `Integration: add 3 items, reorder → items returned in new order`
- `Integration (Socket.IO): two clients join room, one adds item → other receives broadcast`
- `Integration: mark item as discussed → status updated, children accessible`

---

#### 5.2 — AI Agenda Suggestion Engine

**What**: Automatically suggest agenda items for recurring meetings based on open action items, previous meeting decisions, and unresolved topics.

**Design**:

```typescript
// apps/api/src/services/agenda-suggestion.service.ts
export class AgendaSuggestionService {
  async suggestAgendaItems(meetingId: string): Promise<SuggestedAgendaItem[]> {
    // 1. Identify if this meeting is part of a recurring series
    // 2. Load previous meeting(s) in the series:
    //    - Open action items from last meeting
    //    - Deferred agenda items from last meeting
    //    - Unresolved questions from last meeting's summary
    //    - Decisions that may need review
    // 3. Load current context:
    //    - Open action items assigned to participants
    //    - Upcoming deadlines for participant action items
    // 4. Call AI provider with context to generate suggested items
    // 5. Return suggestions with confidence scores and context
  }
}

interface SuggestedAgendaItem {
  title: string;
  description: string;
  suggestedDurationMinutes: number;
  reason: string;              // why this was suggested
  confidence: number;          // 0.0 to 1.0
  sourceType: 'open_action_item' | 'deferred_item' | 'unresolved_question' | 'upcoming_deadline' | 'ai_suggested';
  sourceId?: string;           // ID of the source entity
}
```

```typescript
const AGENDA_SUGGESTION_PROMPT = `You are a meeting preparation assistant. Based on the following context from previous meetings and current open items, suggest agenda topics for an upcoming meeting.

Previous meeting summary:
{{previousSummary}}

Open action items for participants:
{{openActionItems}}

Deferred agenda items:
{{deferredItems}}

Unresolved questions:
{{unresolvedQuestions}}

Suggest 3-7 agenda items. For each, explain why it should be discussed and estimate time needed.`;
```

**Testing**:
- `Unit: suggestAgendaItems for recurring meeting → includes open action items as suggestions`
- `Unit: suggestAgendaItems with deferred items from last meeting → deferred items re-suggested`
- `Unit: suggestAgendaItems for first meeting in series → generic suggestions only`
- `Integration (mocked AI): meeting with 3 open action items → at least 3 suggestions returned`
- `Integration: accept suggestion → agenda item created with aiSuggested=true and aiContext`
- `Fixture: series with 3 previous meetings → suggestions reference specific past discussions`

---

#### 5.3 — Pre-Meeting Brief Generation

**What**: Generate a preparation brief for each meeting that surfaces relevant context, participant backgrounds, and historical decisions related to agenda topics.

**Design**:

```typescript
// API Routes
// GET    /api/meetings/:id/brief              → get or generate pre-meeting brief
// POST   /api/meetings/:id/brief/regenerate   → force regeneration

interface MeetingBrief {
  meetingId: string;
  generatedAt: string;
  sections: {
    overview: string;                          // meeting purpose and context
    participantContext: Array<{
      name: string;
      email: string;
      recentMeetings: number;                  // meetings attended in last 30 days
      openActionItems: number;
      relevantDecisions: string[];             // recent decisions they were involved in
    }>;
    agendaPrep: Array<{
      agendaItemTitle: string;
      context: string;                         // AI-generated context for this topic
      relatedDecisions: Array<{ title: string; date: string; meetingTitle: string }>;
      relatedActionItems: Array<{ title: string; status: string; assignee: string }>;
    }>;
    suggestedReadings: Array<{ title: string; url: string; relevance: string }>;
  };
}
```

**Testing**:
- `Unit: generateBrief for meeting with agenda → agendaPrep section populated for each item`
- `Unit: generateBrief for meeting without agenda → overview only, no agendaPrep`
- `Unit: participantContext → correct action item counts per participant`
- `Integration (mocked AI): brief generation → stored as meeting_note with is_ai_generated=true`
- `Integration: regenerate brief → new version created, old one retained`

---

## Phase 6: Full-Text Search & Conversational AI

### Purpose
Implement comprehensive search across all meeting content and a conversational AI interface that lets users ask questions about their meeting history. This powers the "ask about past meetings" capability that differentiates the platform from transcript-only tools.

### Tasks

#### 6.1 — Search Index & Full-Text Search API

**What**: Populate and query the PostgreSQL TSVECTOR-based search index across meetings, transcripts, action items, decisions, and notes.

**Design**:

```typescript
// apps/api/src/services/search.service.ts
export class SearchService {
  async indexEntity(entity: SearchableEntity): Promise<void> {
    // Insert/upsert into search_index table:
    // - entity_type: 'meeting' | 'transcript' | 'action_item' | 'decision' | 'note'
    // - entity_id: UUID of the source entity
    // - meeting_id: associated meeting
    // - searchable_text: concatenated text content
    // - facets: { date, participants, tags, has_recording, has_action_items }
    // PostgreSQL auto-generates search_vector via TSVECTOR generated column
  }

  async search(query: SearchQuery): Promise<SearchResults> {
    // Use ts_query with websearch syntax for natural language queries
    // Apply facet filters via JSONB containment operators
    // Rank by ts_rank_cd for relevance scoring
    // Include snippet via ts_headline for context
  }
}

// API Routes
// GET /api/search?q=...&type=...&dateFrom=...&dateTo=...

interface SearchQuery {
  query: string;              // natural language search query
  entityTypes?: string[];     // filter by type
  dateFrom?: string;
  dateTo?: string;
  participantEmail?: string;
  tags?: string[];
  hasRecording?: boolean;
  page?: number;
  limit?: number;
}

interface SearchResults {
  total: number;
  results: Array<{
    entityType: string;
    entityId: string;
    meetingId: string;
    meetingTitle: string;
    meetingDate: string;
    snippet: string;          // highlighted matching text
    relevanceScore: number;
    facets: Record<string, unknown>;
  }>;
  facetCounts: {
    entityTypes: Record<string, number>;
    dateRanges: Record<string, number>;
  };
}
```

```sql
-- Search query using websearch_to_tsquery for natural language input
SELECT
  si.entity_type,
  si.entity_id,
  si.meeting_id,
  m.title as meeting_title,
  m.starts_at as meeting_date,
  ts_headline('english', si.searchable_text, websearch_to_tsquery('english', $1),
    'StartSel=<mark>, StopSel=</mark>, MaxWords=50, MinWords=20') as snippet,
  ts_rank_cd(si.search_vector, websearch_to_tsquery('english', $1)) as relevance_score
FROM search_index si
JOIN meetings m ON m.id = si.meeting_id
WHERE si.organization_id = $2
  AND si.search_vector @@ websearch_to_tsquery('english', $1)
ORDER BY relevance_score DESC
LIMIT $3 OFFSET $4;
```

**Testing**:
- `Unit: indexEntity with meeting → search_index row created with correct searchable_text`
- `Unit: indexEntity with transcript → full transcript text indexed`
- `Unit: search "database migration" → meetings discussing that topic returned`
- `Unit: search with entityType filter → only matching types returned`
- `Unit: search with date range → only meetings in range returned`
- `Integration: index 10 meetings, search for specific term → correct meeting(s) returned with snippets`
- `Integration: search with no results → empty results, total=0`
- `Fixture: known corpus of 5 meetings → search queries return expected results`

---

#### 6.2 — Conversational AI (Ask About Meetings)

**What**: Chat interface where users ask natural language questions about their meeting history and receive AI-generated answers grounded in actual meeting data.

**Design**:

```typescript
// API Routes
// POST   /api/chat                            → send message, get AI response
// GET    /api/chat/conversations               → list past conversations
// GET    /api/chat/conversations/:id           → get conversation history

interface ChatRequest {
  message: string;
  conversationId?: string;    // continue existing conversation
}

interface ChatResponse {
  conversationId: string;
  message: string;
  sources: Array<{
    entityType: string;
    entityId: string;
    meetingTitle: string;
    meetingDate: string;
    snippet: string;
  }>;
}
```

RAG (Retrieval-Augmented Generation) pipeline:

```typescript
// apps/api/src/services/chat.service.ts
export class ChatService {
  async processMessage(userId: string, request: ChatRequest): Promise<ChatResponse> {
    // 1. Retrieve conversation history (if continuing)
    // 2. Classify query intent:
    //    - Factual: "What was decided about the API versioning?"
    //    - Temporal: "What happened in last week's standup?"
    //    - People: "What action items does Bob have?"
    //    - Aggregation: "How many meetings did I have this month?"
    // 3. Generate search queries from the user's question
    // 4. Execute search against search_index
    // 5. Retrieve full context for top results (transcript segments, notes, action items)
    // 6. Call AI provider with:
    //    - System prompt: "Answer questions about meetings using only the provided context"
    //    - Retrieved context as grounding data
    //    - User's question
    //    - Conversation history
    // 7. Extract source citations from the AI response
    // 8. Store conversation turn
    // 9. Return response with sources
  }
}
```

**Testing**:
- `Unit: query classification → "What was decided" → factual intent`
- `Unit: search query generation from natural language → relevant search terms`
- `Integration (mocked AI): "What action items does Bob have?" → response lists Bob's action items with meeting sources`
- `Integration (mocked AI): conversation continuation → context from prior turns included`
- `Integration: question about meeting with no transcript → response says no data available`
- `E2E: type question in chat → response with source links to specific meetings`

---

## Phase 7: Speaker Analytics & Meeting Quality

### Purpose
Add speaker-level analytics (talk time, sentiment, participation balance) and meeting quality scoring with coaching suggestions. This addresses the underserved meeting quality coaching opportunity and provides managers with data-driven insights on meeting effectiveness.

### Tasks

#### 7.1 — Speaker Analytics Pipeline

**What**: Compute per-speaker metrics from transcript segments: talk time, word count, question count, interruptions, and sentiment scores.

**Design**:

```typescript
// apps/api/src/workers/analytics.worker.ts
export async function processAnalytics(job: Job<AnalyticsJobData>) {
  const { meetingId, transcriptId } = job.data;

  // 1. Load all transcript segments for the meeting
  // 2. Group segments by speaker_label
  // 3. For each speaker, compute:
  //    - talkTimeSeconds: sum of (endTimeMs - startTimeMs) / 1000
  //    - wordCount: count of words across all segments
  //    - questionCount: segments ending with '?' or classified as questions
  //    - interruptionCount: segments that start before previous speaker's segment ends
  // 4. Call AI provider for sentiment analysis on each speaker's aggregated text
  // 5. Store per-speaker metrics in meeting_participants.participation_data JSONB
  // 6. Compute meeting-level quality metrics
  // 7. Store in meeting_analytics table
}

interface SpeakerMetrics {
  speakerLabel: string;
  speakerEmail: string | null;
  talkTimeSeconds: number;
  talkTimePercentage: number;
  wordCount: number;
  questionCount: number;
  interruptionCount: number;
  sentimentScore: number;       // -1.0 (negative) to 1.0 (positive)
  engagementScore: number;      // 0.0 to 1.0
}
```

**Testing**:
- `Unit: computeTalkTime from 5 segments → correct total seconds`
- `Unit: computeInterruptions with overlapping segments → interruption detected`
- `Unit: computeQuestionCount → segments ending with '?' counted`
- `Unit: talkTimePercentage → percentages sum to ~100%`
- `Integration (mocked AI): sentiment analysis → scores stored per participant`
- `Fixture: 3-speaker transcript → participation_data populated for each`

---

#### 7.2 — Meeting Quality Scoring & Coaching

**What**: Compute overall meeting quality scores and generate AI-powered coaching suggestions for meeting organizers.

**Design**:

```typescript
interface MeetingQualityMetrics {
  participationBalance: number;  // 0.0 (one person dominated) to 1.0 (perfectly even)
  decisionCount: number;
  actionItemCount: number;
  totalDurationSeconds: number;
  agendaAdherence: number | null;  // 0.0 to 1.0 (if agenda exists)
  overallScore: number;            // 0.0 to 1.0
  detailedMetrics: {
    speakerDistribution: Record<string, number>;   // email → percentage
    topicDistribution: Array<{ topic: string; percentage: number }>;
    engagementOverTime: Array<{ minute: number; score: number }>;
    coachingSuggestions: string[];
  };
}
```

```typescript
// API Routes
// GET /api/meetings/:id/analytics        → meeting quality and speaker analytics
// GET /api/analytics/dashboard            → org-wide meeting analytics summary

interface AnalyticsDashboardResponse {
  period: { from: string; to: string };
  totalMeetings: number;
  totalMeetingHours: number;
  averageQualityScore: number;
  averageDurationMinutes: number;
  averageParticipantCount: number;
  actionItemCompletionRate: number;
  topCollaborators: Array<{ name: string; meetingCount: number }>;
  meetingsByDay: Array<{ date: string; count: number; totalHours: number }>;
  qualityTrend: Array<{ week: string; avgScore: number }>;
}
```

Participation balance formula: `1 - standardDeviation(talkTimePercentages) / maxPossibleDeviation`

Coaching suggestion generation:

```typescript
const COACHING_PROMPT = `Analyse these meeting metrics and provide 2-4 brief, actionable coaching suggestions:

Meeting: {{meetingTitle}} ({{durationMinutes}} min, {{participantCount}} participants)
Speaker distribution: {{speakerDistribution}}
Questions asked: {{questionCount}}
Decisions made: {{decisionCount}}
Action items: {{actionItemCount}}
Agenda items covered: {{agendaItemsCovered}}/{{agendaItemsTotal}}

Focus on participation balance, decision efficiency, and actionable improvements.`;
```

**Testing**:
- `Unit: participationBalance with equal talk time → ~1.0`
- `Unit: participationBalance with one person at 80% → < 0.5`
- `Unit: agendaAdherence with 4/5 items discussed → 0.8`
- `Unit: overallScore calculation → weighted composite of sub-scores`
- `Integration (mocked AI): coaching suggestions → 2-4 actionable suggestions returned`
- `Integration: dashboard for org with 20 meetings → correct aggregates`
- `Fixture: meeting where one person spoke 90% → coaching suggests "encourage broader participation"`

---

## Phase 8: CRM & Project Management Integrations

### Purpose
Connect the platform to external business tools: auto-log meetings to Salesforce/HubSpot CRMs and sync action items to Jira/Asana/Monday.com. CRM auto-logging is identified as the most commercially valuable integration for sales-motion buyers.

### Tasks

#### 8.1 — CRM Integration Framework & Salesforce Connector

**What**: Build the CRM sync framework and implement Salesforce integration for auto-logging meeting notes and action items.

**Design**:

```typescript
// apps/api/src/providers/crm/interface.ts
export interface CRMProvider {
  readonly providerName: string;
  logMeeting(integration: Integration, meeting: MeetingWithSummary): Promise<CRMSyncResult>;
  createTask(integration: Integration, actionItem: ActionItem): Promise<CRMSyncResult>;
  updateTask(integration: Integration, actionItem: ActionItem, externalId: string): Promise<void>;
  findContact(integration: Integration, email: string): Promise<CRMContact | null>;
}

export interface CRMSyncResult {
  externalRecordId: string;
  externalRecordUrl: string;
  syncedFields: string[];
}

// apps/api/src/providers/crm/salesforce.ts
export class SalesforceProvider implements CRMProvider {
  readonly providerName = 'salesforce';

  async logMeeting(integration: Integration, meeting: MeetingWithSummary): Promise<CRMSyncResult> {
    const conn = this.getConnection(integration);
    const fieldMapping = integration.providerConfig.fieldMapping || DEFAULT_SALESFORCE_MAPPING;

    // 1. Find related Contact/Lead records for each participant
    // 2. Create an Activity (Event) record with mapped fields:
    //    - Subject → meeting title
    //    - Description → AI-generated summary
    //    - WhoId → primary contact
    //    - WhatId → related opportunity (if configured)
    //    - ActivityDate → meeting date
    // 3. Create Task records for action items assigned to Salesforce contacts
    // 4. Return external record IDs
  }
}
```

CRM sync worker:

```typescript
// apps/api/src/workers/crm-sync.worker.ts
// Triggered after meeting summarisation completes
export async function processCRMSync(job: Job<CRMSyncJobData>) {
  const { meetingId, organizationId } = job.data;

  // 1. Check if org has active CRM integration
  // 2. Load meeting with summary and action items
  // 3. Call CRM provider to log meeting
  // 4. Store sync result in action_items.external_sync JSONB
  // 5. Log sync event in audit_log
  // 6. Handle errors: rate limits → retry with backoff; auth errors → deactivate integration
}
```

**Testing**:
- `Unit: SalesforceProvider.logMeeting with mock API → Activity created with correct field mapping`
- `Unit: SalesforceProvider.createTask with mock API → Task created, linked to Contact`
- `Unit: findContact with known email → Contact returned`
- `Unit: findContact with unknown email → null`
- `Integration (mocked Salesforce): meeting sync → external_sync JSONB updated on action items`
- `Integration (mocked Salesforce): rate limit error → job retried with exponential backoff`
- `Integration (mocked Salesforce): auth token expired → token refreshed, request retried`

---

#### 8.2 — HubSpot CRM Connector

**What**: Implement HubSpot CRM integration using the HubSpot API for meeting logging and task creation.

**Design**:

```typescript
// apps/api/src/providers/crm/hubspot.ts
export class HubSpotProvider implements CRMProvider {
  readonly providerName = 'hubspot';

  async logMeeting(integration: Integration, meeting: MeetingWithSummary): Promise<CRMSyncResult> {
    // 1. Find associated Contacts by email via HubSpot search API
    // 2. Create an Engagement of type 'MEETING' with:
    //    - Subject → meeting title
    //    - Body → HTML-formatted summary
    //    - Attendees → HubSpot contact IDs
    //    - Timestamp → meeting start time
    // 3. Associate engagement with Contact records
    // 4. Create Tasks for action items
  }
}
```

**Testing**:
- `Unit: HubSpotProvider.logMeeting with mock API → Engagement created`
- `Unit: HubSpotProvider.createTask → Task created with association to Contact`
- `Integration (mocked HubSpot): full sync flow → meeting logged, tasks created, sync JSONB updated`

---

#### 8.3 — Project Management Tool Sync

**What**: Sync action items bidirectionally with Jira, Asana, and Monday.com.

**Design**:

```typescript
// apps/api/src/providers/pm/interface.ts
export interface PMToolProvider {
  readonly providerName: string;
  createTask(integration: Integration, actionItem: ActionItem): Promise<PMSyncResult>;
  updateTask(integration: Integration, externalId: string, updates: Partial<ActionItem>): Promise<void>;
  getTaskStatus(integration: Integration, externalId: string): Promise<string>;
  listProjects(integration: Integration): Promise<ExternalProject[]>;
}

export interface PMSyncResult {
  externalTaskId: string;
  externalTaskUrl: string;
}

// PM provider implementations follow the same pattern:
// - JiraProvider: uses Jira REST API v3
// - AsanaProvider: uses Asana REST API
// - MondayProvider: uses Monday.com GraphQL API
```

Bidirectional sync: when an action item status changes locally, push to external tool. When external tool webhook fires, pull status change to local action item.

```typescript
// Webhook route for PM tool callbacks
// POST /api/webhooks/jira      → Jira webhook
// POST /api/webhooks/asana     → Asana webhook
// POST /api/webhooks/monday    → Monday.com webhook
```

**Testing**:
- `Unit: JiraProvider.createTask → Jira issue created with correct issue type`
- `Unit: AsanaProvider.createTask → Asana task created in configured project`
- `Integration (mocked Jira): action item completed locally → Jira issue transitioned`
- `Integration (mocked Jira): Jira webhook with status change → local action item status updated`
- `Integration: action item synced to both Jira and Salesforce → external_sync JSONB has both entries`

---

## Phase 9: Frontend Application

### Purpose
Build the Next.js web application that provides the user-facing experience: meeting dashboard, agenda editor, transcript viewer, action item tracker, search, analytics, and settings. The frontend consumes all APIs built in previous phases.

### Tasks

#### 9.1 — Application Shell & Dashboard

**What**: Next.js application with authentication flow, navigation, organization switching, and a dashboard showing upcoming meetings, recent action items, and meeting stats.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/page.tsx — Dashboard
// Sections:
// 1. Today's Meetings — list of meetings for today with status indicators
// 2. Upcoming Meetings — next 5 meetings with agenda status
// 3. My Action Items — open items assigned to current user
// 4. Recent Decisions — last 5 decisions across all meetings
// 5. Quick Stats — meetings this week, action items completed, meeting hours

// Component hierarchy:
// <DashboardLayout>
//   <DashboardHeader title="Dashboard" />
//   <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
//     <TodaysMeetings meetings={todayMeetings} />
//     <UpcomingMeetings meetings={upcoming} />
//     <MyActionItems items={actionItems} />
//     <RecentDecisions decisions={decisions} />
//     <QuickStats stats={stats} />
//   </div>
// </DashboardLayout>
```

```typescript
// apps/web/src/lib/api-client.ts
// Type-safe API client using fetch with auth token injection
export class ApiClient {
  constructor(private baseUrl: string, private getToken: () => Promise<string>) {}

  async get<T>(path: string, params?: Record<string, string>): Promise<T> {
    const token = await this.getToken();
    const url = new URL(path, this.baseUrl);
    if (params) Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v));
    const res = await fetch(url, { headers: { Authorization: `Bearer ${token}` } });
    if (!res.ok) throw new ApiError(res.status, await res.json());
    return res.json();
  }

  async post<T>(path: string, body: unknown): Promise<T> { /* ... */ }
  async patch<T>(path: string, body: unknown): Promise<T> { /* ... */ }
  async delete(path: string): Promise<void> { /* ... */ }
}
```

**Testing**:
- `E2E: login → dashboard loads with today's meetings`
- `E2E: dashboard shows correct action item count for current user`
- `E2E: click on meeting → navigates to meeting detail page`
- `E2E: organization switcher → dashboard reloads with new org's data`
- `Unit (component): TodaysMeetings with 0 meetings → "No meetings today" message`
- `Unit (component): MyActionItems with 3 items → 3 items rendered with status badges`

---

#### 9.2 — Meeting Detail & Transcript Viewer

**What**: Meeting detail page with tabs for agenda, transcript (with speaker highlighting and timestamp links), notes, action items, and recording playback.

**Design**:

```typescript
// apps/web/src/app/(meeting)/[meetingId]/page.tsx
// Tab layout:
// [Overview] [Agenda] [Transcript] [Notes] [Action Items] [Analytics]

// Transcript viewer component features:
// - Speaker-colored segments
// - Click timestamp → seek recording to that point
// - Search within transcript with highlight
// - Toggle between full text and speaker-by-speaker view
// - Export buttons (WebVTT, SRT, TXT)

// apps/web/src/components/transcript/TranscriptViewer.tsx
interface TranscriptViewerProps {
  segments: TranscriptSegmentResponse[];
  speakers: Map<string, { label: string; color: string }>;
  onTimestampClick?: (timeMs: number) => void;
  searchQuery?: string;
}
```

Recording player with transcript sync:

```typescript
// apps/web/src/components/meeting/RecordingPlayer.tsx
// - Audio/video player using HTML5 <audio> / <video>
// - Presigned URL loaded from API
// - Current playback position synced with transcript viewer
// - Clicking a transcript segment seeks to that timestamp
```

**Testing**:
- `E2E: navigate to meeting with transcript → segments displayed with speaker names`
- `E2E: click transcript timestamp → recording seeks to correct position`
- `E2E: search within transcript → matching segments highlighted`
- `E2E: export as WebVTT → file downloads with .vtt extension`
- `Unit (component): TranscriptViewer with 10 segments → all rendered in order`
- `Unit (component): speaker color assignment → consistent colors per speaker`

---

#### 9.3 — Agenda Editor & Action Item Manager

**What**: Interactive agenda editor with drag-and-drop reordering, time allocation, and AI suggestion panel. Action item board with filtering, status updates, and due date tracking.

**Design**:

```typescript
// Agenda editor features:
// - Drag-and-drop reorder (using @dnd-kit/core)
// - Inline editing of title, description, duration
// - Presenter assignment dropdown
// - Nested item support (drag item into another to nest)
// - AI suggestion sidebar: "Suggested items based on previous meetings"
// - Total duration counter with overrun warning

// Action item manager features:
// - Kanban board view (open → in_progress → completed)
// - List view with sortable columns
// - Filter by assignee, meeting, due date, priority
// - Inline status update (click to cycle status)
// - Overdue highlighting
// - Bulk actions (assign, set priority, complete)
```

**Testing**:
- `E2E: drag agenda item to new position → order updated, persisted on refresh`
- `E2E: add nested agenda item → renders indented under parent`
- `E2E: accept AI suggestion → item added to agenda with AI badge`
- `E2E: action item Kanban → drag item from "open" to "completed" → status updated`
- `E2E: filter action items by assignee → only matching items shown`
- `E2E: overdue action item → highlighted in red`

---

#### 9.4 — Search Page & Chat Interface

**What**: Global search with faceted filtering and a conversational AI chat panel.

**Design**:

```typescript
// Search page: /search
// - Search input with instant results
// - Facet sidebar: entity type, date range, participants, tags
// - Result cards with highlighted snippets
// - Click result → navigate to source entity

// Chat interface: /chat or slide-out panel
// - Conversation list in sidebar
// - Message input with send button
// - AI responses with inline source citations (clickable links to meetings)
// - Conversation history retained across sessions
```

**Testing**:
- `E2E: type search query → results appear with highlighted snippets`
- `E2E: apply entity type facet → results filtered`
- `E2E: click search result → navigates to meeting transcript at relevant segment`
- `E2E: ask question in chat → AI response with source citations`
- `E2E: click source citation → navigates to meeting detail`
- `E2E: continue conversation → previous context retained`

---

#### 9.5 — Settings & Integration Management

**What**: Organization settings, user preferences, and integration connection management (OAuth flows for calendar, CRM, and PM tools).

**Design**:

```typescript
// Settings pages:
// /settings/organization   → name, timezone, plan, data retention policies
// /settings/members        → invite, role management, remove
// /settings/integrations   → connect/disconnect calendar, CRM, PM tools
// /settings/profile        → user display name, avatar, timezone, notification preferences
// /settings/security       → password change, 2FA, active sessions

// Integration connection flow:
// 1. User clicks "Connect Google Calendar"
// 2. Frontend redirects to /api/integrations/google/connect
// 3. API initiates OAuth flow with required scopes
// 4. Provider consent screen → callback
// 5. API stores tokens, redirects to settings with success message
// 6. Integration appears in connected list with sync status
```

**Testing**:
- `E2E: connect Google Calendar → OAuth flow → integration listed as connected`
- `E2E: disconnect integration → removed from list, OAuth tokens deleted`
- `E2E: invite member → invitation email sent, member appears as pending`
- `E2E: change organization timezone → meetings display in new timezone`
- `E2E: update data retention policy → confirmation dialog → saved`

---

## Phase 10: Knowledge Graph & Cross-Meeting Intelligence

### Purpose
Add the graph-relational layer (from Data Model Suggestion 4) to enable cross-meeting knowledge synthesis, longitudinal decision tracking, and topic-based expertise identification. This is the key differentiator that transforms the platform from a note-taking tool into a meeting knowledge graph. After this phase, users can see decision chains across meetings, identify topic experts, and receive weekly synthesis reports.

### Tasks

#### 10.1 — Graph Schema & Sync Infrastructure

**What**: Add graph_nodes and graph_edges tables, create triggers to sync relational data into the graph, and implement the graph query layer.

**Design**:

```sql
-- Migration: add graph tables
CREATE EXTENSION IF NOT EXISTS vector;  -- pgvector for embeddings

CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    node_type VARCHAR(50) NOT NULL,
    entity_id UUID,
    label VARCHAR(500) NOT NULL,
    properties JSONB DEFAULT '{}',
    embedding VECTOR(1536),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_graph_nodes_org_type ON graph_nodes (organization_id, node_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes (entity_id) WHERE entity_id IS NOT NULL;

CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    source_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type VARCHAR(50) NOT NULL,
    weight DECIMAL(5,4) DEFAULT 1.0,
    properties JSONB DEFAULT '{}',
    valid_from TIMESTAMPTZ DEFAULT now(),
    valid_to TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_graph_edges_source ON graph_edges (source_node_id, edge_type);
CREATE INDEX idx_graph_edges_target ON graph_edges (target_node_id, edge_type);
```

```typescript
// apps/api/src/services/graph.service.ts
export class GraphService {
  async syncMeetingToGraph(meetingId: string): Promise<void> {
    // 1. Create/update meeting node
    // 2. Create/update person nodes for each participant
    // 3. Create 'attended' edges from person → meeting
    // 4. Create 'organised' edge from organizer → meeting
  }

  async syncDecisionToGraph(decisionId: string): Promise<void> {
    // 1. Create decision node
    // 2. Create 'decided_in' edge from decision → meeting
    // 3. Create 'decided_by' edge from decision → person
    // 4. If supersedes another decision, create 'supersedes' edge
  }

  async extractTopicsFromTranscript(meetingId: string, transcriptId: string): Promise<void> {
    // 1. Call AI to extract topics and keywords from transcript
    // 2. Create/update topic nodes (deduplicate by canonical name)
    // 3. Create 'discussed' edges from meeting → topic with weight based on duration
    // 4. Create 'mentioned' edges from person → topic with weight based on talk time
    // 5. Generate embeddings for meeting and topic nodes via AI
  }

  async computeCollaborationEdges(organizationId: string): Promise<void> {
    // Batch job: compute 'collaborates_with' edges from co-attendance patterns
    // Weight = normalized count of shared meetings in last 90 days
  }

  async computeExpertiseEdges(organizationId: string): Promise<void> {
    // Batch job: compute 'expert_in' edges from speaking time on topics
    // Weight = normalized total talk time on topic across all meetings
  }
}
```

**Testing**:
- `Unit: syncMeetingToGraph → meeting node + person nodes + attended edges created`
- `Unit: syncDecisionToGraph with supersedes → supersedes edge created`
- `Integration: meeting with 3 participants synced → 4 nodes, 3 attended edges`
- `Integration (mocked AI): topic extraction → topic nodes and discussed edges created`
- `Integration: computeCollaborationEdges → edges created between co-attendees`

---

#### 10.2 — Longitudinal Decision Tracking

**What**: API and UI for tracking decision chains across meetings — when a decision supersedes a previous one, surface the full chain.

**Design**:

```typescript
// API Routes
// GET /api/decisions/:id/chain       → full decision chain (recursive)
// GET /api/decisions/active           → all currently active (non-superseded) decisions
// GET /api/topics/:topicId/decisions  → decisions related to a topic

interface DecisionChainResponse {
  current: DecisionResponse;
  predecessors: Array<{
    decision: DecisionResponse;
    meetingTitle: string;
    meetingDate: string;
    depth: number;
  }>;
}
```

Decision chain query using recursive CTE on graph edges (as defined in Data Model Suggestion 4).

**Testing**:
- `Unit: decision chain with 3 levels → all predecessors returned in order`
- `Unit: active decisions → excludes superseded decisions`
- `Integration: create decision → supersede it → chain shows both`
- `E2E: view decision detail → "Previous decisions" section shows chain`

---

#### 10.3 — Cross-Meeting Knowledge Synthesis

**What**: Weekly AI-generated synthesis of decisions, action items, and emerging themes across all meetings for a user or team.

**Design**:

```typescript
// Scheduled job: runs weekly (configurable per org)
export async function generateWeeklySynthesis(organizationId: string, userId: string): Promise<WeeklySynthesis> {
  // 1. Query graph for all meetings user attended in the past 7 days
  // 2. Collect: decisions made, action items created, topics discussed
  // 3. Query graph for topic trends (new topics, recurring topics, fading topics)
  // 4. Call AI with aggregated context to generate synthesis
  // 5. Store synthesis and send via email/Slack notification
}

interface WeeklySynthesis {
  period: { from: string; to: string };
  userId: string;
  meetingCount: number;
  meetingHours: number;
  keyDecisions: Array<{ title: string; meetingTitle: string; date: string }>;
  actionItemsSummary: {
    created: number;
    completed: number;
    overdue: number;
    topItems: Array<{ title: string; assignee: string; dueDate: string }>;
  };
  emergingThemes: Array<{
    topic: string;
    meetingsDiscussed: number;
    trend: 'rising' | 'stable' | 'fading';
    context: string;
  }>;
  aiInsights: string;  // AI-generated narrative summary
}

// API Routes
// GET /api/synthesis/weekly              → latest weekly synthesis for current user
// GET /api/synthesis/weekly/history      → past syntheses
// POST /api/synthesis/weekly/generate    → trigger manual generation
```

**Testing**:
- `Unit: generateWeeklySynthesis with 5 meetings → synthesis includes all 5`
- `Unit: emerging themes detection → topics discussed in 3+ meetings flagged as recurring`
- `Integration (mocked AI): synthesis generation → stored and retrievable`
- `Integration: synthesis with 0 meetings → "No meetings this week" response`
- `E2E: view weekly synthesis → decisions, themes, and action items displayed`

---

## Phase 11: Privacy, Compliance & Self-Hosted Deployment

### Purpose
Harden the platform for privacy-sensitive deployments: GDPR compliance features, recording consent workflows, data retention automation, and fully self-hosted Docker deployment with offline capabilities. This addresses the underserved legal, healthcare, and government market segments.

### Tasks

#### 11.1 — GDPR Compliance & Data Retention

**What**: Implement right-to-erasure (GDPR Article 17), data export (Article 20), consent management, and automated data retention enforcement.

**Design**:

```typescript
// API Routes
// POST /api/privacy/export-my-data          → GDPR data export (Article 20)
// POST /api/privacy/delete-my-data          → GDPR right to erasure (Article 17)
// GET  /api/privacy/consents                → list user's consents
// POST /api/privacy/consents/:id/revoke     → revoke specific consent

// Data export: generates a ZIP containing:
// - user-profile.json (user data)
// - meetings.json (all meetings user attended)
// - action-items.json (all action items assigned to user)
// - decisions.json (all decisions user was involved in)
// - transcripts/ (transcript segments where user spoke)
// - recordings/ (recordings user consented to, if downloadable)

// Right to erasure:
// 1. Anonymize user record (email → hash, display_name → "Deleted User")
// 2. Remove user from meeting_participants (keep meeting intact)
// 3. Anonymize transcript segments attributed to user
// 4. Reassign or delete action items
// 5. Remove from graph_nodes (person node)
// 6. Log deletion in audit_log (retain for 3 years per GDPR Article 17(3))
```

Data retention automation:

```typescript
// Scheduled job: runs nightly
export async function enforceDataRetention(organizationId: string): Promise<RetentionReport> {
  // 1. Load org's data_retention_policies from org settings JSONB
  // 2. For each entity type with a retention policy:
  //    - Find entities older than retention_days
  //    - Delete recordings from S3
  //    - Delete transcript text (retain metadata)
  //    - Mark meetings as archived
  // 3. Generate retention report
  // 4. Log actions in audit_log
}
```

**Testing**:
- `Unit: exportUserData → ZIP contains all expected files`
- `Unit: deleteUserData → user anonymized, references cleaned`
- `Unit: deleteUserData preserves meeting structure → meeting still exists without user details`
- `Integration: retention policy 90 days → recordings older than 90 days deleted`
- `Integration: retention enforcement → audit_log entries created for each deletion`
- `E2E: request data export → download link provided within 24 hours`

---

#### 11.2 — Recording Consent Workflow

**What**: Implement per-participant consent collection before and during recording, with org-level consent policies.

**Design**:

```typescript
// Consent flow:
// 1. Before recording starts, check org consent policy:
//    - 'org_policy': all members consented via org-level agreement
//    - 'pre_meeting': send consent request emails before meeting
//    - 'in_meeting': collect consent at recording start
// 2. For pre_meeting: send email to participants with consent link
// 3. For in_meeting: show consent modal when recording starts
// 4. Store consent in recording.consent_status JSONB
// 5. If any participant declines: recording proceeds but their segments are excluded

// API Routes
// POST /api/recordings/:id/consent          → submit consent response
// GET  /api/recordings/:id/consent-status   → check consent status

interface ConsentRequest {
  participantEmail: string;
  consented: boolean;
  method: 'in_meeting_prompt' | 'pre_meeting_email' | 'org_policy';
}
```

**Testing**:
- `Unit: org_policy consent mode → all org members auto-consented`
- `Unit: pre_meeting consent → emails sent to all external participants`
- `Unit: consent declined → consent_status.allConsented = false`
- `Integration: recording with mixed consent → allConsented reflects actual state`
- `E2E: consent modal appears → user accepts → recording starts`

---

#### 11.3 — Self-Hosted Docker Deployment

**What**: Production-ready Docker Compose configuration for fully self-hosted deployment with all dependencies, health checks, and configuration documentation.

**Design**:

```yaml
# docker-compose.production.yml
services:
  api:
    image: meeting-platform/api:latest
    build: { context: ., dockerfile: Dockerfile.api, target: production }
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://meeting_user:${DB_PASSWORD}@postgres:5432/meeting_platform
      REDIS_URL: redis://redis:6379
      S3_ENDPOINT: http://minio:9000
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/health"]
      interval: 30s
      retries: 3
    deploy:
      replicas: 2
      resources:
        limits: { cpus: '1.0', memory: 1G }

  worker:
    image: meeting-platform/worker:latest
    build: { context: ., dockerfile: Dockerfile.worker, target: production }
    environment:
      DATABASE_URL: postgresql://meeting_user:${DB_PASSWORD}@postgres:5432/meeting_platform
      REDIS_URL: redis://redis:6379
    deploy:
      replicas: 2

  web:
    image: meeting-platform/web:latest
    build: { context: ., dockerfile: Dockerfile.web, target: production }
    ports: ["443:3000"]

  postgres:
    image: pgvector/pgvector:pg16
    volumes: ["pgdata:/var/lib/postgresql/data"]
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}

  redis:
    image: redis:7-alpine
    volumes: ["redisdata:/data"]

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    volumes: ["miniodata:/data"]

  whisper:
    image: meeting-platform/whisper:latest
    deploy:
      resources:
        reservations: { devices: [{ capabilities: [gpu] }] }
```

Multi-stage Dockerfile for minimal production image:

```dockerfile
# Dockerfile.api
FROM node:22-alpine AS base
RUN corepack enable pnpm

FROM base AS deps
WORKDIR /app
COPY pnpm-lock.yaml package.json pnpm-workspace.yaml ./
COPY packages/shared/package.json packages/shared/
COPY apps/api/package.json apps/api/
RUN pnpm install --frozen-lockfile --prod

FROM base AS builder
WORKDIR /app
COPY . .
RUN pnpm install --frozen-lockfile
RUN pnpm turbo build --filter=api

FROM base AS production
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/apps/api/dist ./dist
CMD ["node", "dist/server.js"]
EXPOSE 3001
```

**Testing**:
- `Integration: docker compose build → all images build successfully`
- `Integration: docker compose up → all services healthy within 60 seconds`
- `Integration: API accessible at configured port → /health returns 200`
- `Integration: worker processes jobs → BullMQ dashboard shows active workers`
- `E2E: full self-hosted deployment → login, create meeting, upload recording, view transcript`

---

## Phase 12: MCP Server & API Ecosystem

### Purpose
Expose the platform as an MCP (Model Context Protocol) server so AI agents can query meeting data, and provide a public REST API with documentation for third-party integrations. This aligns with the 2026 trend of meeting data as a primary knowledge source for AI agents (Fireflies, Fellow, Otter, and Granola all have MCP servers).

### Tasks

#### 12.1 — MCP Server Implementation

**What**: Implement an MCP server that exposes meeting data tools for AI agents (Claude Desktop, ChatGPT, etc.).

**Design**:

```typescript
// MCP tools exposed:
// - meeting_search: Search across meeting history
// - meeting_get: Get full meeting details including transcript and action items
// - action_items_list: List action items with filters
// - decisions_list: List decisions with filters
// - transcript_search: Search within transcripts

import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';

const server = new McpServer({
  name: 'meeting-management-platform',
  version: '1.0.0',
});

server.tool(
  'meeting_search',
  'Search across meeting history by keyword, date range, or participant',
  {
    query: z.string().describe('Search query'),
    dateFrom: z.string().optional().describe('Start date (ISO 8601)'),
    dateTo: z.string().optional().describe('End date (ISO 8601)'),
    limit: z.number().optional().default(10).describe('Max results'),
  },
  async ({ query, dateFrom, dateTo, limit }) => {
    const results = await searchService.search({
      query, dateFrom, dateTo, limit,
      organizationId: currentOrgId,
    });
    return { content: [{ type: 'text', text: JSON.stringify(results) }] };
  }
);

server.tool(
  'meeting_get',
  'Get full details of a specific meeting including transcript, action items, and decisions',
  { meetingId: z.string().uuid() },
  async ({ meetingId }) => {
    const meeting = await meetingService.getFullMeeting(meetingId);
    return { content: [{ type: 'text', text: JSON.stringify(meeting) }] };
  }
);
```

**Testing**:
- `Unit: meeting_search tool → returns search results in correct MCP format`
- `Unit: meeting_get tool → returns full meeting details`
- `Unit: action_items_list with filters → filtered results`
- `Integration: MCP client connects → tool list returned`
- `Integration: MCP client calls meeting_search → results returned`
- `Integration: unauthorized MCP request → authentication error`

---

#### 12.2 — Public REST API & Documentation

**What**: Finalise the public API with rate limiting, API key authentication, OpenAPI 3.1 documentation, and developer portal.

**Design**:

```typescript
// API key management
// POST /api/admin/api-keys              → create API key
// GET  /api/admin/api-keys              → list API keys
// DELETE /api/admin/api-keys/:id        → revoke API key

// Rate limiting configuration
const RATE_LIMITS = {
  free: { windowMs: 60_000, max: 60 },       // 60 req/min
  pro: { windowMs: 60_000, max: 300 },        // 300 req/min
  business: { windowMs: 60_000, max: 1000 },  // 1000 req/min
  enterprise: { windowMs: 60_000, max: 5000 }, // 5000 req/min
};

// OpenAPI spec auto-generated by @fastify/swagger
// Served at /api/docs (Swagger UI) and /api/docs/openapi.json
```

Webhook system for event notifications:

```typescript
// Webhook events:
// meeting.created, meeting.updated, meeting.completed, meeting.cancelled
// transcript.completed
// action_item.created, action_item.completed
// decision.recorded
// recording.ready

// POST /api/admin/webhooks              → register webhook URL
// GET  /api/admin/webhooks              → list webhooks
// DELETE /api/admin/webhooks/:id        → remove webhook

interface WebhookRegistration {
  url: string;
  events: string[];          // event types to subscribe to
  secret: string;            // for HMAC signature verification
}
```

**Testing**:
- `Unit: API key authentication → valid key passes, invalid key → 401`
- `Unit: rate limiting → 61st request in free tier → 429`
- `Integration: OpenAPI spec generated → valid OpenAPI 3.1 document`
- `Integration: webhook delivery → POST to registered URL with HMAC signature`
- `Integration: webhook delivery failure → retried with exponential backoff`
- `E2E: developer portal loads → API documentation browsable and try-it-out works`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                     ─── required by everything
    │
Phase 2: Meeting CRUD & Calendar Sync   ─── requires Phase 1
    │
Phase 3: Recording & Transcription      ─── requires Phase 2
    │
Phase 4: AI Notes & Action Items        ─── requires Phase 3
    │
    ├── Phase 5: Agenda Builder         ─── requires Phase 4 (for AI suggestions)
    │       │                               can parallel with Phase 6
    │       │
    ├── Phase 6: Search & Chat          ─── requires Phase 4 (for indexed content)
    │       │                               can parallel with Phase 5
    │       │
    └── Phase 7: Analytics & Coaching   ─── requires Phase 3 (for transcript data)
            │                               can parallel with Phases 5, 6
            │
Phase 8: CRM & PM Integrations         ─── requires Phase 4 (for summaries & action items)
    │                                       can parallel with Phases 5, 6, 7
    │
Phase 9: Frontend Application          ─── requires Phases 1-7 (API endpoints)
    │                                       Phase 8 optional (integrations settings)
    │
Phase 10: Knowledge Graph              ─── requires Phases 4, 6 (decisions, search)
    │
Phase 11: Privacy & Self-Hosted        ─── requires Phases 1-9 (full platform)
    │                                       can parallel with Phase 10
    │
Phase 12: MCP Server & API             ─── requires Phases 1-6 (core API surface)
                                            can parallel with Phases 10, 11
```

### Parallelism Opportunities

- **Phases 5, 6, 7** can be developed concurrently after Phase 4 is complete
- **Phase 8** can be developed in parallel with Phases 5, 6, 7 (independent integration work)
- **Phases 10, 11, 12** can be developed concurrently after Phase 9 is complete
- **Phase 9** (frontend) can begin partial work after Phase 2, with stubs for later API endpoints

---

## Definition of Done (per phase)

1. All tasks implemented with production-quality code (no TODOs in critical paths).
2. All unit tests pass (`pnpm test`).
3. All integration tests pass with testcontainers (`pnpm test:integration`).
4. ESLint passes with zero warnings (`pnpm lint`).
5. Prettier formatting applied (`pnpm format:check`).
6. TypeScript strict mode compiles without errors (`pnpm typecheck`).
7. Docker images build successfully (`docker compose build`).
8. Feature works end-to-end in Docker Compose environment.
9. Database migrations are idempotent and reversible.
10. New API endpoints appear in auto-generated OpenAPI spec.
11. New configuration options documented in `.env.example`.
12. Audit log entries created for all data-modifying operations.
13. RLS policies enforced for all new tenant-scoped queries.
14. No secrets or credentials committed to source control.

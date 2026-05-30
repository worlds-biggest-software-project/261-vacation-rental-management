# Vacation Rental Management — Phased Development Plan

> Project: 261-vacation-rental-management · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` proposals into a single buildable specification for an AI-native, open-source, self-hostable short-term-rental (STR) property management system (PMS).

**Core value proposition:** Unify multi-channel listing/calendar/rate sync (Airbnb, Vrbo, Booking.com, iCal), guest communication, cleaning/maintenance coordination, and financial reporting on one self-hostable codebase — with AI as the operations layer (autonomous pricing, LLM guest messaging, predictive operations, compliance automation) rather than a bolt-on. It also publishes an **MCP server** so LLM agents can operate the PMS directly, matching the emerging hospitality-AI integration standard (Apaleo, 2025).

**Primary personas:** independent hosts (1–5 properties), professional property managers (10–200), STR investment funds (200+).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | **TypeScript (Node.js 22 LTS)** | The product is integration- and API-heavy (OTA REST/XML APIs, webhooks, OAuth, an MCP server, and a web dashboard). A single TS codebase shares types across backend, MCP server, and frontend. The MCP TypeScript SDK and the Anthropic/OpenAI SDKs are first-class. ML-heavy pieces (pricing models) are isolated behind a service boundary so a Python worker can be added later without a language rewrite of the core. |
| API framework | **Fastify 5 + `@fastify/swagger`** | High-throughput, schema-first. JSON Schema request/response validation is native and auto-generates an **OpenAPI 3.1** spec (standards.md requirement). Lower overhead than Express for the webhook/sync hot paths. |
| Validation / schema | **Zod + `zod-to-openapi`** | Single source of truth for runtime validation and OpenAPI/JSON-Schema generation. Validates JSONB payloads (data model #3) at the application layer, which is mandatory because JSONB has no DB-level type enforcement. |
| Database | **PostgreSQL 16** | Hybrid relational + JSONB model (data model #3) requires mature JSONB, GIN indexes, generated columns, `earthdistance`/PostGIS geo queries, and row-level security for multi-tenancy. Single-node Postgres also satisfies the self-hosted "zero-extra-infra" goal. |
| ORM / migrations | **Drizzle ORM + drizzle-kit** | TypeScript-native, thin over SQL (so the hand-tuned JSONB/GIN DDL from data model #3 survives intact), first-class JSONB column typing, and SQL-file migrations that are reviewable. |
| Cache / queue | **Redis 7 + BullMQ** | Channel sync, webhook processing, AI calls, scheduled messages, and tax/permit reminders are async and retryable. BullMQ gives delayed jobs (scheduled guest messages), repeatable jobs (nightly pricing, permit-expiry scans), and dead-letter handling. |
| Frontend | **Next.js 15 (App Router) + React 19 + Tailwind + shadcn/ui** | Operator dashboard: calendar grid, unified inbox, pricing view, ops board. Server Components for data-heavy lists; the same repo consumes the shared Zod types. |
| Auth | **OAuth 2.1 (Auth Code + PKCE) for users; Lucia/Auth.js session layer; JWT (RFC 7519) bearer tokens for API; per-org RLS** | standards.md mandates OAuth 2.0/2.1 + JWT. PKCE-on-all-flows per OAuth 2.1 draft. |
| LLM access | **Anthropic SDK (Claude) primary, provider-abstracted via a `LlmProvider` interface** | Guest messaging, compliance summarisation, message-derived maintenance detection. Abstraction lets operators self-host with OpenAI/local models. |
| AI agent surface | **Model Context Protocol (MCP) server via `@modelcontextprotocol/sdk`** | standards.md treats MCP publication as a first-class requirement; lets Claude/GPT agents operate the PMS. |
| Channel connectivity | **Adapter pattern**: official APIs where partner status exists (Airbnb, Vrbo Rapid, Booking.com), **iCal (RFC 5545)** universal fallback via `node-ical` + custom VEVENT writer | Channel API access is gated (research.md); iCal is the universal baseline every operator can use day one. |
| Payments | **Stripe** (Payment Intents, 3DS2/SCA, Connect for owner payouts) | PCI DSS scope reduction via tokenisation; native PSD2/SCA support (standards.md). No raw card data ever stored. |
| Messaging transport | **Twilio (SMS/WhatsApp) + SMTP/Resend (email)** | Matches incumbent guest-comms stack (standards.md). |
| Tax compliance | **Avalara MyLodgeTax adapter** behind a `TaxProvider` interface | US lodging tax automation; interface allows EU VAT providers later. |
| Testing | **Vitest** (unit/integration) + **Playwright** (E2E web) + **Testcontainers** (real Postgres/Redis in CI) | Fast TS-native runner; Testcontainers gives real DB integration tests without mocking SQL. |
| Code quality | **Biome** (lint+format) + **tsc** strict | Single fast tool for lint and format; strict type checking across the monorepo. |
| Monorepo | **pnpm workspaces + Turborepo** | Shared `packages/` (db, types, channel-adapters, ai) consumed by `apps/api`, `apps/web`, `apps/mcp`, `apps/worker`. |
| Containerisation | **Docker + docker-compose** | One-command self-host: api, worker, web, postgres, redis. |
| Observability | **Pino** (structured logs) + **OpenTelemetry** traces | Sync/webhook debugging across async boundaries. |

### Data Model Choice

The **Hybrid Relational + JSONB model (data-model-suggestion-3.md)** is adopted as the primary schema. Rationale: it is explicitly the pragmatic MVP choice for a startup that must support multi-channel and multi-jurisdiction heterogeneity from day one without a migration per city/channel. The **normalized 3NF model (data-model-suggestion-1.md)** informs the typed relational backbone (typed columns + FKs for `reservations`, `payments`, `calendar_days`, `guests`). The event-sourced model (#2) is deferred: a lightweight **`domain_events` outbox table** is included for audit + future ML training without committing to full CQRS. The graph model (#4) is out of scope for MVP.

### Project Structure

```
vacation-rental-management/
├── package.json                  # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── biome.json
├── tsconfig.base.json
├── docker-compose.yml            # postgres, redis, api, worker, web
├── Dockerfile                    # multi-stage; targets api | worker | web | mcp
├── .env.example
├── packages/
│   ├── types/                    # shared Zod schemas + inferred TS types (DTOs)
│   │   └── src/{property,reservation,guest,pricing,channel,...}.ts
│   ├── db/                       # Drizzle schema, migrations, RLS, seed
│   │   ├── src/schema/*.ts
│   │   ├── src/migrate.ts
│   │   ├── src/seed.ts
│   │   └── drizzle/              # generated SQL migrations
│   ├── core/                     # domain services (framework-agnostic business logic)
│   │   └── src/{properties,reservations,calendar,pricing,messaging,ops,finance,compliance}/
│   ├── channels/                 # channel adapter interface + impls
│   │   └── src/{adapter.ts,ical/,airbnb/,vrbo/,booking/}
│   ├── ai/                       # LlmProvider abstraction + prompt templates
│   │   └── src/{provider.ts,anthropic.ts,prompts/}
│   └── integrations/             # stripe, twilio, email, avalara adapters
├── apps/
│   ├── api/                      # Fastify HTTP API (REST + webhooks + OpenAPI)
│   │   └── src/{server.ts,routes/,plugins/,webhooks/}
│   ├── worker/                   # BullMQ processors (sync, ai, scheduled msgs, reminders)
│   │   └── src/{queues/,processors/,scheduler.ts}
│   ├── mcp/                      # MCP server exposing PMS tools
│   │   └── src/{server.ts,tools/}
│   └── web/                      # Next.js operator dashboard
│       └── src/app/...
└── tests/
    ├── fixtures/                 # sample iCal, OTA payloads, webhook bodies
    └── e2e/                      # Playwright specs
```

Group is by **concern** (db, core domain, channels, ai, integrations) so every phase adds modules without restructuring.

---

## Phase 1: Foundation — Monorepo, Database, Multi-Tenancy, Auth

### Purpose
Establish the buildable skeleton: pnpm/Turbo monorepo, Postgres schema with the hybrid model's identity tables, multi-tenant row-level security, the OpenAPI-emitting Fastify server, and OAuth 2.1/JWT authentication. After this phase the system can register an organisation, authenticate users, and enforce tenant isolation — the substrate every later phase depends on.

### Tasks

#### 1.1 — Monorepo & tooling scaffold

**What**: Create the pnpm workspace, Turborepo pipeline, Biome, strict tsconfig, Docker Compose, and `.env.example`.

**Design**:
- `pnpm-workspace.yaml` includes `packages/*` and `apps/*`.
- `turbo.json` tasks: `build`, `lint`, `typecheck`, `test`, `db:migrate`, `dev`.
- `tsconfig.base.json`: `"strict": true`, `"noUncheckedIndexedAccess": true`, `"module": "NodeNext"`, path aliases `@vrm/*` → `packages/*/src`.
- `docker-compose.yml` services: `postgres:16` (with `earthdistance`+`cube` extensions enabled via init SQL), `redis:7`, plus build targets for `api`, `worker`, `web`.
- `.env.example` keys: `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `ANTHROPIC_API_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `TWILIO_*`, `AVALARA_*`, `APP_BASE_URL`, `ENCRYPTION_MASTER_KEY`.

**Testing**:
- `Unit (smoke): pnpm install && turbo build` → all packages compile, exit 0.
- `Unit: tsc --noEmit` across workspace → zero type errors.
- `Integration: docker compose up postgres redis` → both containers healthy; psql connects; `SELECT earth_distance(...)` works (extension loaded).

#### 1.2 — Database package: identity, multi-tenancy, RLS

**What**: Drizzle schema + migration for `organisations`, `users`, and a `domain_events` outbox; enable Postgres row-level security keyed on `organisation_id`.

**Design**:
Tables (from data model #3, with JSONB `settings`/`preferences`):

```ts
// packages/db/src/schema/identity.ts
export const organisations = pgTable('organisations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  subscriptionTier: varchar('subscription_tier', { length: 50 }).notNull().default('free'),
  defaultCurrency: char('default_currency', { length: 3 }).notNull().default('USD'), // ISO 4217
  defaultTimezone: varchar('default_timezone', { length: 50 }).notNull().default('UTC'),
  settings: jsonb('settings').$type<OrgSettings>().notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id),
  email: varchar('email', { length: 255 }).notNull(),
  passwordHash: varchar('password_hash', { length: 255 }),
  fullName: varchar('full_name', { length: 255 }).notNull(),
  role: varchar('role', { length: 50 }).notNull().default('member'), // owner|admin|member|cleaner|viewer
  isActive: boolean('is_active').notNull().default(true),
  preferences: jsonb('preferences').$type<UserPrefs>().notNull().default({}),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [uniqueIndex('uq_users_org_email').on(t.organisationId, t.email)]);
```

```ts
// packages/db/src/schema/events.ts  — lightweight audit/outbox (deferred-CQRS hedge)
export const domainEvents = pgTable('domain_events', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull(),
  streamType: varchar('stream_type', { length: 50 }).notNull(),   // reservation|rate|cleaning|...
  streamId: uuid('stream_id').notNull(),
  eventType: varchar('event_type', { length: 100 }).notNull(),    // reservation.created, ...
  data: jsonb('data').notNull(),
  metadata: jsonb('metadata').notNull().default({}),              // actor_id, actor_type, correlation_id, source
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [index('idx_events_stream').on(t.streamId), index('idx_events_org_created').on(t.organisationId, t.createdAt)]);
```

RLS migration (raw SQL appended to migration file):
```sql
ALTER TABLE properties ENABLE ROW LEVEL SECURITY; -- (applied to every tenant table as added)
CREATE POLICY tenant_isolation ON properties
  USING (organisation_id = current_setting('app.current_org')::uuid);
```
- A Fastify/db helper `withTenant(orgId, fn)` runs `SET LOCAL app.current_org = $1` inside a transaction before any tenant query.
- `appendEvent(tx, {...})` helper writes to `domain_events`; called by domain services on every state change.

**Testing**:
- `Unit: OrgSettings/UserPrefs Zod parse` → valid object passes, unknown nested key stripped or flagged per schema.
- `Integration (Testcontainers): insert org + 2 users with same email different org` → succeeds; same email same org → unique violation.
- `Integration (RLS): set app.current_org=A, query properties` → returns only org-A rows; without the setting → zero rows (deny-by-default).
- `Integration: appendEvent within failing tx` → event rolled back (no orphan audit row).

#### 1.3 — Fastify server, OpenAPI, error model

**What**: Bootstrap `apps/api` with health check, Zod-validated routing, OpenAPI 3.1 generation, and a uniform error envelope.

**Design**:
- `GET /healthz` → `{ status: 'ok', db: 'up'|'down', redis: 'up'|'down' }`.
- `@fastify/swagger` serves `/openapi.json` (OAS 3.1) + Swagger UI at `/docs`.
- Error envelope: `{ error: { code: string, message: string, details?: unknown, requestId: string } }`; HTTP codes 400/401/403/404/409/422/429/500. A `RFC 8288` `Link` header is emitted on all paginated list endpoints.
- Pagination convention: `?limit=&cursor=`; responses include `nextCursor` and `Link: <...>; rel="next"`.

**Testing**:
- `Unit: error mapper` → ValidationError(zod) → 422 with field path in `details`.
- `Integration: GET /healthz with db down` → 503, `db: 'down'`.
- `Integration: GET /openapi.json` → valid OAS 3.1 (validate with `@apidevtools/swagger-parser`).

#### 1.4 — Authentication & RBAC

**What**: OAuth 2.1 password+PKCE login, JWT issuance, session middleware, role-based guards.

**Design**:
- `POST /auth/register` → creates org + owner user (Argon2id hash).
- `POST /auth/login` → returns access JWT (15 min) + refresh token (httpOnly cookie, 30 d). JWT claims: `sub`, `org`, `role`, `exp`, `iat` (RFC 7519).
- `POST /auth/refresh`, `POST /auth/logout`.
- `requireAuth` plugin verifies JWT, loads `{ userId, orgId, role }` onto `request.auth`, and calls `withTenant`.
- `requireRole(...roles)` guard; permission matrix: `cleaner` → cleaning tasks only; `viewer` → read-only; `owner/admin` → full.

**Testing**:
- `Unit: JWT verify expired token` → 401 `token_expired`.
- `Integration: login wrong password` → 401, no token; correct → 200 + valid JWT.
- `Integration: cleaner calls POST /properties` → 403.
- `Integration: refresh with revoked refresh token` → 401.

---

## Phase 2: Property & Listing Domain

### Purpose
Model the heart of the catalogue: physical properties (with JSONB `details`/`owners`/`photos`/`compliance`) and their per-channel listings. This is the entity every reservation, calendar day, rate, and task hangs off. After this phase an operator can fully CRUD their portfolio via API.

### Tasks

#### 2.1 — Properties schema & service

**What**: Implement the `properties` table (data model #3) and a `PropertyService`.

**Design**:
```ts
export const properties = pgTable('properties', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id),
  name: varchar('name', { length: 255 }).notNull(),
  propertyType: varchar('property_type', { length: 50 }).notNull(), // house|apartment|cabin|villa|condo
  status: varchar('status', { length: 50 }).notNull().default('active'), // active|inactive|onboarding|archived
  bedrooms: smallint('bedrooms').notNull().default(1),
  bathrooms: numeric('bathrooms', { precision: 3, scale: 1 }).notNull().default('1.0'),
  maxGuests: smallint('max_guests').notNull().default(2),
  checkInTime: time('check_in_time').notNull().default('15:00'),
  checkOutTime: time('check_out_time').notNull().default('11:00'),
  minStayNights: smallint('min_stay_nights').notNull().default(1),
  timezone: varchar('timezone', { length: 50 }).notNull().default('UTC'),
  addressLine1: varchar('address_line_1', { length: 255 }),
  city: varchar('city', { length: 100 }),
  stateProvince: varchar('state_province', { length: 100 }),
  postalCode: varchar('postal_code', { length: 20 }),
  countryCode: char('country_code', { length: 2 }), // ISO 3166-1
  latitude: numeric('latitude', { precision: 10, scale: 7 }),
  longitude: numeric('longitude', { precision: 10, scale: 7 }),
  details: jsonb('details').$type<PropertyDetails>().notNull().default({}),     // amenities, smart_home, rules
  owners: jsonb('owners').$type<PropertyOwner[]>().notNull().default([]),
  photos: jsonb('photos').$type<PropertyPhoto[]>().notNull().default([]),
  compliance: jsonb('compliance').$type<PropertyCompliance>().notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
// Indexes: idx_properties_org, idx_properties_status, idx_properties_country,
//          GIN(details), GIN(compliance), GiST geo index on (latitude,longitude)
```
- Zod schemas in `packages/types` validate `details`/`owners`/`photos`/`compliance` on write (mandatory JSONB validation). `PropertyDetails.amenities: z.array(AmenityEnum)`.
- `PropertyService.create/update/get/list/archive`; every mutation calls `appendEvent('property.created'|'property.updated'|'property.archived')`.

**Testing**:
- `Unit: PropertyDetails Zod` → amenities with unknown value → ValidationError naming the bad entry.
- `Integration: create property with pool, query details @> '{"amenities":["pool"]}'` → returns it.
- `Integration: list properties paginated` → `nextCursor` + `Link` header correct.
- `Integration: archive property` → status archived, `property.archived` event written.

#### 2.2 — Channels & listings

**What**: `channels` catalogue and `listings` (a property's presence on one channel).

**Design**:
```ts
export const channels = pgTable('channels', {
  id: uuid('id').primaryKey().defaultRandom(),
  code: varchar('code', { length: 50 }).notNull().unique(), // airbnb|vrbo|booking_com|direct|ical
  name: varchar('name', { length: 100 }).notNull(),
  apiType: varchar('api_type', { length: 50 }), // rest|xml|ical_only
  isActive: boolean('is_active').notNull().default(true),
  config: jsonb('config').notNull().default({}),
});

export const listings = pgTable('listings', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull(),
  propertyId: uuid('property_id').notNull().references(() => properties.id),
  channelId: uuid('channel_id').notNull().references(() => channels.id),
  externalId: varchar('external_id', { length: 255 }),
  listingUrl: text('listing_url'),
  status: varchar('status', { length: 50 }).notNull().default('draft'), // draft|active|paused|unlisted
  title: varchar('title', { length: 255 }).notNull(),
  description: text('description'),
  syncEnabled: boolean('sync_enabled').notNull().default(true),
  channelData: jsonb('channel_data').notNull().default({}), // channel-specific raw fields (round-trip fidelity)
  lastSyncedAt: timestamp('last_synced_at', { withTimezone: true }),
}, (t) => [uniqueIndex('uq_listing_property_channel').on(t.propertyId, t.channelId)]);
```
- Seed `channels` with airbnb, vrbo, booking_com, direct, ical.

**Testing**:
- `Integration: create two listings same property+channel` → unique violation.
- `Unit: ListingStatus enum` rejects invalid status.

#### 2.3 — Guidebooks (web/MVP-light)

**What**: JSONB-backed digital guidebook attached to a property (differentiator from Hostfully).

**Design**: `guidebooks` table: `id, property_id, title, is_published, share_token (unique), sections JSONB[] ({title, body, type, order, icon})`. Public read endpoint `GET /g/:shareToken` (no auth).

**Testing**:
- `Integration: published guidebook fetched by share token` → 200; unpublished → 404.

---

## Phase 3: Reservations, Guests & Calendar (Core Booking Engine)

### Purpose
The operational core: guests, reservations with typed financial breakdown, a per-night calendar with availability + pricing, and double-booking prevention. After this phase the system can hold the authoritative booking state that every channel sync reconciles against.

### Tasks

#### 3.1 — Guests with GDPR support

**What**: `guests` table with soft-delete + consent records.

**Design**:
```ts
export const guests = pgTable('guests', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull(),
  firstName: varchar('first_name', { length: 100 }).notNull(),
  lastName: varchar('last_name', { length: 100 }).notNull(),
  email: varchar('email', { length: 255 }),
  phone: varchar('phone', { length: 50 }),
  nationality: char('nationality', { length: 2 }),          // ISO 3166-1
  preferredLanguage: varchar('preferred_language', { length: 10 }), // ISO 639-1
  idVerified: boolean('id_verified').notNull().default(false),
  notes: text('notes'),
  isDeleted: boolean('is_deleted').notNull().default(false), // GDPR soft-delete
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
// consent_records: guest_id, consent_type, granted, ip_address, granted_at, revoked_at
```
- `GET /guests/:id/export` (GDPR portability) → JSON of all guest data + reservations.
- `DELETE /guests/:id` → soft-delete + scrub PII fields to `[erased]`, retain non-PII reservation financials.

**Testing**:
- `Integration: erasure request` → guest PII scrubbed, linked reservation totals intact.
- `Integration: export` → includes reservations + messages + consents.

#### 3.2 — Reservations with double-booking guard

**What**: `reservations` (typed financials from data model #1) + `reservation_nightly_rates`.

**Design**:
```ts
export const reservations = pgTable('reservations', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull(),
  propertyId: uuid('property_id').notNull().references(() => properties.id),
  listingId: uuid('listing_id').references(() => listings.id),
  guestId: uuid('guest_id').notNull().references(() => guests.id),
  channelId: uuid('channel_id').references(() => channels.id),
  externalId: varchar('external_id', { length: 255 }), // channel confirmation code
  status: varchar('status', { length: 50 }).notNull().default('pending'),
    // pending|confirmed|checked_in|checked_out|cancelled|no_show
  checkInDate: date('check_in_date').notNull(),
  checkOutDate: date('check_out_date').notNull(),
  guestsCount: smallint('guests_count').notNull().default(1),
  nightlyRate: numeric('nightly_rate', { precision: 12, scale: 2 }).notNull(),
  subtotal: numeric('subtotal', { precision: 12, scale: 2 }).notNull(),
  cleaningFee: numeric('cleaning_fee', { precision: 12, scale: 2 }).notNull().default('0'),
  serviceFee: numeric('service_fee', { precision: 12, scale: 2 }).notNull().default('0'),
  taxTotal: numeric('tax_total', { precision: 12, scale: 2 }).notNull().default('0'),
  totalAmount: numeric('total_amount', { precision: 12, scale: 2 }).notNull(),
  currencyCode: char('currency_code', { length: 3 }).notNull().default('USD'),
  payoutAmount: numeric('payout_amount', { precision: 12, scale: 2 }),
  cancellationPolicy: varchar('cancellation_policy', { length: 50 }),
  bookedAt: timestamp('booked_at', { withTimezone: true }).notNull().defaultNow(),
});
// CHECK (check_out_date > check_in_date); index (property_id, check_in_date, check_out_date)
```
**State machine**: `pending → confirmed → checked_in → checked_out`; `confirmed → cancelled`; `confirmed → no_show`. Illegal transitions → 409.

**Double-booking guard**: a Postgres exclusion constraint using `daterange` and `gist`:
```sql
ALTER TABLE reservations ADD CONSTRAINT no_overlap
  EXCLUDE USING gist (
    property_id WITH =,
    daterange(check_in_date, check_out_date, '[)') WITH &&
  ) WHERE (status NOT IN ('cancelled', 'no_show'));
```
- `createReservation` runs inside `withTenant` transaction; on overlap → 409 `dates_unavailable`. Writes `reservation.created` event + marks `calendar_days` booked.

**Testing**:
- `Integration: two overlapping confirmed reservations same property` → second → 409.
- `Integration: overlapping where first is cancelled` → second succeeds.
- `Unit: state machine confirmed→checked_out direct` → rejected; pending→checked_in → rejected.
- `Integration: create reservation` → calendar days marked booked, event written.

#### 3.3 — Calendar & availability

**What**: `calendar_days` (one row per property per date) + availability query + iCal export.

**Design**:
```ts
export const calendarDays = pgTable('calendar_days', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull(),
  propertyId: uuid('property_id').notNull(),
  date: date('date').notNull(),
  status: varchar('status', { length: 50 }).notNull().default('available'), // available|booked|blocked|maintenance
  reservationId: uuid('reservation_id').references(() => reservations.id),
  baseRate: numeric('base_rate', { precision: 12, scale: 2 }),
  adjustedRate: numeric('adjusted_rate', { precision: 12, scale: 2 }),
  minStay: smallint('min_stay'),
  currencyCode: char('currency_code', { length: 3 }).notNull().default('USD'),
}, (t) => [uniqueIndex('uq_calendar_property_date').on(t.propertyId, t.date)]);
```
- `GET /properties/:id/calendar?from=&to=` → day grid with status + effective rate.
- `GET /properties/:id/calendar.ics` → **RFC 5545**-compliant VEVENT export of booked/blocked ranges (busy export), no auth but signed token in path. Use a hand-written serializer (CRLF line endings, `DTSTART;VALUE=DATE`, `UID`, `DTSTAMP`).

**Testing**:
- `Unit: ical serializer` → output parses cleanly with `node-ical`; UIDs stable across re-export.
- `Integration: calendar range` → booked dates reflect reservation 3.2.
- `Fixture: golden .ics file` comparison for a known reservation set.

---

## Phase 4: Pricing & Rate Management

### Purpose
Add rule-based rate plans and a pricing-resolution engine that computes the effective nightly rate for any date, plus a recommendation store that later AI/3rd-party pricing engines write into. This delivers the "dynamic pricing and rate management" MVP feature and lays the seam for autonomous pricing (Phase 8).

### Tasks

#### 4.1 — Rate plans & resolution engine

**What**: `rate_plans` table + `resolveRate(propertyId, date)` function.

**Design**:
```ts
export const ratePlans = pgTable('rate_plans', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull(),
  propertyId: uuid('property_id').notNull(),
  name: varchar('name', { length: 100 }).notNull(),
  rateType: varchar('rate_type', { length: 50 }).notNull(), // base|weekend|seasonal|event|last_minute
  nightlyRate: numeric('nightly_rate', { precision: 12, scale: 2 }).notNull(),
  minStay: smallint('min_stay').notNull().default(1),
  startDate: date('start_date'),
  endDate: date('end_date'),
  daysOfWeek: smallint('days_of_week').array(), // 0..6, null = all
  priority: smallint('priority').notNull().default(0),
  isActive: boolean('is_active').notNull().default(true),
  currencyCode: char('currency_code', { length: 3 }).notNull().default('USD'),
});
```
Resolution algorithm (pure function, testable):
```
resolveRate(propertyId, date):
  candidates = active rate_plans where
      (startDate is null or date >= startDate) and (endDate is null or date <= endDate)
      and (daysOfWeek is null or dow(date) in daysOfWeek)
  if none: return property base rate (calendar_days.baseRate or details.base_rate)
  return candidate with max(priority) (ties → most specific date range)
```

**Testing**:
- `Unit: weekend plan (dow=[5,6], priority 10) vs base (priority 0)` → Friday returns weekend rate, Tuesday returns base.
- `Unit: overlapping seasonal + event, event higher priority` → event wins.
- `Unit: no plan` → falls back to base.

#### 4.2 — Pricing recommendations store

**What**: `pricing_recommendations` table + apply endpoint.

**Design**:
```ts
export const pricingRecommendations = pgTable('pricing_recommendations', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull(),
  propertyId: uuid('property_id').notNull(),
  date: date('date').notNull(),
  recommendedRate: numeric('recommended_rate', { precision: 12, scale: 2 }).notNull(),
  confidence: numeric('confidence', { precision: 5, scale: 4 }),
  factors: text('factors'),            // human-readable explanation
  source: varchar('source', { length: 50 }).notNull(), // internal_ml|pricelabs|beyond
  applied: boolean('applied').notNull().default(false),
  currencyCode: char('currency_code', { length: 3 }).notNull().default('USD'),
}, (t) => [uniqueIndex('uq_rec').on(t.propertyId, t.date, t.source)]);
```
- `POST /properties/:id/pricing/recommendations/apply` → copies recommended rate into `calendar_days.adjustedRate`, flags `applied`, writes `rate.recommendation_accepted` event.
- `GET /properties/:id/pricing` → merged view: base, plan, recommendation, effective.

**Testing**:
- `Integration: apply recommendation` → calendar adjustedRate updated; event written.
- `Integration: upsert same (property,date,source)` → updates not duplicates.

---

## Phase 5: Channel Integration (iCal first, adapters for OTAs)

### Purpose
Connect the internal booking state to the outside world. Build the channel-adapter abstraction and ship the universal **iCal (RFC 5545)** two-way sync first (works for every operator immediately), then the OAuth-gated OTA adapters behind the same interface, plus inbound webhook ingestion. This is where the product becomes a real channel manager.

### Tasks

#### 5.1 — Channel adapter interface & connection credentials

**What**: `ChannelAdapter` interface + encrypted `channel_connections` store.

**Design**:
```ts
export interface ChannelAdapter {
  readonly code: string; // airbnb|vrbo|booking_com|ical
  pullReservations(conn: ChannelConnection, since: Date): Promise<NormalizedReservation[]>;
  pushAvailability(conn: ChannelConnection, listing: Listing, days: CalendarDay[]): Promise<SyncResult>;
  pushRates(conn: ChannelConnection, listing: Listing, days: CalendarDay[]): Promise<SyncResult>;
  parseWebhook?(headers: Headers, body: Buffer): Promise<DomainEventInput[]>;
  verifyWebhook?(headers: Headers, body: Buffer, secret: string): boolean;
}
```
```ts
export const channelConnections = pgTable('channel_connections', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull(),
  channelId: uuid('channel_id').notNull(),
  authType: varchar('auth_type', { length: 50 }).notNull(), // oauth2|api_key|ical
  accessToken: text('access_token'),  // AES-256-GCM encrypted at rest with ENCRYPTION_MASTER_KEY
  refreshToken: text('refresh_token'),
  tokenExpiresAt: timestamp('token_expires_at', { withTimezone: true }),
  apiKey: text('api_key'),
  status: varchar('status', { length: 50 }).notNull().default('connected'),
  lastSyncAt: timestamp('last_sync_at', { withTimezone: true }),
}, (t) => [uniqueIndex('uq_conn').on(t.organisationId, t.channelId)]);
// channel_sync_log: direction, entity_type, status, request/response payload, error, duration_ms
```
- `encryptSecret/decryptSecret` helpers (Node `crypto`, AES-256-GCM, master key from env). Tokens never logged.

**Testing**:
- `Unit: encrypt → decrypt round-trip` → plaintext recovered; ciphertext differs each call (random IV).
- `Integration: store + read connection` → token decrypts correctly; never appears in sync_log payloads.

#### 5.2 — iCal two-way sync adapter

**What**: `IcalAdapter` — import external `.ics` URLs into blocks, export internal calendar (reuses 3.3 serializer).

**Design**:
- Import: fetch each `ical_url` configured on a listing, parse VEVENTs (`node-ical`), create `calendar_days` `status='blocked'` for external busy ranges (idempotent on UID).
- Export: already in 3.3.
- Scheduled BullMQ repeatable job every 15 min per connection.

**Testing**:
- `Fixture: import sample Airbnb-style .ics` → correct blocked ranges created.
- `Integration: re-import unchanged feed` → no duplicate blocks (idempotent by UID).
- `Integration: event removed from feed on re-import` → corresponding block cleared.

#### 5.3 — OTA adapters (Airbnb / Vrbo Rapid / Booking.com)

**What**: REST adapters implementing `ChannelAdapter`, with OAuth 2.0 (Airbnb), API-key+HMAC (Vrbo Rapid), OAuth/XML (Booking.com). Built against documented endpoints; live calls feature-flagged (require partner credentials).

**Design**:
- Per-adapter normaliser maps OTA payloads → `NormalizedReservation` (preserve raw payload in `channelData` for round-trip fidelity, per data model #3).
- Booking.com XML handled via `fast-xml-parser`; map OTA_HotelResNotif-style messages to domain events.
- Rate-limit + retry via BullMQ backoff; respect RFC 8288 pagination Link headers.

**Testing**:
- `Unit (mocked HTTP): Airbnb reservation payload → NormalizedReservation` correct mapping incl. nightly rates.
- `Unit (mocked): Vrbo HMAC signature` computed correctly.
- `Unit: Booking.com XML res notif → domain event`.
- `Integration (mocked): pull → upsert reservation` idempotent on externalId.

#### 5.4 — Inbound webhooks & sync orchestration

**What**: `POST /webhooks/:channel` endpoint + sync worker that reconciles inbound events into reservations/calendar.

**Design**:
- Signature verification per adapter (`verifyWebhook`); invalid → 401, nothing enqueued.
- Valid webhook → enqueue `channel-sync` job with raw body; processor calls `parseWebhook` → upserts reservations (idempotent on `channelId+externalId`) → updates calendar → writes events.
- Conflict policy: if inbound booking overlaps an existing confirmed reservation from another channel → flag `sync_conflict` (no silent double-book) and notify operator.

**Testing**:
- `Integration (mocked): valid signed webhook` → 200, job enqueued, reservation upserted.
- `Integration: invalid signature` → 401, no job, no DB write.
- `Integration: duplicate webhook (same externalId)` → single reservation (idempotent).
- `Integration: cross-channel overlap` → conflict flagged, operator notification queued.

---

## Phase 6: Guest Communication & Unified Inbox

### Purpose
Deliver the unified, multi-channel messaging that operators expect: conversations and messages across Airbnb in-app, SMS/WhatsApp (Twilio), and email, plus templated automated messages on booking-lifecycle triggers. Lays the foundation for LLM-personalised messaging in Phase 8.

### Tasks

#### 6.1 — Conversations, messages, templates

**What**: `conversations`, `messages`, `message_templates` tables + send/receive service.

**Design**:
```ts
// messages: conversation_id, sender_type(guest|host|system|ai), channel(email|sms|whatsapp|airbnb|in_app),
//           body, is_ai_generated, ai_model, external_id, read_at, created_at
// message_templates: trigger_event(booking_confirmed|pre_arrival|check_in|check_out|review_request),
//           delay_hours, body_template ({{guest_name}},{{property_name}},{{check_in_date}}), channel, language
```
- `renderTemplate(template, ctx)` interpolates `{{var}}` against a typed context; missing var → 422 (fail loud).
- `sendMessage` routes to the correct transport adapter (Twilio/email/OTA) based on `channel`; persists message; writes `message.sent` event.
- Inbound (Twilio/email/OTA webhook) → append `message.received`, mark conversation `open`, increment unread.

**Testing**:
- `Unit: renderTemplate` → all vars substituted; unknown var → error naming it.
- `Integration (mocked Twilio): send SMS` → Twilio called with correct to/body; message persisted.
- `Integration: inbound message` → conversation reopened, unread++.

#### 6.2 — Automated lifecycle messaging

**What**: BullMQ delayed jobs that fire templates on reservation events.

**Design**:
- On `reservation.confirmed` → schedule jobs at `bookedAt+delay` for `booking_confirmed`, and at `checkInDate - delay` for `pre_arrival`/`check_in`, `checkOutDate + delay` for `review_request`.
- Cancellation cancels pending scheduled jobs for that reservation.

**Testing**:
- `Integration: confirm reservation` → 4 delayed jobs enqueued with correct run-at.
- `Integration: cancel reservation` → pending message jobs removed.

---

## Phase 7: Operations — Cleaning, Maintenance, Finance

### Purpose
Coordinate the field operations and money. Cleaning turnovers auto-generated from checkouts, maintenance ticketing, payments via Stripe (PCI-safe), owner statements, and tax records. After this phase the platform covers the full operational + financial MVP scope.

### Tasks

#### 7.1 — Cleaning & maintenance

**What**: `cleaning_tasks` (+ JSONB checklist), `maintenance_requests`.

**Design**:
```ts
// cleaning_tasks: property_id, reservation_id, assigned_to(user), task_type(turnover|deep_clean|inspection),
//   status(pending|assigned|in_progress|completed|cancelled), scheduled_date/time, rating(1-5),
//   checklist JSONB ([{item, done, photo_url}])
// maintenance_requests: property_id, category, priority(low|medium|high|urgent),
//   status(open|assigned|in_progress|waiting_parts|completed|cancelled), title, description,
//   assigned_to, estimated_cost, actual_cost, scheduled_date
```
- On `reservation.checked_out` (or checkout date reached) → auto-create a `turnover` cleaning task scheduled for checkout day.
- `cleaner` role sees only tasks assigned to them.

**Testing**:
- `Integration: reservation checkout` → turnover task auto-created for that date.
- `Integration: cleaner lists tasks` → only own assignments (RLS + role).
- `Unit: maintenance state machine` rejects open→completed skip if business rule requires assignment first (configurable).

#### 7.2 — Payments (Stripe, PCI-safe)

**What**: `payments` table + Stripe Payment Intents with 3DS2/SCA; webhook reconciliation. No card data stored.

**Design**:
```ts
// payments: reservation_id, payment_type(booking|security_deposit|refund|payout|adjustment),
//   status(pending|processing|completed|failed|refunded), amount, currency_code,
//   gateway('stripe'), gateway_txn_id(PaymentIntent id), gateway_fee, paid_at
```
- `POST /reservations/:id/payments/intent` → creates Stripe PaymentIntent (`automatic_payment_methods`, SCA enforced), returns `client_secret` (tokenisation client-side → PCI SAQ-A scope).
- `POST /webhooks/stripe` → verify signature (`STRIPE_WEBHOOK_SECRET`) → update payment status, write `payment.completed|failed|refunded` event.

**Testing**:
- `Integration (mocked Stripe): create intent` → returns client_secret; payment row `pending`.
- `Integration: stripe webhook payment_intent.succeeded` → payment `completed`, event written.
- `Integration: invalid Stripe signature` → 400, no state change.
- `Assert: no PAN/CVV ever persisted` (schema review test — no card columns exist).

#### 7.3 — Owner statements & tax records

**What**: `owner_statements` (+ line items) and `tax_records`; financial report endpoints.

**Design**:
```ts
// owner_statements: owner ref(in property.owners JSONB), period_start/end, gross_revenue,
//   management_fee, expenses, net_payout, currency_code, status(draft|approved|paid), items JSONB
// tax_records: reservation_id, tax_type(occupancy_tax|sales_tax|vat), jurisdiction, tax_rate,
//   taxable_amount, tax_amount, remitted, remitted_at
```
- `generateStatement(orgId, ownerId, period)` aggregates reservations for that owner's properties, applies `commission_pct` from `property.owners` JSONB, deducts maintenance costs → net payout.
- `GET /reports/financial?from=&to=&propertyId=` → revenue, occupancy %, ADR, RevPAR.

**Testing**:
- `Unit: statement aggregation` → gross/fees/net arithmetic correct for fixture reservations.
- `Integration: financial report` → occupancy% = booked nights / available nights for range.
- `Unit: tax record creation on reservation` → tax_amount = taxable_amount * rate.

---

## Phase 8: AI-Native Layer

### Purpose
This is the differentiator and the project's reason to exist: AI as the operations layer. Add LLM-personalised guest messaging, autonomous/assisted dynamic pricing, message-derived maintenance detection, and compliance summarisation — all behind a provider-abstracted interface so operators can self-host any model.

### Tasks

#### 8.1 — LLM provider abstraction & prompt library

**What**: `LlmProvider` interface + Anthropic implementation + versioned prompt templates.

**Design**:
```ts
export interface LlmProvider {
  complete(req: { system: string; messages: LlmMessage[]; maxTokens?: number; json?: boolean }): Promise<LlmResult>;
}
```
- Anthropic impl uses Claude with prompt caching on the (large, static) system prompt; structured outputs via tool/JSON mode.
- Prompts stored as templates in `packages/ai/src/prompts/` with explicit version tags; logged with each AI message for auditability (`messages.ai_model`).

System prompt for guest messaging (structure):
```
You are the property host's assistant for {{property_name}} in {{city}}.
Reservation: {{check_in}}–{{check_out}}, {{guests}} guests. Guest language: {{language}}.
House rules: {{rules}}. Check-in instructions: {{check_in_info}}.
Write a {{message_type}} message. Tone: warm, concise, professional. Reply ONLY in {{language}}.
Never invent amenities or policies not listed. If guest asks something you cannot answer, say a human will follow up.
```

**Testing**:
- `Unit (mocked provider): generate pre-arrival message` → contains property name, respects language param.
- `Unit: provider returns malformed JSON in json-mode` → retried once then surfaced as error.
- `Integration: AI message persisted` with `is_ai_generated=true`, `ai_model` recorded.

#### 8.2 — AI guest messaging (draft + autosend modes)

**What**: AI-generated replies in the unified inbox, configurable per-org (`settings.features.ai_messaging`).

**Design**:
- On inbound guest message → enqueue `ai-reply` job → build context (reservation, property, guidebook, prior messages) → `LlmProvider.complete` → if org mode `autosend` and confidence high → send; else create draft for human approval.
- Guardrail: AI never sends payment links or cancels bookings; those require human action.

**Testing**:
- `Integration (mocked LLM): inbound question, autosend on` → reply sent, flagged AI.
- `Integration: draft mode` → draft created, not sent.
- `Unit: guardrail` → AI output containing a refund instruction → blocked, escalated to human.

#### 8.3 — AI/assisted dynamic pricing

**What**: A pricing recommendation generator that writes into `pricing_recommendations` (source `internal_ml`), plus adapters to pull PriceLabs/Beyond recommendations.

**Design**:
- Heuristic+LLM hybrid for MVP: features = base rate, lead time, day-of-week, occupancy of surrounding dates, configured event dates, season. Produces `recommendedRate` + `confidence` + human-readable `factors`.
- Nightly BullMQ repeatable job per property generates recommendations for the next 365 days.
- Autonomous mode (`settings.features.ai_pricing`): auto-apply recommendations above confidence threshold; otherwise surface for approval.
- `PricingProvider` adapter interface lets PriceLabs/Beyond write recommendations through the same store.

**Testing**:
- `Unit: recommendation engine` → high-demand date (event + low surrounding availability) → rate above base; explanation lists factors.
- `Integration: autonomous mode above threshold` → calendar adjustedRate updated automatically + event written.
- `Integration (mocked PriceLabs): pull recommendations` → stored with source=pricelabs.

#### 8.4 — Predictive ops & compliance AI

**What**: (a) Detect maintenance issues from guest messages; (b) summarise/track jurisdiction compliance from `property.compliance` JSONB.

**Design**:
- Maintenance detection: on inbound message, LLM classifies `{issue: bool, category, urgency}`; if issue → auto-create `maintenance_request` linked to property/reservation.
- Compliance: nightly scan of `property.compliance.permits[]` for expiries within `renewal_reminder_days` → notify; LLM summarises new jurisdiction rules into structured compliance JSONB (human-reviewed before save).

**Testing**:
- `Unit (mocked LLM): "the AC is broken" message` → maintenance_request created, category=hvac, urgency=high.
- `Unit: "great stay, thanks!"` → no maintenance request.
- `Integration: permit expiring in 20 days (reminder=30)` → reminder notification enqueued.

---

## Phase 9: MCP Server, Reviews, Hardening

### Purpose
Publish the AI-agent integration surface (MCP) that standards.md flags as a first-class requirement, complete the reviews/reputation feature, and harden the platform (rate limiting, audit, RBAC review, GDPR endpoints). After this phase the product is feature-complete for v1.0 + the AI-agent operating surface.

### Tasks

#### 9.1 — MCP server

**What**: `apps/mcp` exposing PMS operations as MCP tools for LLM agents (Claude, GPT-4o).

**Design**:
- Tools: `list_properties`, `get_calendar`, `create_reservation`, `block_dates`, `set_rate`, `list_reservations`, `send_guest_message`, `get_financial_report`, `list_open_tasks`.
- Auth: MCP server authenticates as a service principal scoped to one organisation via an API token (JWT); all calls go through the same `core` services + RLS (no privilege bypass).
- Tool schemas generated from the same Zod definitions in `packages/types`.
- Destructive tools (`create_reservation`, `set_rate`) require an `confirm` flag or operate in a sandboxed/approval mode by config.

**Testing**:
- `Integration: MCP list_properties` → returns only the authenticated org's properties.
- `Integration: MCP create_reservation overlapping dates` → returns structured error (not crash).
- `Unit: tool input schema` rejects malformed args with field-level message.

#### 9.2 — Reviews & reputation

**What**: `reviews` table (guest↔host, multi-criteria), AI-assisted review responses.

**Design**:
```ts
// reviews: reservation_id, property_id, guest_id, direction(guest_to_host|host_to_guest),
//   overall_rating, cleanliness, communication, check_in, accuracy, location, value,
//   body, response, is_public, published_at
```
- `POST /reviews/:id/ai-response` → LLM drafts a response using property/stay context.

**Testing**:
- `Integration: create review + AI response draft` → response non-empty, references stay.
- `Integration: aggregate property rating` → average across reviews correct.

#### 9.3 — Hardening: rate limiting, audit, GDPR, security

**What**: Cross-cutting production-readiness.

**Design**:
- `@fastify/rate-limit` (per-org + per-IP); 429 with `Retry-After`.
- Audit: ensure every mutating service writes a `domain_events` row; expose `GET /audit?entity=&id=`.
- GDPR endpoints finalised (export/erase from 3.1) + `consent_records` enforcement on marketing messages.
- Security headers (`@fastify/helmet`), secret rotation doc, dependency audit in CI.
- OWASP review: parameterised queries (Drizzle), RLS deny-by-default, encrypted tokens, no secrets in logs.

**Testing**:
- `Integration: exceed rate limit` → 429 + Retry-After.
- `Integration: marketing message to guest without consent` → blocked.
- `Security: SQL injection attempt in list filter` → safely parameterised, no leakage.
- `Integration: audit query` → returns ordered event history for an entity.

---

## Phase 10: Web Dashboard & Deployment

### Purpose
Ship the operator-facing UI and the one-command self-host/managed-cloud deployment. This is the surface most users touch and the final piece that makes the platform usable end-to-end without the API directly.

### Tasks

#### 10.1 — Operator dashboard (Next.js)

**What**: Multi-property dashboard, calendar grid, unified inbox, pricing view, ops board, reports.

**Design**:
- App Router; server components fetch via the API using the shared Zod types.
- Screens: Login/Org setup; Portfolio (property list/detail with JSONB-backed amenities/compliance editors); Multi-calendar grid (drag to block, inline rate edit); Unified Inbox (conversations, AI-draft approve/send); Pricing (base/plan/recommendation overlay, accept recs); Operations board (cleaning/maintenance kanban); Reports (occupancy, ADR, RevPAR, owner statements); Settings (channels connect, feature flags, team/RBAC).
- shadcn/ui components + Tailwind; optimistic updates on calendar edits.

**Testing**:
- `E2E (Playwright): register → create property → block calendar dates` → reflected in calendar API.
- `E2E: receive guest message → approve AI draft → sends` → message persisted.
- `E2E: connect iCal feed → import → external block appears on grid`.
- `E2E: cleaner login` → sees only assigned tasks, no pricing/finance nav.

#### 10.2 — Deployment & docs

**What**: Production Docker images, compose for self-host, env validation, seed/demo data, README/runbook.

**Design**:
- Multi-stage `Dockerfile` with `api`/`worker`/`web`/`mcp` build targets.
- `docker compose up` brings up the full stack; `pnpm db:migrate && pnpm db:seed` initialises schema + demo org.
- Env validation at boot (Zod) → fail fast with a clear message listing missing keys.
- Healthchecks wired into compose; OpenTelemetry/Pino logging on by default.
- Managed-cloud path documented (same images, external Postgres/Redis URLs).

**Testing**:
- `E2E (CI): docker compose up + migrate + seed + healthz` → all services healthy.
- `Integration: boot with missing DATABASE_URL` → process exits non-zero with explicit message.
- `Smoke: web container serves dashboard, hits api /healthz`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, db, RLS, auth, OpenAPI)   ─── required by everything
    │
Phase 2: Property & Listing domain                       ─── requires 1
    │
Phase 3: Reservations, Guests, Calendar (booking engine) ─── requires 2
    │
    ├── Phase 4: Pricing & Rate management               ─── requires 3
    ├── Phase 5: Channel integration (iCal + OTA + webhooks) ─ requires 3 (4 helps for rate push)
    └── Phase 6: Guest communication & inbox             ─── requires 3
              │
    ┌─────────┴───────────────┐
Phase 7: Operations (cleaning, maintenance, finance)     ─── requires 3 (5 for channel payouts)
    │
Phase 8: AI-native layer (msging, pricing, ops, compliance) ─ requires 4, 6, 7
    │
Phase 9: MCP server, reviews, hardening                  ─── requires 2–8
    │
Phase 10: Web dashboard & deployment                     ─── requires 1–9 (incremental UI per phase ok)
```

**Parallelism opportunities:**
- After Phase 3, **Phases 4, 5, and 6 can be developed concurrently** (pricing, channel sync, and messaging are independent given the booking engine).
- Within Phase 8, sub-tasks 8.2 (messaging AI), 8.3 (pricing AI), and 8.4 (ops/compliance AI) are independent once 8.1 (provider abstraction) lands.
- The **web dashboard (Phase 10.1) can be built incrementally** alongside Phases 2–9 (one screen per backend domain) rather than all at the end; it is listed last only for definition-of-done sequencing.
- OTA adapters (5.3) requiring partner credentials can be stubbed/feature-flagged and finished asynchronously without blocking the iCal-based MVP (5.2).

---

## Definition of Done (per phase)

Every phase is complete only when **all** of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`turbo test`).
3. Biome lint + format pass with zero errors (`turbo lint`).
4. `tsc --noEmit` passes across the workspace (strict mode, no `any` leaks).
5. New/changed Drizzle migrations are generated, committed, and apply cleanly on a fresh DB.
6. New API endpoints appear in the auto-generated **OpenAPI 3.1** spec and validate against it.
7. New JSONB columns have a corresponding Zod schema in `packages/types` and are validated on write.
8. Every mutating operation writes a `domain_events` audit row.
9. Multi-tenant isolation is enforced (RLS deny-by-default verified by an integration test).
10. Secrets are never logged; any new external credentials are encrypted at rest.
11. Docker build succeeds for affected targets; `docker compose up` remains healthy.
12. The phase's feature works end-to-end (verified by at least one integration or E2E test).
13. New config/env keys are added to `.env.example` and validated at boot.

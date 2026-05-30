# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Vacation Rental Management · Created: 2026-05-22

## Philosophy

This model treats every state change as an immutable event appended to an event store. The current state of any entity — a reservation, a property, a rate — is derived by replaying the sequence of events that produced it. The write side (command model) validates business rules and appends events; the read side (query model) maintains denormalized projections optimised for specific query patterns. This is the Command Query Responsibility Segregation (CQRS) pattern combined with Event Sourcing.

Event sourcing is the natural fit for a vacation rental management platform because the domain is fundamentally event-driven: bookings are created, modified, and cancelled; rates change daily; guests message back and forth; cleanings are scheduled, started, and completed; permits expire and are renewed. Every one of these state transitions is a business event that stakeholders may need to audit, replay, or analyze. When an owner disputes a payout, the system can reconstruct the exact sequence of rate changes, booking modifications, and fee calculations that produced the final number.

This architecture also unlocks AI capabilities that a traditional CRUD schema cannot: ML models can train directly on event streams to detect booking fraud patterns, predict cancellations, identify pricing anomalies, and generate insights like "properties that receive a rate increase within 48 hours of a competitor listing update achieve 12% higher occupancy." The event store becomes the training dataset.

**Best for:** Organisations that need bulletproof audit trails, temporal queries ("what was the state on date X?"), AI/ML training on operational event streams, and complex event-driven automation workflows.

**Trade-offs:**
- Pro: Complete, immutable audit history — every change is recorded with timestamp, actor, and causation
- Pro: Temporal queries are trivial — replay events up to any point in time
- Pro: Event streams are ideal training data for ML pricing, fraud detection, and operational analytics
- Pro: Easy to add new read projections without changing the write model
- Pro: Natural fit for webhook-driven channel integrations (Airbnb, Booking.com events map directly to domain events)
- Con: Higher storage requirements — events accumulate indefinitely
- Con: Read model staleness — projections are eventually consistent, not immediately consistent
- Con: Increased complexity — developers must understand event replay, projection rebuilds, and idempotency
- Con: Schema evolution is harder — changing event schemas requires versioning and upcasting
- Con: Debugging requires understanding the event chain, not just inspecting current state

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 3166-1 alpha-2 | Country codes in address and jurisdiction events |
| ISO 3166-2 | Subdivision codes in jurisdiction-related events |
| ISO 4217 | Currency codes embedded in all monetary event payloads |
| RFC 5545 (iCalendar) | Calendar projections generate iCal-compliant output from availability events |
| OpenTravel Alliance (OTA) | Inbound OTA messages (OTA_HotelResNotif) are translated into domain events |
| CloudEvents v1.0 | Event envelope format follows CloudEvents specification for interoperability |
| OCSF (Open Cybersecurity Schema Framework) | Security-relevant events (login, permission change, data export) follow OCSF categories |
| GDPR (EU 2016/679) | Crypto-shredding pattern: guest PII is encrypted with per-guest keys; key deletion satisfies right-to-erasure without mutating the event store |

---

## Event Store — The Source of Truth

```sql
-- The single append-only event store
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,              -- aggregate root ID (property_id, reservation_id, etc.)
    stream_type     VARCHAR(50) NOT NULL,        -- property, reservation, guest, rate_plan, cleaning_task, etc.
    event_type      VARCHAR(100) NOT NULL,        -- e.g., 'reservation.created', 'rate.adjusted', 'cleaning.completed'
    event_version   INTEGER NOT NULL,             -- sequential within stream for ordering
    data            JSONB NOT NULL,               -- event payload
    metadata        JSONB NOT NULL DEFAULT '{}',  -- actor, causation_id, correlation_id, ip_address
    organisation_id UUID NOT NULL,                -- tenant partition key
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
);

-- Partition by organisation for multi-tenant isolation
CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(event_type, created_at);
CREATE INDEX idx_events_org ON events(organisation_id, created_at);
CREATE INDEX idx_events_created ON events(created_at);

-- Example: event metadata follows CloudEvents-inspired envelope
-- {
--   "actor_id": "user-uuid",
--   "actor_type": "user",           -- user, system, ai_agent, channel_webhook
--   "correlation_id": "corr-uuid",  -- links related events across aggregates
--   "causation_id": "event-uuid",   -- the event that caused this event
--   "ip_address": "192.168.1.1",
--   "user_agent": "Mozilla/5.0...",
--   "source": "airbnb_webhook"      -- origin system
-- }
```

### Event Type Taxonomy

```sql
-- Registry of all known event types (for documentation and schema validation)
CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    schema_version  INTEGER NOT NULL DEFAULT 1,
    payload_schema  JSONB,                    -- JSON Schema for validating event data
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types:
-- Property lifecycle:     property.created, property.updated, property.archived
-- Listing lifecycle:      listing.created, listing.activated, listing.paused, listing.synced
-- Reservation lifecycle:  reservation.created, reservation.confirmed, reservation.modified,
--                         reservation.checked_in, reservation.checked_out, reservation.cancelled
-- Rate events:            rate.base_set, rate.adjustment_applied, rate.recommendation_received,
--                         rate.recommendation_accepted, rate.recommendation_rejected
-- Guest events:           guest.created, guest.updated, guest.merged, guest.data_exported,
--                         guest.data_erased
-- Communication events:   message.received, message.sent, message.ai_generated, message.read
-- Cleaning events:        cleaning.scheduled, cleaning.assigned, cleaning.started,
--                         cleaning.completed, cleaning.rated
-- Maintenance events:     maintenance.reported, maintenance.assigned, maintenance.completed
-- Payment events:         payment.initiated, payment.completed, payment.failed, payment.refunded
-- Compliance events:      permit.issued, permit.renewed, permit.expired, tax.calculated, tax.remitted
-- Channel events:         channel.connected, channel.disconnected, channel.sync_started,
--                         channel.sync_completed, channel.sync_failed
```

---

## Read Projections (Query Side)

Projections are materialised views rebuilt from events. Each projection is optimised for a specific read pattern.

### Projection: Current Property State

```sql
-- Materialised view of current property state (rebuilt from property.* events)
CREATE TABLE projection_properties (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    property_type   VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    bedrooms        SMALLINT NOT NULL,
    bathrooms       NUMERIC(3,1) NOT NULL,
    max_guests      SMALLINT NOT NULL,
    area_sqft       INTEGER,
    description     TEXT,
    check_in_time   TIME,
    check_out_time  TIME,
    min_stay_nights SMALLINT,
    timezone        VARCHAR(50),
    address_line_1  VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2),
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    amenities       JSONB DEFAULT '[]',     -- denormalised array of amenity names
    photo_urls      JSONB DEFAULT '[]',     -- denormalised array of photo objects
    owner_names     JSONB DEFAULT '[]',     -- denormalised array of owner info
    listing_count   INTEGER DEFAULT 0,
    active_listings JSONB DEFAULT '[]',     -- [{channel, external_id, status, url}]
    last_event_id   UUID,                   -- for projection tracking
    last_event_at   TIMESTAMPTZ,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_properties_org ON projection_properties(organisation_id);
CREATE INDEX idx_proj_properties_status ON projection_properties(status);
```

### Projection: Current Reservation State

```sql
-- Materialised view of current reservation state
CREATE TABLE projection_reservations (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    property_id     UUID NOT NULL,
    property_name   VARCHAR(255),           -- denormalised for display
    guest_id        UUID NOT NULL,
    guest_name      VARCHAR(255),           -- denormalised
    guest_email     VARCHAR(255),
    channel_code    VARCHAR(50),
    external_id     VARCHAR(255),
    status          VARCHAR(50) NOT NULL,
    check_in_date   DATE NOT NULL,
    check_out_date  DATE NOT NULL,
    nights          SMALLINT,
    guests_count    SMALLINT,
    adults          SMALLINT,
    children        SMALLINT,
    nightly_rate    NUMERIC(12,2),
    subtotal        NUMERIC(12,2),
    cleaning_fee    NUMERIC(12,2),
    service_fee     NUMERIC(12,2),
    tax_total       NUMERIC(12,2),
    total_amount    NUMERIC(12,2),
    currency_code   CHAR(3),
    payout_amount   NUMERIC(12,2),
    special_requests TEXT,
    cancellation_policy VARCHAR(50),
    payment_status  VARCHAR(50),
    cleaning_status VARCHAR(50),
    nightly_rates   JSONB DEFAULT '[]',     -- [{date, rate}]
    status_history  JSONB DEFAULT '[]',     -- [{status, changed_at, changed_by}]
    last_event_id   UUID,
    last_event_at   TIMESTAMPTZ,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_res_org ON projection_reservations(organisation_id);
CREATE INDEX idx_proj_res_property ON projection_reservations(property_id);
CREATE INDEX idx_proj_res_guest ON projection_reservations(guest_id);
CREATE INDEX idx_proj_res_dates ON projection_reservations(property_id, check_in_date, check_out_date);
CREATE INDEX idx_proj_res_status ON projection_reservations(status);
CREATE INDEX idx_proj_res_checkin ON projection_reservations(check_in_date);
```

### Projection: Calendar / Availability

```sql
-- Materialised calendar view (rebuilt from reservation.*, rate.*, and availability.* events)
CREATE TABLE projection_calendar (
    property_id     UUID NOT NULL,
    date            DATE NOT NULL,
    status          VARCHAR(50) NOT NULL,   -- available, booked, blocked, maintenance
    reservation_id  UUID,
    guest_name      VARCHAR(255),           -- denormalised
    base_rate       NUMERIC(12,2),
    adjusted_rate   NUMERIC(12,2),
    min_stay        SMALLINT,
    currency_code   CHAR(3),
    note            VARCHAR(255),
    last_event_id   UUID,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (property_id, date)
);
CREATE INDEX idx_proj_cal_status ON projection_calendar(status);
CREATE INDEX idx_proj_cal_date ON projection_calendar(date);
```

### Projection: Guest Profile

```sql
CREATE TABLE projection_guests (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    email           VARCHAR(255),
    phone           VARCHAR(50),
    nationality     CHAR(2),
    preferred_language VARCHAR(10),
    id_verified     BOOLEAN DEFAULT false,
    total_stays     INTEGER DEFAULT 0,
    total_revenue   NUMERIC(12,2) DEFAULT 0,
    average_rating  NUMERIC(3,1),
    first_stay_date DATE,
    last_stay_date  DATE,
    is_repeat_guest BOOLEAN DEFAULT false,
    tags            JSONB DEFAULT '[]',
    notes           TEXT,
    is_erased       BOOLEAN DEFAULT false,  -- GDPR: crypto-shredded
    last_event_id   UUID,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_guests_org ON projection_guests(organisation_id);
CREATE INDEX idx_proj_guests_email ON projection_guests(email);
```

### Projection: Financial Summary

```sql
CREATE TABLE projection_financial_summary (
    organisation_id UUID NOT NULL,
    property_id     UUID NOT NULL,
    month           DATE NOT NULL,          -- first day of month
    total_bookings  INTEGER DEFAULT 0,
    total_nights    INTEGER DEFAULT 0,
    occupancy_rate  NUMERIC(5,2) DEFAULT 0,
    gross_revenue   NUMERIC(12,2) DEFAULT 0,
    cleaning_fees   NUMERIC(12,2) DEFAULT 0,
    service_fees    NUMERIC(12,2) DEFAULT 0,
    tax_collected   NUMERIC(12,2) DEFAULT 0,
    refunds         NUMERIC(12,2) DEFAULT 0,
    net_revenue     NUMERIC(12,2) DEFAULT 0,
    management_fees NUMERIC(12,2) DEFAULT 0,
    owner_payout    NUMERIC(12,2) DEFAULT 0,
    currency_code   CHAR(3) DEFAULT 'USD',
    last_event_id   UUID,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organisation_id, property_id, month)
);
```

### Projection: Operations Dashboard

```sql
CREATE TABLE projection_operations (
    organisation_id UUID NOT NULL,
    property_id     UUID NOT NULL,
    date            DATE NOT NULL,
    -- Cleaning status
    cleaning_task_id UUID,
    cleaning_status VARCHAR(50),
    cleaner_name    VARCHAR(255),
    cleaning_scheduled_time TIME,
    -- Maintenance
    open_maintenance_count INTEGER DEFAULT 0,
    urgent_maintenance     BOOLEAN DEFAULT false,
    -- Guest communication
    unread_messages INTEGER DEFAULT 0,
    last_message_at TIMESTAMPTZ,
    -- Check-in/out
    checking_in_today BOOLEAN DEFAULT false,
    checking_out_today BOOLEAN DEFAULT false,
    guest_name      VARCHAR(255),
    -- Compliance
    permit_expiring_soon BOOLEAN DEFAULT false,
    permit_expiry_date DATE,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organisation_id, property_id, date)
);
```

---

## Projection Tracking

```sql
-- Tracks the last event processed by each projection (for replay and catch-up)
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    last_rebuilt_at TIMESTAMPTZ,
    status          VARCHAR(50) NOT NULL DEFAULT 'running', -- running, paused, rebuilding, error
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Snapshots (Performance Optimisation)

```sql
-- Periodic snapshots of aggregate state to avoid full replay on every read
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    event_version   INTEGER NOT NULL,         -- version at which snapshot was taken
    state           JSONB NOT NULL,           -- full aggregate state at this point
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, event_version)
);
CREATE INDEX idx_snapshots_stream ON snapshots(stream_id);

-- Snapshot policy: take a snapshot every 100 events per stream
-- When loading an aggregate:
--   1. Load the latest snapshot for the stream
--   2. Replay only events after the snapshot's event_version
--   3. This limits replay to at most 100 events
```

---

## Command Handlers (Write Side Reference)

The write side does NOT have its own tables — it reads from the event store and snapshots to hydrate aggregates, validates commands against business rules, and appends new events.

```sql
-- Example: Creating a reservation command flow (pseudocode in SQL comments)

-- 1. Load the property calendar projection to check availability
-- SELECT * FROM projection_calendar
-- WHERE property_id = $1 AND date BETWEEN $check_in AND $check_out
-- AND status = 'available';

-- 2. If available, append a reservation.created event
-- INSERT INTO events (stream_id, stream_type, event_type, event_version, data, metadata, organisation_id)
-- VALUES (
--   gen_random_uuid(),                         -- new reservation stream
--   'reservation',
--   'reservation.created',
--   1,
--   '{
--     "property_id": "...",
--     "guest_id": "...",
--     "channel": "airbnb",
--     "external_id": "HMAK123456",
--     "check_in_date": "2026-07-15",
--     "check_out_date": "2026-07-20",
--     "guests_count": 4,
--     "adults": 2,
--     "children": 2,
--     "nightly_rates": [
--       {"date": "2026-07-15", "rate": 250.00},
--       {"date": "2026-07-16", "rate": 250.00},
--       {"date": "2026-07-17", "rate": 275.00},
--       {"date": "2026-07-18", "rate": 275.00},
--       {"date": "2026-07-19", "rate": 300.00}
--     ],
--     "cleaning_fee": 150.00,
--     "tax_total": 162.50,
--     "total_amount": 1662.50,
--     "currency_code": "USD",
--     "cancellation_policy": "moderate"
--   }',
--   '{"actor_id": "...", "actor_type": "channel_webhook", "source": "airbnb", "correlation_id": "..."}',
--   'org-uuid'
-- );

-- 3. Event processors asynchronously update projections:
--    - projection_reservations (insert new row)
--    - projection_calendar (mark dates as booked)
--    - projection_guests (increment total_stays)
--    - projection_financial_summary (update monthly totals)
--    - projection_operations (set checking_in_today flag)
```

---

## Temporal Queries

```sql
-- "What was the rate for property X on March 15, 2026?"
-- Replay rate events up to that date:

SELECT data->>'adjusted_rate' AS rate,
       data->>'base_rate' AS base_rate,
       created_at AS effective_at
FROM events
WHERE stream_id = $property_id
  AND stream_type = 'rate_plan'
  AND event_type IN ('rate.base_set', 'rate.adjustment_applied')
  AND created_at <= '2026-03-15 23:59:59+00'
  AND (data->>'date')::date = '2026-03-15'
ORDER BY event_version DESC
LIMIT 1;

-- "Show me all modifications to reservation X"
SELECT event_type,
       data,
       metadata->>'actor_id' AS changed_by,
       metadata->>'actor_type' AS actor_type,
       created_at
FROM events
WHERE stream_id = $reservation_id
  AND stream_type = 'reservation'
ORDER BY event_version ASC;
```

---

## GDPR Compliance: Crypto-Shredding

```sql
-- Per-guest encryption keys (for crypto-shredding pattern)
CREATE TABLE guest_encryption_keys (
    guest_id        UUID PRIMARY KEY,
    encryption_key  BYTEA NOT NULL,         -- AES-256 key, itself encrypted with a master key
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- When a guest exercises right-to-erasure:
-- 1. DELETE FROM guest_encryption_keys WHERE guest_id = $1;
-- 2. All events containing that guest's PII become unreadable (encrypted fields cannot be decrypted)
-- 3. Projections are rebuilt — guest fields show "[erased]"
-- 4. The event store remains immutable — no events are deleted
-- 5. Non-PII event data (booking dates, amounts, property_id) remains intact for financial reporting
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | events, event_type_registry, snapshots |
| Projection: Properties | 1 | projection_properties |
| Projection: Reservations | 1 | projection_reservations |
| Projection: Calendar | 1 | projection_calendar |
| Projection: Guests | 1 | projection_guests |
| Projection: Financials | 1 | projection_financial_summary |
| Projection: Operations | 1 | projection_operations |
| Infrastructure | 1 | projection_checkpoints |
| GDPR | 1 | guest_encryption_keys |
| **Total** | **11** | Plus additional projections as needed |

---

## Key Design Decisions

1. **Single event store table** — all domain events (reservations, rates, cleaning, maintenance, payments, compliance) flow into one `events` table. This simplifies infrastructure (one write path, one replication stream) and enables cross-aggregate correlation queries. The `stream_type` column allows filtering to specific aggregate types.

2. **CloudEvents-inspired metadata** — every event carries structured metadata including `actor_id`, `actor_type`, `correlation_id`, and `causation_id`. This enables full traceability: given any system state, you can trace exactly who did what, when, and why — critical for owner disputes and regulatory audits.

3. **Projections are disposable** — read-side projections can be dropped and rebuilt from the event store at any time. This means new reporting requirements (e.g., "we need a competitor pricing comparison dashboard") can be added by creating a new projection without touching the write model.

4. **Snapshot-based aggregate loading** — to avoid replaying thousands of events for long-lived aggregates (a property with years of rate changes), snapshots are taken every 100 events. Aggregate loading reads the latest snapshot plus subsequent events, keeping latency bounded.

5. **Event type registry with JSON Schema** — the `event_type_registry` table documents every event type with a JSON Schema for payload validation. This serves as living documentation and enables automated event validation in the write pipeline.

6. **Crypto-shredding for GDPR** — rather than deleting events (which would violate the immutability guarantee), guest PII is encrypted with per-guest keys. Deleting the key makes the PII unrecoverable while preserving the event store's integrity and non-PII data for financial reporting.

7. **Denormalized projections** — projections like `projection_reservations` embed guest name, property name, and nightly rate breakdowns directly. This eliminates joins on the read side, making API responses fast without the complexity of multi-table queries.

8. **Natural mapping to channel webhooks** — Airbnb, Vrbo, and Booking.com all deliver state changes via webhooks (new booking, modification, cancellation). These map directly to domain events (`reservation.created`, `reservation.modified`, `reservation.cancelled`), making the inbound integration layer a thin translator rather than a complex state-management layer.

9. **ML-ready event streams** — the event store doubles as a training dataset for AI models. Pricing events can train dynamic pricing models; guest communication events can train response quality models; cleaning events can train scheduling optimizers — all without building separate data pipelines.

10. **Eventually consistent reads** — projections are updated asynchronously after events are appended. For most vacation rental operations (calendar views, guest lists, financial reports), millisecond-level staleness is acceptable. For availability checks during booking, the command handler reads directly from the event store to ensure consistency.

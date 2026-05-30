# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Vacation Rental Management · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form (3NF) relational design, giving every domain concept its own table with explicit foreign key relationships. Each entity — property, listing, reservation, guest, owner, channel, rate rule, cleaning task, compliance permit — lives in a dedicated table with well-typed columns and referential integrity enforced at the database level. The schema is designed to be read by any SQL-literate developer without needing to understand JSON structures or event replay mechanics.

This approach mirrors how mature hospitality platforms like Booking.com's Connectivity API and the OpenTravel Alliance (OTA) message standards model their data: discrete, well-defined objects (Hotel, Room, Reservation, Guest, Rate Plan) connected by explicit identifiers. It is the most natural fit for a domain where data integrity is paramount — double-bookings, incorrect rates, or lost guest data are business-critical failures.

The normalized design excels when the team needs complex cross-entity queries (e.g., "show me all reservations for properties owned by X where the guest has stayed before and the cleaning was rated below 3 stars") and when regulatory compliance requires clear, auditable data lineage without event replay.

**Best for:** Teams that prioritise data integrity, complex reporting, and SQL-first development over schema flexibility.

**Trade-offs:**
- Pro: Strong referential integrity prevents orphaned records and inconsistent state
- Pro: Complex cross-entity joins are natural and performant with proper indexing
- Pro: Well-understood by most developers; extensive tooling support
- Pro: Maps directly to OTA/OpenTravel message structures for channel integration
- Con: Schema migrations required for every new field or entity type
- Con: Many tables (80+) can feel overwhelming during onboarding
- Con: Jurisdiction-specific fields (varying permit types, tax structures) require either wide tables with nullable columns or additional junction tables
- Con: Historical state queries ("what was the rate on March 15?") require explicit history tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 3166-1 alpha-2 | `country_code` columns on addresses, jurisdictions, and guest nationality |
| ISO 3166-2 | `subdivision_code` for state/province in jurisdiction and address tables |
| ISO 4217 | `currency_code` (CHAR(3)) on all monetary amount columns |
| RFC 5545 (iCalendar) | Calendar export views generate iCal-compliant VEVENT/VFREEBUSY from reservation and availability tables |
| OpenTravel Alliance (OTA) | Entity names and field semantics align with OTA HotelRes, HotelAvail, and HotelDescriptiveInfo message types |
| OAuth 2.0 (RFC 6749) | `channel_connections` table stores OAuth tokens for Airbnb, Vrbo, Booking.com |
| PCI DSS v4.0 | Payment card data is NOT stored; `payments` table references external gateway tokens only |
| GDPR (EU 2016/679) | `guests` table supports soft-delete and data export; `consent_records` table tracks legal basis |
| ISO/IEC 27001 | Audit columns (`created_at`, `updated_at`, `created_by`) on every table |

---

## Core Identity & Multi-Tenancy

```sql
-- Organisation / account that owns the PMS subscription
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'free', -- free, professional, enterprise
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',       -- ISO 4217
    default_timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users who can log in and operate the system
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    role            VARCHAR(50) NOT NULL DEFAULT 'member', -- owner, admin, member, cleaner, viewer
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);
CREATE INDEX idx_users_org ON users(organisation_id);
CREATE INDEX idx_users_email ON users(email);

-- Role-based access control
CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role            VARCHAR(50) NOT NULL,
    resource        VARCHAR(100) NOT NULL,  -- e.g., 'properties', 'reservations', 'financials'
    action          VARCHAR(50) NOT NULL,   -- e.g., 'read', 'write', 'delete', 'admin'
    UNIQUE (role, resource, action)
);
```

## Property & Listing Management

```sql
-- Physical property (a house, apartment, cabin, etc.)
CREATE TABLE properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    property_type   VARCHAR(50) NOT NULL,  -- house, apartment, cabin, villa, condo, townhouse
    status          VARCHAR(50) NOT NULL DEFAULT 'active', -- active, inactive, onboarding, archived
    bedrooms        SMALLINT NOT NULL DEFAULT 1,
    bathrooms       NUMERIC(3,1) NOT NULL DEFAULT 1.0,
    max_guests      SMALLINT NOT NULL DEFAULT 2,
    area_sqft       INTEGER,
    description     TEXT,
    internal_notes  TEXT,
    check_in_time   TIME NOT NULL DEFAULT '15:00',
    check_out_time  TIME NOT NULL DEFAULT '11:00',
    min_stay_nights SMALLINT NOT NULL DEFAULT 1,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_properties_org ON properties(organisation_id);

-- Property address
CREATE TABLE property_addresses (
    property_id     UUID PRIMARY KEY REFERENCES properties(id),
    address_line_1  VARCHAR(255) NOT NULL,
    address_line_2  VARCHAR(255),
    city            VARCHAR(100) NOT NULL,
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL,         -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),               -- ISO 3166-2
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7)
);

-- Amenities catalogue (pool, wifi, parking, etc.)
CREATE TABLE amenities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category        VARCHAR(50) NOT NULL,  -- essentials, features, safety, accessibility
    name            VARCHAR(100) NOT NULL UNIQUE,
    icon            VARCHAR(50)
);

CREATE TABLE property_amenities (
    property_id     UUID NOT NULL REFERENCES properties(id),
    amenity_id      UUID NOT NULL REFERENCES amenities(id),
    notes           VARCHAR(255),
    PRIMARY KEY (property_id, amenity_id)
);

-- Property photos
CREATE TABLE property_photos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    url             TEXT NOT NULL,
    caption         VARCHAR(255),
    sort_order      SMALLINT NOT NULL DEFAULT 0,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    room_tag        VARCHAR(50),  -- bedroom, bathroom, kitchen, exterior, living_room
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_photos_property ON property_photos(property_id);

-- Property owners (the person/entity who owns the physical property)
CREATE TABLE property_owners (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    full_name       VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(50),
    tax_id          VARCHAR(50),
    payout_method   VARCHAR(50),  -- bank_transfer, cheque, paypal
    payout_details  TEXT,         -- encrypted or tokenised reference
    commission_pct  NUMERIC(5,2) NOT NULL DEFAULT 0.00,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_owners_org ON property_owners(organisation_id);

-- Many-to-many: a property can have multiple owners; an owner can own multiple properties
CREATE TABLE property_owner_assignments (
    property_id     UUID NOT NULL REFERENCES properties(id),
    owner_id        UUID NOT NULL REFERENCES property_owners(id),
    ownership_pct   NUMERIC(5,2) NOT NULL DEFAULT 100.00,
    effective_from  DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to    DATE,
    PRIMARY KEY (property_id, owner_id, effective_from)
);

-- Listings: a property's presence on a specific channel
CREATE TABLE listings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    channel_id      UUID NOT NULL REFERENCES channels(id),
    external_id     VARCHAR(255),       -- Airbnb listing ID, Vrbo property ID, etc.
    listing_url     TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft', -- draft, active, paused, unlisted
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    sync_enabled    BOOLEAN NOT NULL DEFAULT true,
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, channel_id)
);
CREATE INDEX idx_listings_property ON listings(property_id);
CREATE INDEX idx_listings_channel ON listings(channel_id);
CREATE INDEX idx_listings_external ON listings(external_id);
```

## Channel Integration

```sql
-- Supported distribution channels (Airbnb, Vrbo, Booking.com, Direct)
CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,  -- airbnb, vrbo, booking_com, direct, google
    name            VARCHAR(100) NOT NULL,
    api_type        VARCHAR(50),     -- rest, xml, ical_only
    is_active       BOOLEAN NOT NULL DEFAULT true
);

-- OAuth / API credentials per organisation per channel
CREATE TABLE channel_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    channel_id      UUID NOT NULL REFERENCES channels(id),
    auth_type       VARCHAR(50) NOT NULL,  -- oauth2, api_key, ical
    access_token    TEXT,                   -- encrypted
    refresh_token   TEXT,                   -- encrypted
    token_expires_at TIMESTAMPTZ,
    api_key         TEXT,                   -- encrypted
    partner_id      VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'connected',
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, channel_id)
);
CREATE INDEX idx_channel_conn_org ON channel_connections(organisation_id);

-- Sync log: tracks every inbound/outbound sync operation
CREATE TABLE channel_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_connection_id UUID NOT NULL REFERENCES channel_connections(id),
    direction       VARCHAR(10) NOT NULL,   -- inbound, outbound
    entity_type     VARCHAR(50) NOT NULL,   -- reservation, availability, rate, listing
    entity_id       UUID,
    status          VARCHAR(50) NOT NULL,   -- success, failed, partial
    request_payload TEXT,
    response_payload TEXT,
    error_message   TEXT,
    duration_ms     INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sync_log_conn ON channel_sync_log(channel_connection_id);
CREATE INDEX idx_sync_log_created ON channel_sync_log(created_at);
```

## Reservations & Guests

```sql
-- Guests
CREATE TABLE guests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(50),
    nationality     CHAR(2),                -- ISO 3166-1 alpha-2
    preferred_language VARCHAR(10),          -- ISO 639-1
    id_verified     BOOLEAN NOT NULL DEFAULT false,
    notes           TEXT,
    is_deleted      BOOLEAN NOT NULL DEFAULT false,  -- GDPR soft-delete
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_guests_org ON guests(organisation_id);
CREATE INDEX idx_guests_email ON guests(email);
CREATE INDEX idx_guests_name ON guests(organisation_id, last_name, first_name);

-- Reservations
CREATE TABLE reservations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    property_id     UUID NOT NULL REFERENCES properties(id),
    listing_id      UUID REFERENCES listings(id),
    guest_id        UUID NOT NULL REFERENCES guests(id),
    channel_id      UUID REFERENCES channels(id),
    external_id     VARCHAR(255),               -- channel's reservation/confirmation code
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
        -- pending, confirmed, checked_in, checked_out, cancelled, no_show
    check_in_date   DATE NOT NULL,
    check_out_date  DATE NOT NULL,
    nights          SMALLINT GENERATED ALWAYS AS (check_out_date - check_in_date) STORED,
    guests_count    SMALLINT NOT NULL DEFAULT 1,
    adults          SMALLINT NOT NULL DEFAULT 1,
    children        SMALLINT NOT NULL DEFAULT 0,
    infants         SMALLINT NOT NULL DEFAULT 0,
    pets            SMALLINT NOT NULL DEFAULT 0,
    nightly_rate    NUMERIC(12,2) NOT NULL,
    subtotal        NUMERIC(12,2) NOT NULL,
    cleaning_fee    NUMERIC(12,2) NOT NULL DEFAULT 0,
    service_fee     NUMERIC(12,2) NOT NULL DEFAULT 0,
    tax_total       NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    payout_amount   NUMERIC(12,2),
    special_requests TEXT,
    cancellation_policy VARCHAR(50),           -- flexible, moderate, strict, super_strict
    cancelled_at    TIMESTAMPTZ,
    cancellation_reason TEXT,
    booked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_dates CHECK (check_out_date > check_in_date)
);
CREATE INDEX idx_reservations_org ON reservations(organisation_id);
CREATE INDEX idx_reservations_property ON reservations(property_id);
CREATE INDEX idx_reservations_guest ON reservations(guest_id);
CREATE INDEX idx_reservations_dates ON reservations(property_id, check_in_date, check_out_date);
CREATE INDEX idx_reservations_status ON reservations(status);
CREATE INDEX idx_reservations_external ON reservations(external_id);

-- Nightly rate breakdown (allows different rates per night)
CREATE TABLE reservation_nightly_rates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reservation_id  UUID NOT NULL REFERENCES reservations(id) ON DELETE CASCADE,
    date            DATE NOT NULL,
    rate            NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    UNIQUE (reservation_id, date)
);
CREATE INDEX idx_nightly_rates_res ON reservation_nightly_rates(reservation_id);

-- Reservation status history
CREATE TABLE reservation_status_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reservation_id  UUID NOT NULL REFERENCES reservations(id),
    old_status      VARCHAR(50),
    new_status      VARCHAR(50) NOT NULL,
    changed_by      UUID REFERENCES users(id),
    reason          TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_res_status_history ON reservation_status_history(reservation_id);
```

## Calendar & Availability

```sql
-- Daily availability and pricing calendar per property
CREATE TABLE calendar_days (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    date            DATE NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'available',
        -- available, booked, blocked, maintenance
    reservation_id  UUID REFERENCES reservations(id),
    base_rate       NUMERIC(12,2),
    adjusted_rate   NUMERIC(12,2),       -- after dynamic pricing adjustments
    min_stay        SMALLINT,
    max_stay        SMALLINT,
    note            VARCHAR(255),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, date)
);
CREATE INDEX idx_calendar_property_date ON calendar_days(property_id, date);
CREATE INDEX idx_calendar_status ON calendar_days(status);

-- Seasonal / rule-based rate plans
CREATE TABLE rate_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    name            VARCHAR(100) NOT NULL,
    rate_type       VARCHAR(50) NOT NULL,  -- base, weekend, seasonal, event, last_minute
    nightly_rate    NUMERIC(12,2) NOT NULL,
    min_stay        SMALLINT NOT NULL DEFAULT 1,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    start_date      DATE,
    end_date        DATE,
    days_of_week    SMALLINT[],  -- 0=Sun, 1=Mon, ... 6=Sat; NULL = all days
    priority        SMALLINT NOT NULL DEFAULT 0,  -- higher = takes precedence
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rate_plans_property ON rate_plans(property_id);

-- Dynamic pricing recommendations from ML engine
CREATE TABLE pricing_recommendations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    date            DATE NOT NULL,
    recommended_rate NUMERIC(12,2) NOT NULL,
    confidence      NUMERIC(5,4),          -- 0.0000 to 1.0000
    factors         TEXT,                   -- human-readable explanation
    source          VARCHAR(50) NOT NULL,   -- internal_ml, pricelabs, beyond
    applied         BOOLEAN NOT NULL DEFAULT false,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, date, source)
);
CREATE INDEX idx_pricing_rec_property ON pricing_recommendations(property_id, date);
```

## Guest Communication

```sql
-- Conversation threads (one per reservation + guest)
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    reservation_id  UUID REFERENCES reservations(id),
    guest_id        UUID NOT NULL REFERENCES guests(id),
    channel_id      UUID REFERENCES channels(id),
    subject         VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'open', -- open, snoozed, closed, archived
    last_message_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_conversations_org ON conversations(organisation_id);
CREATE INDEX idx_conversations_reservation ON conversations(reservation_id);

-- Individual messages
CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id),
    sender_type     VARCHAR(20) NOT NULL,    -- guest, host, system, ai
    sender_id       UUID,                     -- user_id or guest_id
    channel         VARCHAR(50) NOT NULL,     -- email, sms, whatsapp, airbnb, vrbo, in_app
    body            TEXT NOT NULL,
    is_ai_generated BOOLEAN NOT NULL DEFAULT false,
    ai_model        VARCHAR(50),              -- claude-4, gpt-4o, etc.
    external_id     VARCHAR(255),
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_messages_conversation ON messages(conversation_id);
CREATE INDEX idx_messages_created ON messages(created_at);

-- Message templates (for automated workflows)
CREATE TABLE message_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(100) NOT NULL,
    trigger_event   VARCHAR(50) NOT NULL,  -- booking_confirmed, pre_arrival, check_in, check_out, review_request
    delay_hours     SMALLINT NOT NULL DEFAULT 0,
    subject         VARCHAR(255),
    body_template   TEXT NOT NULL,           -- supports {{guest_name}}, {{property_name}}, etc.
    channel         VARCHAR(50) NOT NULL DEFAULT 'email',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    language        VARCHAR(10) DEFAULT 'en',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_msg_templates_org ON message_templates(organisation_id);
```

## Operations — Cleaning & Maintenance

```sql
-- Cleaning tasks
CREATE TABLE cleaning_tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    property_id     UUID NOT NULL REFERENCES properties(id),
    reservation_id  UUID REFERENCES reservations(id),
    assigned_to     UUID REFERENCES users(id),
    task_type       VARCHAR(50) NOT NULL,    -- turnover, deep_clean, inspection, mid_stay
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
        -- pending, assigned, in_progress, completed, cancelled
    scheduled_date  DATE NOT NULL,
    scheduled_time  TIME,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    duration_minutes INTEGER,
    rating          SMALLINT,                -- 1-5 quality rating
    notes           TEXT,
    checklist_completed BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cleaning_property ON cleaning_tasks(property_id);
CREATE INDEX idx_cleaning_assigned ON cleaning_tasks(assigned_to);
CREATE INDEX idx_cleaning_date ON cleaning_tasks(scheduled_date);
CREATE INDEX idx_cleaning_status ON cleaning_tasks(status);

-- Cleaning checklist items
CREATE TABLE cleaning_checklist_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cleaning_task_id UUID NOT NULL REFERENCES cleaning_tasks(id) ON DELETE CASCADE,
    item_name       VARCHAR(255) NOT NULL,
    is_completed    BOOLEAN NOT NULL DEFAULT false,
    completed_at    TIMESTAMPTZ,
    photo_url       TEXT,
    sort_order      SMALLINT NOT NULL DEFAULT 0
);
CREATE INDEX idx_checklist_task ON cleaning_checklist_items(cleaning_task_id);

-- Maintenance requests
CREATE TABLE maintenance_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    property_id     UUID NOT NULL REFERENCES properties(id),
    reported_by     UUID REFERENCES users(id),
    reservation_id  UUID REFERENCES reservations(id),
    category        VARCHAR(50) NOT NULL,    -- plumbing, electrical, appliance, hvac, structural, cosmetic
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium', -- low, medium, high, urgent
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
        -- open, assigned, in_progress, waiting_parts, completed, cancelled
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    assigned_to     UUID REFERENCES users(id),
    vendor_name     VARCHAR(255),
    estimated_cost  NUMERIC(12,2),
    actual_cost     NUMERIC(12,2),
    currency_code   CHAR(3) DEFAULT 'USD',
    scheduled_date  DATE,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_maintenance_property ON maintenance_requests(property_id);
CREATE INDEX idx_maintenance_status ON maintenance_requests(status);
CREATE INDEX idx_maintenance_priority ON maintenance_requests(priority);
```

## Financial Management

```sql
-- Payments received or refunded
CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    reservation_id  UUID REFERENCES reservations(id),
    payment_type    VARCHAR(50) NOT NULL,    -- booking, security_deposit, refund, payout, adjustment
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
        -- pending, processing, completed, failed, refunded
    amount          NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    payment_method  VARCHAR(50),             -- card, bank_transfer, paypal, cash, channel_collect
    gateway         VARCHAR(50),             -- stripe, paypal, manual, airbnb, vrbo
    gateway_txn_id  VARCHAR(255),
    gateway_fee     NUMERIC(12,2),
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payments_org ON payments(organisation_id);
CREATE INDEX idx_payments_reservation ON payments(reservation_id);
CREATE INDEX idx_payments_status ON payments(status);

-- Owner statements / payout reports
CREATE TABLE owner_statements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    owner_id        UUID NOT NULL REFERENCES property_owners(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    gross_revenue   NUMERIC(12,2) NOT NULL,
    management_fee  NUMERIC(12,2) NOT NULL,
    expenses        NUMERIC(12,2) NOT NULL DEFAULT 0,
    net_payout      NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(50) NOT NULL DEFAULT 'draft', -- draft, approved, paid
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_owner_stmts_owner ON owner_statements(owner_id);
CREATE INDEX idx_owner_stmts_period ON owner_statements(period_start, period_end);

-- Line items on owner statements
CREATE TABLE owner_statement_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id    UUID NOT NULL REFERENCES owner_statements(id) ON DELETE CASCADE,
    reservation_id  UUID REFERENCES reservations(id),
    item_type       VARCHAR(50) NOT NULL,  -- rental_income, cleaning_fee, management_fee, maintenance, tax, adjustment
    description     VARCHAR(255) NOT NULL,
    amount          NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD'
);
CREATE INDEX idx_stmt_items_stmt ON owner_statement_items(statement_id);

-- Tax records
CREATE TABLE tax_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    reservation_id  UUID NOT NULL REFERENCES reservations(id),
    tax_type        VARCHAR(50) NOT NULL,    -- occupancy_tax, sales_tax, vat, tourism_levy
    jurisdiction_id UUID REFERENCES jurisdictions(id),
    tax_rate        NUMERIC(6,4) NOT NULL,   -- e.g., 0.1200 = 12%
    taxable_amount  NUMERIC(12,2) NOT NULL,
    tax_amount      NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    remitted        BOOLEAN NOT NULL DEFAULT false,
    remitted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tax_records_reservation ON tax_records(reservation_id);
CREATE INDEX idx_tax_records_jurisdiction ON tax_records(jurisdiction_id);
```

## Regulatory Compliance

```sql
-- Jurisdictions (countries, states, cities with STR regulations)
CREATE TABLE jurisdictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    level           VARCHAR(50) NOT NULL,    -- country, state, county, city
    parent_id       UUID REFERENCES jurisdictions(id),
    country_code    CHAR(2) NOT NULL,        -- ISO 3166-1
    subdivision_code VARCHAR(6),              -- ISO 3166-2
    city_name       VARCHAR(100),
    max_nights_per_year SMALLINT,
    requires_permit BOOLEAN NOT NULL DEFAULT false,
    requires_tax_registration BOOLEAN NOT NULL DEFAULT false,
    occupancy_tax_rate NUMERIC(6,4),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_jurisdictions_country ON jurisdictions(country_code);
CREATE INDEX idx_jurisdictions_parent ON jurisdictions(parent_id);

-- Permits and licences held per property
CREATE TABLE property_permits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    permit_type     VARCHAR(50) NOT NULL,    -- str_licence, business_licence, fire_safety, health_inspection
    permit_number   VARCHAR(100),
    issued_date     DATE,
    expiry_date     DATE,
    status          VARCHAR(50) NOT NULL DEFAULT 'active', -- pending, active, expired, revoked
    renewal_reminder_days SMALLINT DEFAULT 30,
    document_url    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_permits_property ON property_permits(property_id);
CREATE INDEX idx_permits_expiry ON property_permits(expiry_date);
CREATE INDEX idx_permits_status ON property_permits(status);

-- GDPR consent records
CREATE TABLE consent_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guest_id        UUID NOT NULL REFERENCES guests(id),
    consent_type    VARCHAR(50) NOT NULL,    -- marketing, data_processing, analytics
    granted         BOOLEAN NOT NULL,
    ip_address      INET,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ
);
CREATE INDEX idx_consent_guest ON consent_records(guest_id);
```

## Reviews & Reputation

```sql
-- Guest reviews (received and sent)
CREATE TABLE reviews (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    reservation_id  UUID NOT NULL REFERENCES reservations(id),
    property_id     UUID NOT NULL REFERENCES properties(id),
    guest_id        UUID NOT NULL REFERENCES guests(id),
    direction       VARCHAR(20) NOT NULL,    -- guest_to_host, host_to_guest
    channel_id      UUID REFERENCES channels(id),
    overall_rating  NUMERIC(3,1),            -- 1.0 to 5.0
    cleanliness     NUMERIC(3,1),
    communication   NUMERIC(3,1),
    check_in        NUMERIC(3,1),
    accuracy        NUMERIC(3,1),
    location        NUMERIC(3,1),
    value           NUMERIC(3,1),
    body            TEXT,
    response        TEXT,
    is_public       BOOLEAN NOT NULL DEFAULT true,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_reviews_property ON reviews(property_id);
CREATE INDEX idx_reviews_reservation ON reviews(reservation_id);
CREATE INDEX idx_reviews_guest ON reviews(guest_id);
```

## Guest Guidebooks

```sql
-- Digital guidebooks per property
CREATE TABLE guidebooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    title           VARCHAR(255) NOT NULL,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    share_url       TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_guidebooks_property ON guidebooks(property_id);

-- Guidebook sections (house rules, wifi info, local recommendations)
CREATE TABLE guidebook_sections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guidebook_id    UUID NOT NULL REFERENCES guidebooks(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    body            TEXT NOT NULL,
    section_type    VARCHAR(50) NOT NULL,  -- house_rules, wifi, check_in, local_tips, emergency, transport
    sort_order      SMALLINT NOT NULL DEFAULT 0,
    icon            VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_guidebook_sections ON guidebook_sections(guidebook_id);
```

## Audit Trail

```sql
-- General audit log for all entity changes
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID REFERENCES users(id),
    entity_type     VARCHAR(50) NOT NULL,   -- reservation, property, guest, payment, etc.
    entity_id       UUID NOT NULL,
    action          VARCHAR(50) NOT NULL,   -- create, update, delete, status_change
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organisations, users, permissions |
| Property & Listing Management | 8 | properties, addresses, amenities, photos, owners, assignments, listings |
| Channel Integration | 3 | channels, channel_connections, channel_sync_log |
| Reservations & Guests | 4 | guests, reservations, nightly_rates, status_history |
| Calendar & Availability | 3 | calendar_days, rate_plans, pricing_recommendations |
| Guest Communication | 3 | conversations, messages, message_templates |
| Operations | 4 | cleaning_tasks, checklist_items, maintenance_requests |
| Financial Management | 5 | payments, owner_statements, statement_items, tax_records |
| Regulatory Compliance | 3 | jurisdictions, property_permits, consent_records |
| Reviews & Reputation | 1 | reviews |
| Guest Guidebooks | 2 | guidebooks, guidebook_sections |
| Audit Trail | 1 | audit_log |
| **Total** | **40** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation across multiple application servers and avoids sequential ID enumeration attacks. Aligns with modern SaaS practice used by Guesty, Hostaway, and Stripe.

2. **Organisation-scoped multi-tenancy via foreign keys** — every tenant-owned table references `organisation_id`. Row-level security (RLS) policies can be applied at the PostgreSQL level to enforce tenant isolation without application-layer bugs.

3. **Separate `properties` vs `listings` tables** — a property is a physical building; a listing is that property's representation on a specific channel. One property can have listings on Airbnb, Vrbo, and Booking.com simultaneously, each with its own external ID, title, and sync status. This mirrors how Guesty and Hostaway model the distinction.

4. **Calendar-day-level availability and pricing** — the `calendar_days` table stores one row per property per date, enabling granular rate overrides and availability blocks. This is the pattern used by Airbnb's Connectivity API and PriceLabs for per-night pricing.

5. **Nightly rate breakdown on reservations** — `reservation_nightly_rates` allows different rates per night within a single stay (weekend vs weekday, dynamic pricing changes). This matches the Booking.com and Vrbo API patterns.

6. **Explicit jurisdiction hierarchy** — the `jurisdictions` table uses self-referential `parent_id` to model country > state > county > city hierarchies. Each jurisdiction tracks its own STR regulations (permit requirements, max nights, tax rates), supporting the complex regulatory landscape.

7. **Owner statements with line items** — financial reporting follows a statement > line-items pattern, enabling per-reservation revenue attribution and transparent owner payouts with management fee deductions.

8. **GDPR compliance built in** — `guests.is_deleted` supports soft-delete for right-to-erasure requests; `consent_records` tracks explicit consent grants and revocations with timestamps and IP addresses.

9. **Channel-agnostic messaging model** — the `conversations` and `messages` tables handle communication across all channels (Airbnb in-app, SMS via Twilio, email, WhatsApp) with a unified schema, and flag AI-generated responses for transparency.

10. **Audit log with old/new value capture** — the `audit_log` table uses JSONB columns to store before/after snapshots of changed fields, providing compliance-ready change tracking without requiring event replay infrastructure.

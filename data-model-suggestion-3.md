# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Vacation Rental Management · Created: 2026-05-22

## Philosophy

This model keeps the structural backbone relational — properties, reservations, guests, and payments live in typed columns with foreign keys and constraints — but pushes variable, jurisdiction-specific, channel-specific, and rapidly-evolving fields into JSONB columns. The result is a schema that is stable where stability matters (you will always have a `check_in_date DATE`) and flexible where flexibility matters (Airbnb sends different metadata than Booking.com; Tokyo STR regulations require different permit fields than Austin).

This approach is inspired by how modern SaaS platforms like Stripe and Shopify design their schemas: a core relational model with a `metadata JSONB` column on nearly every table. It is the pragmatic middle ground for a startup that needs to ship an MVP quickly, iterate on schema based on real customer feedback, and support the wildly heterogeneous vacation rental regulatory landscape without a migration for every new city's permit requirements.

PostgreSQL's JSONB support is mature enough to make this viable at scale: GIN indexes on JSONB columns enable fast containment queries (`@>` operator), partial indexes can target specific JSON paths, and generated columns can extract frequently-queried JSON fields into indexable typed columns when performance demands it.

**Best for:** Startups building an MVP that must support multi-channel, multi-jurisdiction operations from day one without over-engineering the schema. Teams that iterate rapidly and prefer adding a JSON field over running a migration.

**Trade-offs:**
- Pro: Fastest path to MVP — fewer tables, fewer migrations, faster iteration
- Pro: Naturally accommodates channel-specific metadata (Airbnb vs Vrbo vs Booking.com each send different fields)
- Pro: Jurisdiction-specific compliance fields (permit types, tax structures) don't require schema changes
- Pro: Custom fields per organisation are trivial — just add keys to the JSONB column
- Pro: PostgreSQL JSONB is battle-tested with GIN indexing, containment operators, and JSON path queries
- Con: No database-level type enforcement on JSONB fields — validation must happen in the application layer
- Con: JSONB fields can become "junk drawers" without disciplined documentation and JSON Schema validation
- Con: Complex queries on deeply nested JSONB can be slower than equivalent relational joins
- Con: ORM support for JSONB is inconsistent — some frameworks treat it as opaque
- Con: Schema discovery requires reading documentation rather than inspecting table definitions

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 3166-1 alpha-2 | `country_code CHAR(2)` columns on addresses; also in jurisdiction JSONB as `"country": "US"` |
| ISO 3166-2 | Subdivision codes in jurisdiction and address JSONB fields |
| ISO 4217 | `currency_code CHAR(3)` as typed column on all monetary tables |
| RFC 5545 (iCalendar) | Calendar export generates iCal from `calendar_days` table |
| JSON Schema Draft 2020-12 | JSONB columns validated against JSON Schema in the application layer; schemas stored in `jsonb_schemas` table |
| OpenTravel Alliance (OTA) | Channel-specific JSONB metadata preserves original OTA field names for round-trip fidelity |
| OpenAPI 3.1+ | API documentation uses JSON Schema (compatible with JSONB validation schemas) |
| GDPR (EU 2016/679) | Guest PII in relational columns supports targeted deletion; JSONB `preferences` excluded from PII scope |

---

## Schema Documentation Table

```sql
-- Documents the expected structure of every JSONB column in the system
-- Application-layer validation uses these schemas; they also serve as developer documentation
CREATE TABLE jsonb_schemas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name      VARCHAR(100) NOT NULL,
    column_name     VARCHAR(100) NOT NULL,
    description     TEXT,
    json_schema     JSONB NOT NULL,           -- JSON Schema Draft 2020-12
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (table_name, column_name, version)
);
```

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'free',
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    default_timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    -- JSONB: branding, feature flags, notification preferences, custom terminology
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "branding": {"logo_url": "...", "primary_color": "#2563EB"},
    --   "features": {"ai_pricing": true, "ai_messaging": true, "guidebooks": false},
    --   "notifications": {"email": true, "sms": false, "slack_webhook": "https://..."},
    --   "terminology": {"property": "unit", "guest": "traveler"},
    --   "tax_settings": {"auto_remit": false, "default_tax_rate": 0.12}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: notification preferences, UI preferences, assigned property groups
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "notifications": {"new_booking": "push", "message": "email+push"},
    --   "dashboard_layout": "calendar",
    --   "property_groups": ["beachfront", "downtown"]
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);
CREATE INDEX idx_users_org ON users(organisation_id);
CREATE INDEX idx_users_email ON users(email);
```

## Property Management

```sql
CREATE TABLE properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    property_type   VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',

    -- Core relational fields (always present, always typed)
    bedrooms        SMALLINT NOT NULL DEFAULT 1,
    bathrooms       NUMERIC(3,1) NOT NULL DEFAULT 1.0,
    max_guests      SMALLINT NOT NULL DEFAULT 2,
    check_in_time   TIME NOT NULL DEFAULT '15:00',
    check_out_time  TIME NOT NULL DEFAULT '11:00',
    min_stay_nights SMALLINT NOT NULL DEFAULT 1,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',

    -- Address (relational for indexing and geospatial queries)
    address_line_1  VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2),                  -- ISO 3166-1
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),

    -- JSONB: everything that varies by property type, region, or preference
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "area_sqft": 1800,
    --   "floor": 3,
    --   "year_built": 2015,
    --   "parking": {"type": "garage", "spaces": 2},
    --   "amenities": ["wifi", "pool", "hot_tub", "ac", "washer", "dryer", "bbq"],
    --   "house_rules": ["no_smoking", "no_parties", "quiet_hours_10pm"],
    --   "accessibility": ["wheelchair_ramp", "ground_floor_bedroom"],
    --   "smart_home": {
    --     "lock_type": "schlage_encode",
    --     "thermostat": "nest",
    --     "lock_api_id": "lock-123"
    --   },
    --   "description": "Spacious beachfront villa with ocean views...",
    --   "internal_notes": "Key is under the blue pot on the left side."
    -- }

    -- JSONB: owner information (supports multiple owners with varying structures)
    owners          JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "owner_id": "uuid",
    --     "name": "Jane Smith",
    --     "email": "jane@example.com",
    --     "ownership_pct": 60,
    --     "commission_pct": 20,
    --     "payout_method": "bank_transfer",
    --     "tax_id": "XX-XXXXXXX"
    --   },
    --   {
    --     "owner_id": "uuid",
    --     "name": "Bob Smith",
    --     "ownership_pct": 40,
    --     "commission_pct": 20
    --   }
    -- ]

    -- JSONB: photos with flexible metadata
    photos          JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"url": "https://...", "caption": "Living room", "room": "living_room", "is_primary": true, "order": 1},
    --   {"url": "https://...", "caption": "Master bedroom", "room": "bedroom", "order": 2}
    -- ]

    -- JSONB: compliance data (varies dramatically by jurisdiction)
    compliance      JSONB NOT NULL DEFAULT '{}',
    -- Example (Austin, TX):
    -- {
    --   "jurisdiction": {"country": "US", "state": "TX", "city": "Austin"},
    --   "permits": [
    --     {"type": "str_licence", "number": "STR-2024-12345", "status": "active",
    --      "issued": "2024-01-15", "expires": "2025-01-15", "document_url": "https://..."}
    --   ],
    --   "max_nights_per_year": null,
    --   "occupancy_tax_rate": 0.09,
    --   "requires_fire_inspection": true,
    --   "fire_inspection_date": "2024-06-01",
    --   "insurance_policy": "POL-123456"
    -- }
    -- Example (Tokyo, Japan):
    -- {
    --   "jurisdiction": {"country": "JP", "prefecture": "Tokyo", "ward": "Shibuya"},
    --   "permits": [
    --     {"type": "minpaku_registration", "number": "M1300001234", "status": "active",
    --      "issued": "2024-03-01", "expires": null}
    --   ],
    --   "max_nights_per_year": 180,
    --   "nights_used_this_year": 45,
    --   "requires_neighborhood_notification": true,
    --   "notification_date": "2024-02-15",
    --   "fire_safety_equipment": ["smoke_detector", "fire_extinguisher", "emergency_lighting"]
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_properties_org ON properties(organisation_id);
CREATE INDEX idx_properties_status ON properties(status);
CREATE INDEX idx_properties_country ON properties(country_code);
CREATE INDEX idx_properties_location ON properties USING gist (
    ll_to_earth(latitude, longitude)
) WHERE latitude IS NOT NULL AND longitude IS NOT NULL;
-- GIN index on JSONB for amenity and compliance queries
CREATE INDEX idx_properties_details ON properties USING gin (details);
CREATE INDEX idx_properties_compliance ON properties USING gin (compliance);

-- Example queries using JSONB:

-- Find all properties with a pool:
-- SELECT * FROM properties WHERE details @> '{"amenities": ["pool"]}';

-- Find all properties in Austin, TX:
-- SELECT * FROM properties WHERE compliance @> '{"jurisdiction": {"city": "Austin", "state": "TX"}}';

-- Find all properties with expiring permits:
-- SELECT id, name, permit
-- FROM properties, jsonb_array_elements(compliance->'permits') AS permit
-- WHERE (permit->>'expires')::date < CURRENT_DATE + INTERVAL '30 days'
--   AND permit->>'status' = 'active';
```

## Channel & Listings

```sql
CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL,
    api_type        VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: channel-specific configuration (API endpoints, rate limits, field mappings)
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "base_url": "https://api.hostaway.com/v1",
    --   "rate_limit_per_minute": 60,
    --   "supported_features": ["instant_book", "smart_pricing", "reviews"],
    --   "field_mapping": {"property_type": "roomType", "max_guests": "personCapacity"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE listings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    channel_id      UUID NOT NULL REFERENCES channels(id),
    external_id     VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    sync_enabled    BOOLEAN NOT NULL DEFAULT true,
    last_synced_at  TIMESTAMPTZ,

    -- JSONB: channel-specific listing data that varies per OTA
    channel_data    JSONB NOT NULL DEFAULT '{}',
    -- Example (Airbnb):
    -- {
    --   "listing_url": "https://airbnb.com/rooms/12345",
    --   "instant_book": true,
    --   "superhost": true,
    --   "smart_pricing_enabled": false,
    --   "listing_type": "entire_home",
    --   "cancellation_policy": "moderate",
    --   "airbnb_category": "beachfront",
    --   "co_host_ids": ["abc123"]
    -- }
    -- Example (Booking.com):
    -- {
    --   "listing_url": "https://booking.com/hotel/us/...",
    --   "property_id_bcom": 12345678,
    --   "room_type_id": 987654,
    --   "genius_eligible": true,
    --   "cancellation_policy_code": "FLEX",
    --   "commission_rate": 0.15,
    --   "content_score": 92
    -- }

    -- JSONB: credentials for this specific listing's channel connection
    connection      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "auth_type": "oauth2",
    --   "access_token": "encrypted:...",
    --   "refresh_token": "encrypted:...",
    --   "token_expires_at": "2026-06-01T00:00:00Z",
    --   "partner_id": "partner-123"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, channel_id)
);
CREATE INDEX idx_listings_property ON listings(property_id);
CREATE INDEX idx_listings_channel ON listings(channel_id);
CREATE INDEX idx_listings_external ON listings(external_id);
```

## Reservations & Guests

```sql
CREATE TABLE guests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(50),
    nationality     CHAR(2),
    preferred_language VARCHAR(10),

    -- JSONB: flexible guest profile data
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "id_verified": true,
    --   "verification_method": "government_id",
    --   "airbnb_profile_url": "https://airbnb.com/users/show/123",
    --   "previous_stay_count": 3,
    --   "average_rating_given": 4.2,
    --   "tags": ["repeat_guest", "pet_owner", "business_traveler"],
    --   "preferences": {"early_checkin": true, "extra_towels": true},
    --   "notes": "Allergic to cats. Prefers ground-floor units.",
    --   "screening": {
    --     "risk_score": 0.15,
    --     "screened_at": "2026-05-01T10:00:00Z",
    --     "flags": []
    --   }
    -- }

    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_guests_org ON guests(organisation_id);
CREATE INDEX idx_guests_email ON guests(email);
CREATE INDEX idx_guests_name ON guests(organisation_id, last_name, first_name);
CREATE INDEX idx_guests_profile ON guests USING gin (profile);

CREATE TABLE reservations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    property_id     UUID NOT NULL REFERENCES properties(id),
    listing_id      UUID REFERENCES listings(id),
    guest_id        UUID NOT NULL REFERENCES guests(id),
    channel_code    VARCHAR(50),
    external_id     VARCHAR(255),

    -- Core relational fields (always present, always typed)
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    check_in_date   DATE NOT NULL,
    check_out_date  DATE NOT NULL,
    nights          SMALLINT GENERATED ALWAYS AS (check_out_date - check_in_date) STORED,
    guests_count    SMALLINT NOT NULL DEFAULT 1,
    adults          SMALLINT NOT NULL DEFAULT 1,
    children        SMALLINT NOT NULL DEFAULT 0,

    -- Financial (typed for aggregation and reporting)
    nightly_rate    NUMERIC(12,2) NOT NULL,
    subtotal        NUMERIC(12,2) NOT NULL,
    cleaning_fee    NUMERIC(12,2) NOT NULL DEFAULT 0,
    service_fee     NUMERIC(12,2) NOT NULL DEFAULT 0,
    tax_total       NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    payout_amount   NUMERIC(12,2),

    -- JSONB: nightly rate breakdown, channel metadata, guest details, tax breakdown
    booking_details JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "nightly_rates": [
    --     {"date": "2026-07-15", "rate": 250.00, "source": "dynamic_pricing"},
    --     {"date": "2026-07-16", "rate": 250.00, "source": "dynamic_pricing"},
    --     {"date": "2026-07-17", "rate": 275.00, "source": "weekend_rate"},
    --     {"date": "2026-07-18", "rate": 275.00, "source": "weekend_rate"},
    --     {"date": "2026-07-19", "rate": 300.00, "source": "event_surge"}
    --   ],
    --   "tax_breakdown": [
    --     {"type": "occupancy_tax", "rate": 0.09, "amount": 117.00, "jurisdiction": "Austin, TX"},
    --     {"type": "state_hotel_tax", "rate": 0.06, "amount": 78.00, "jurisdiction": "Texas"}
    --   ],
    --   "fees": [
    --     {"type": "pet_fee", "amount": 50.00},
    --     {"type": "late_checkout_fee", "amount": 25.00}
    --   ],
    --   "special_requests": "Need a crib for the baby. Arriving late around 10pm.",
    --   "cancellation_policy": "moderate",
    --   "infants": 1,
    --   "pets": 1
    -- }

    -- JSONB: channel-specific reservation metadata
    channel_metadata JSONB NOT NULL DEFAULT '{}',
    -- Example (Airbnb):
    -- {
    --   "confirmation_code": "HMAK123456",
    --   "thread_id": "thread_abc",
    --   "instant_book": true,
    --   "host_payout": 1250.00,
    --   "airbnb_service_fee": 187.50,
    --   "guest_service_fee": 225.00,
    --   "resolution_id": null
    -- }

    booked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    cancelled_at    TIMESTAMPTZ,
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
CREATE INDEX idx_reservations_details ON reservations USING gin (booking_details);
CREATE INDEX idx_reservations_channel ON reservations USING gin (channel_metadata);
```

## Calendar & Pricing

```sql
CREATE TABLE calendar_days (
    property_id     UUID NOT NULL REFERENCES properties(id),
    date            DATE NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'available',
    reservation_id  UUID REFERENCES reservations(id),
    base_rate       NUMERIC(12,2),
    adjusted_rate   NUMERIC(12,2),
    min_stay        SMALLINT,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',

    -- JSONB: pricing metadata, rate sources, and overrides
    pricing_data    JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "rate_source": "dynamic_pricing",
    --   "ml_confidence": 0.87,
    --   "competitor_avg_rate": 285.00,
    --   "demand_score": 0.72,
    --   "event_name": "SXSW 2026",
    --   "manual_override": false,
    --   "override_reason": null,
    --   "seasonal_multiplier": 1.15,
    --   "occupancy_adjustment": 0.95,
    --   "last_updated_by": "ai_pricing_agent"
    -- }

    note            VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (property_id, date)
);
CREATE INDEX idx_calendar_status ON calendar_days(status);
CREATE INDEX idx_calendar_date ON calendar_days(date);

CREATE TABLE rate_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    name            VARCHAR(100) NOT NULL,
    rate_type       VARCHAR(50) NOT NULL,
    nightly_rate    NUMERIC(12,2) NOT NULL,
    min_stay        SMALLINT NOT NULL DEFAULT 1,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    start_date      DATE,
    end_date        DATE,
    priority        SMALLINT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: complex rate rules that vary per plan type
    rules           JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "days_of_week": [5, 6],
    --   "length_of_stay_discounts": [
    --     {"min_nights": 7, "discount_pct": 10},
    --     {"min_nights": 28, "discount_pct": 25}
    --   ],
    --   "early_bird_discount": {"days_ahead": 60, "discount_pct": 5},
    --   "last_minute_surcharge": {"days_ahead": 3, "surcharge_pct": 10},
    --   "gap_fill": {"max_gap_nights": 2, "discount_pct": 15},
    --   "channel_adjustments": {"airbnb": 0, "vrbo": 5, "direct": -10}
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rate_plans_property ON rate_plans(property_id);
```

## Guest Communication

```sql
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    reservation_id  UUID REFERENCES reservations(id),
    guest_id        UUID NOT NULL REFERENCES guests(id),
    channel_code    VARCHAR(50),
    subject         VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
    last_message_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_conversations_org ON conversations(organisation_id);
CREATE INDEX idx_conversations_reservation ON conversations(reservation_id);
CREATE INDEX idx_conversations_status ON conversations(status);

CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id),
    sender_type     VARCHAR(20) NOT NULL,
    sender_id       UUID,
    channel         VARCHAR(50) NOT NULL,
    body            TEXT NOT NULL,

    -- JSONB: AI metadata, translation info, attachments, delivery status
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "is_ai_generated": true,
    --   "ai_model": "claude-4",
    --   "ai_confidence": 0.95,
    --   "original_language": "ja",
    --   "translated_from": "ja",
    --   "translation_engine": "google",
    --   "attachments": [
    --     {"type": "image", "url": "https://...", "filename": "damage_photo.jpg"}
    --   ],
    --   "delivery_status": "delivered",
    --   "external_message_id": "msg-abc-123",
    --   "sentiment_score": 0.82
    -- }

    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_messages_conversation ON messages(conversation_id);
CREATE INDEX idx_messages_created ON messages(created_at);

CREATE TABLE message_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(100) NOT NULL,
    trigger_event   VARCHAR(50) NOT NULL,
    delay_hours     SMALLINT NOT NULL DEFAULT 0,
    subject         VARCHAR(255),
    body_template   TEXT NOT NULL,
    channel         VARCHAR(50) NOT NULL DEFAULT 'email',
    language        VARCHAR(10) DEFAULT 'en',
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: conditional logic, A/B test variants
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "conditions": {"min_stay_nights": 3, "channels": ["airbnb", "direct"]},
    --   "variants": [
    --     {"name": "A", "weight": 50, "body": "Welcome to {{property_name}}!..."},
    --     {"name": "B", "weight": 50, "body": "Hi {{guest_name}}, we're excited..."}
    --   ],
    --   "personalization_level": "high"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_msg_templates_org ON message_templates(organisation_id);
```

## Operations

```sql
CREATE TABLE tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    property_id     UUID NOT NULL REFERENCES properties(id),
    reservation_id  UUID REFERENCES reservations(id),
    task_type       VARCHAR(50) NOT NULL,     -- cleaning, maintenance, inspection, supplies
    category        VARCHAR(50),              -- for maintenance: plumbing, electrical, hvac, etc.
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    assigned_to     UUID REFERENCES users(id),
    scheduled_date  DATE,
    scheduled_time  TIME,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    duration_minutes INTEGER,

    -- JSONB: task-type-specific data (cleaning checklists, maintenance details, vendor info)
    task_data       JSONB NOT NULL DEFAULT '{}',
    -- Example (cleaning):
    -- {
    --   "checklist": [
    --     {"item": "Strip and remake beds", "completed": true, "photo_url": "https://..."},
    --     {"item": "Clean bathrooms", "completed": true},
    --     {"item": "Vacuum all rooms", "completed": false},
    --     {"item": "Restock toiletries", "completed": false},
    --     {"item": "Check for damage", "completed": false}
    --   ],
    --   "quality_rating": 4,
    --   "supply_usage": {"toilet_paper_rolls": 4, "soap_bars": 2},
    --   "time_estimate_minutes": 90,
    --   "predicted_by_ml": true,
    --   "photos_after": ["https://...", "https://..."]
    -- }
    -- Example (maintenance):
    -- {
    --   "vendor": {"name": "ABC Plumbing", "phone": "512-555-0100", "email": "abc@plumbing.com"},
    --   "estimated_cost": 350.00,
    --   "actual_cost": 425.00,
    --   "currency": "USD",
    --   "parts": ["kitchen faucet cartridge", "supply line"],
    --   "photos_before": ["https://..."],
    --   "photos_after": ["https://..."],
    --   "warranty_expiry": "2027-05-22",
    --   "reported_by_guest": true,
    --   "ai_detected": false
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tasks_org ON tasks(organisation_id);
CREATE INDEX idx_tasks_property ON tasks(property_id);
CREATE INDEX idx_tasks_assigned ON tasks(assigned_to);
CREATE INDEX idx_tasks_date ON tasks(scheduled_date);
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_type ON tasks(task_type);
CREATE INDEX idx_tasks_data ON tasks USING gin (task_data);
```

## Financial Management

```sql
CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    reservation_id  UUID REFERENCES reservations(id),
    payment_type    VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    amount          NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    gateway         VARCHAR(50),
    gateway_txn_id  VARCHAR(255),

    -- JSONB: gateway-specific response data, refund details, dispute info
    gateway_data    JSONB NOT NULL DEFAULT '{}',
    -- Example (Stripe):
    -- {
    --   "stripe_payment_intent_id": "pi_3Abc...",
    --   "stripe_charge_id": "ch_3Abc...",
    --   "card_brand": "visa",
    --   "card_last4": "4242",
    --   "card_country": "US",
    --   "stripe_fee": 4.59,
    --   "net_amount": 153.41,
    --   "receipt_url": "https://...",
    --   "3ds_authenticated": true
    -- }

    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payments_org ON payments(organisation_id);
CREATE INDEX idx_payments_reservation ON payments(reservation_id);
CREATE INDEX idx_payments_status ON payments(status);

CREATE TABLE owner_statements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    owner_id        UUID NOT NULL,             -- references owner in properties.owners JSONB
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    gross_revenue   NUMERIC(12,2) NOT NULL,
    management_fee  NUMERIC(12,2) NOT NULL,
    expenses        NUMERIC(12,2) NOT NULL DEFAULT 0,
    net_payout      NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    paid_at         TIMESTAMPTZ,

    -- JSONB: line items, property breakdown, tax summary
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "line_items": [
    --     {"reservation_id": "uuid", "property": "Beach House", "dates": "Jul 15-20",
    --      "gross": 1662.50, "cleaning": 150.00, "commission": 332.50, "net": 1180.00},
    --     {"reservation_id": "uuid", "property": "Downtown Apt", "dates": "Jul 18-22",
    --      "gross": 800.00, "cleaning": 100.00, "commission": 160.00, "net": 540.00}
    --   ],
    --   "expense_items": [
    --     {"type": "maintenance", "description": "Faucet repair", "amount": 425.00, "property": "Beach House"}
    --   ],
    --   "tax_summary": {"occupancy_tax_collected": 195.00, "remitted": true}
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_owner_stmts_org ON owner_statements(organisation_id);
CREATE INDEX idx_owner_stmts_period ON owner_statements(period_start, period_end);
```

## Reviews & Guidebooks

```sql
CREATE TABLE reviews (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    reservation_id  UUID NOT NULL REFERENCES reservations(id),
    property_id     UUID NOT NULL REFERENCES properties(id),
    guest_id        UUID NOT NULL REFERENCES guests(id),
    direction       VARCHAR(20) NOT NULL,
    channel_code    VARCHAR(50),
    overall_rating  NUMERIC(3,1),

    -- JSONB: channel-specific rating categories, response, AI analysis
    review_data     JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "ratings": {
    --     "cleanliness": 4.0, "communication": 5.0, "check_in": 5.0,
    --     "accuracy": 4.0, "location": 5.0, "value": 3.0
    --   },
    --   "body": "Great location but the kitchen was a bit dated...",
    --   "response": "Thank you for your feedback! We're updating the kitchen next month.",
    --   "response_at": "2026-07-25T10:00:00Z",
    --   "is_public": true,
    --   "published_at": "2026-07-22T14:00:00Z",
    --   "ai_sentiment": "mixed_positive",
    --   "ai_action_items": ["update kitchen appliances", "improve cleanliness checklist"]
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_reviews_property ON reviews(property_id);
CREATE INDEX idx_reviews_reservation ON reviews(reservation_id);

CREATE TABLE guidebooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES properties(id),
    title           VARCHAR(255) NOT NULL,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    share_url       TEXT,

    -- JSONB: sections with flexible content types
    sections        JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"type": "house_rules", "title": "House Rules", "icon": "rules",
    --    "body": "No smoking. Quiet hours after 10pm...", "order": 1},
    --   {"type": "wifi", "title": "WiFi", "icon": "wifi",
    --    "body": "Network: BeachHouse5G\nPassword: sunset2026", "order": 2},
    --   {"type": "check_in", "title": "Check-in Instructions", "icon": "key",
    --    "body": "Your door code is {{door_code}}...", "order": 3},
    --   {"type": "local_tips", "title": "Local Recommendations", "icon": "map",
    --    "items": [
    --      {"name": "Joe's Crab Shack", "category": "restaurant", "distance": "0.3mi", "notes": "Best seafood"},
    --      {"name": "Sunset Beach", "category": "beach", "distance": "0.1mi"}
    --    ], "order": 4}
    -- ]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_guidebooks_property ON guidebooks(property_id);
```

## Audit & Sync Logs

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(50) NOT NULL,
    changes         JSONB,                    -- {field: {old: ..., new: ...}}
    metadata        JSONB DEFAULT '{}',       -- ip_address, user_agent, source
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);

CREATE TABLE channel_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id      UUID NOT NULL REFERENCES listings(id),
    direction       VARCHAR(10) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    error_message   TEXT,
    duration_ms     INTEGER,
    -- JSONB: full request/response for debugging
    request_data    JSONB,
    response_data   JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sync_log_listing ON channel_sync_log(listing_id);
CREATE INDEX idx_sync_log_created ON channel_sync_log(created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | organisations, users |
| Property Management | 1 | properties (owners, photos, amenities, compliance all in JSONB) |
| Channel & Listings | 2 | channels, listings |
| Reservations & Guests | 2 | guests, reservations (nightly rates, tax breakdown in JSONB) |
| Calendar & Pricing | 2 | calendar_days, rate_plans |
| Guest Communication | 3 | conversations, messages, message_templates |
| Operations | 1 | tasks (unified cleaning + maintenance with JSONB task_data) |
| Financial Management | 2 | payments, owner_statements (line items in JSONB) |
| Reviews & Guidebooks | 2 | reviews, guidebooks (sections in JSONB) |
| Audit & Sync | 2 | audit_log, channel_sync_log |
| Infrastructure | 1 | jsonb_schemas |
| **Total** | **20** | Half the tables of the normalized model |

---

## Key Design Decisions

1. **JSONB for variable data, typed columns for query-critical fields** — `check_in_date`, `total_amount`, `status`, and `currency_code` are always typed columns because they appear in WHERE clauses, aggregations, and reports. Channel metadata, amenity lists, compliance permits, and nightly rate breakdowns are JSONB because their structure varies by context.

2. **Properties table absorbs 7 tables from the normalized model** — owners, photos, amenities, addresses (partially), compliance permits, and details are all JSONB columns on the `properties` table. This eliminates 6 junction/child tables and their associated joins, dramatically simplifying queries like "give me everything about this property."

3. **Unified `tasks` table replaces separate cleaning and maintenance tables** — the `task_type` column discriminates, and the `task_data` JSONB holds type-specific fields (cleaning checklists vs. vendor info). Adding a new task type (e.g., "inventory restock") requires zero schema changes.

4. **Owner statements with JSONB line items** — statement details (per-reservation breakdowns, expense items, tax summaries) live in a `details` JSONB column. This avoids a separate `owner_statement_items` table and makes statement generation a single row insert.

5. **GIN indexes on JSONB columns** — PostgreSQL GIN indexes enable fast containment queries (`@>`) on JSONB. Finding all properties with a pool, all reservations from Airbnb with instant book, or all tasks with a specific vendor is efficient without denormalization.

6. **`jsonb_schemas` table for documentation and validation** — each JSONB column has a corresponding JSON Schema stored in this table. The application layer validates writes against these schemas, compensating for the lack of database-level type enforcement. This also serves as living documentation for developers.

7. **Channel-specific data preserved in original structure** — the `channel_metadata` JSONB on reservations and `channel_data` JSONB on listings store channel-specific fields in their native format. This preserves round-trip fidelity: data received from Airbnb's API is stored as-is, avoiding lossy mapping to a rigid schema.

8. **20 tables instead of 40+** — the hybrid approach cuts the table count roughly in half compared to a fully normalized model. This means fewer migrations, fewer ORM models, fewer JOIN operations, and faster onboarding for new developers.

9. **Progressive extraction pattern** — if a JSONB field becomes a frequent query target, it can be promoted to a generated column: `ALTER TABLE properties ADD COLUMN has_pool BOOLEAN GENERATED ALWAYS AS ((details->'amenities')::jsonb @> '"pool"') STORED;` — no application changes required.

10. **Same query capabilities, different trade-offs** — the hybrid model can answer all the same questions as the normalized model, but some queries trade compile-time type safety for runtime JSONB path expressions. The team must weigh "fewer migrations and faster iteration" against "the database catches type errors for us."

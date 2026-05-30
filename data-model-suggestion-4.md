# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Vacation Rental Management · Created: 2026-05-22

## Philosophy

This model combines a relational core for transactional operations (reservations, payments, calendar) with a property graph layer for relationship-intensive queries. The graph layer models the rich web of connections that define a vacation rental business: properties are owned by owners who belong to organisations; guests stay at properties and are connected to other guests (travel companions, repeat visitors); cleaners service properties and are rated by guests; listings connect properties to channels; permits link properties to jurisdictions.

The insight driving this architecture is that many of the highest-value queries in vacation rental management are graph traversal problems disguised as SQL queries. "Show me all guests who have stayed at any property managed by this organisation and also stayed at a competitor's property on Airbnb" is a multi-hop relationship traversal. "Find potential conflicts of interest where an owner also has a cleaning contract" is a pattern-matching problem. "What is the shortest path from a guest complaint to a maintenance vendor resolution?" is a path query. These queries are either impossible or prohibitively expensive in a purely relational model but trivial in a graph.

The architecture uses PostgreSQL for the transactional core (reservations, payments, calendar availability) where ACID guarantees matter, and Apache AGE (the open-source graph extension for PostgreSQL) or a dedicated graph database (Neo4j) for the relationship layer. The two layers synchronize via change-data-capture (CDC) or application-level dual-writes.

**Best for:** Organisations managing large portfolios (50+ properties) where guest relationship insights, ownership chain analysis, vendor network optimization, and cross-property intelligence are competitive advantages.

**Trade-offs:**
- Pro: Relationship queries (guest networks, ownership chains, vendor coverage) are orders of magnitude faster than SQL JOINs
- Pro: Pattern matching ("find guests who always book beachfront properties in summer") is native
- Pro: Recommendation engines (similar properties, repeat guest preferences) are natural graph algorithms
- Pro: Fraud detection (connected guest accounts, suspicious booking patterns) leverages graph connectivity
- Pro: AI agents can traverse the graph to answer complex natural-language questions about the business
- Con: Two storage systems to maintain (relational + graph) increases operational complexity
- Con: Dual-write consistency requires careful engineering (CDC or transactional outbox pattern)
- Con: Graph databases have a steeper learning curve (Cypher/openCypher query language)
- Con: Smaller ecosystem of tooling, ORMs, and migration frameworks compared to pure relational
- Con: Overkill for small operators (1-10 properties) who don't need relationship intelligence

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 3166-1 alpha-2 | Country codes on address nodes and jurisdiction vertices |
| ISO 3166-2 | Subdivision codes on jurisdiction vertices in the graph |
| ISO 4217 | Currency codes on all monetary properties in both relational and graph layers |
| RFC 5545 (iCalendar) | Calendar export from relational `calendar_days` table |
| OpenTravel Alliance (OTA) | Reservation entities align with OTA HotelRes message semantics |
| openCypher | Graph query language specification used by Apache AGE and Neo4j |
| Apache TinkerPop / Gremlin | Alternative graph traversal API (supported if using JanusGraph or AWS Neptune) |
| W3C RDF/OWL | Graph ontology can be exported as RDF for semantic web interoperability |
| GDPR (EU 2016/679) | Graph vertex deletion cascades to connected edges; relational layer supports soft-delete |

---

## Relational Core (PostgreSQL) — Transactional Operations

The relational layer handles all write-heavy, ACID-critical operations.

### Identity & Configuration

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'free',
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    default_timezone VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
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
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);
CREATE INDEX idx_users_org ON users(organisation_id);
CREATE INDEX idx_users_email ON users(email);
```

### Reservations (Transactional Core)

```sql
CREATE TABLE reservations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    property_id     UUID NOT NULL,           -- references graph Property vertex
    guest_id        UUID NOT NULL,           -- references graph Guest vertex
    channel_code    VARCHAR(50),
    external_id     VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    check_in_date   DATE NOT NULL,
    check_out_date  DATE NOT NULL,
    nights          SMALLINT GENERATED ALWAYS AS (check_out_date - check_in_date) STORED,
    guests_count    SMALLINT NOT NULL DEFAULT 1,
    adults          SMALLINT NOT NULL DEFAULT 1,
    children        SMALLINT NOT NULL DEFAULT 0,
    nightly_rate    NUMERIC(12,2) NOT NULL,
    subtotal        NUMERIC(12,2) NOT NULL,
    cleaning_fee    NUMERIC(12,2) NOT NULL DEFAULT 0,
    service_fee     NUMERIC(12,2) NOT NULL DEFAULT 0,
    tax_total       NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(12,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    payout_amount   NUMERIC(12,2),
    special_requests TEXT,
    cancellation_policy VARCHAR(50),
    booking_details JSONB NOT NULL DEFAULT '{}',
    channel_metadata JSONB NOT NULL DEFAULT '{}',
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
```

### Calendar & Pricing

```sql
CREATE TABLE calendar_days (
    property_id     UUID NOT NULL,
    date            DATE NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'available',
    reservation_id  UUID REFERENCES reservations(id),
    base_rate       NUMERIC(12,2),
    adjusted_rate   NUMERIC(12,2),
    min_stay        SMALLINT,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    pricing_data    JSONB NOT NULL DEFAULT '{}',
    note            VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (property_id, date)
);
CREATE INDEX idx_calendar_status ON calendar_days(status);
CREATE INDEX idx_calendar_date ON calendar_days(date);

CREATE TABLE rate_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL,
    name            VARCHAR(100) NOT NULL,
    rate_type       VARCHAR(50) NOT NULL,
    nightly_rate    NUMERIC(12,2) NOT NULL,
    min_stay        SMALLINT NOT NULL DEFAULT 1,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    start_date      DATE,
    end_date        DATE,
    priority        SMALLINT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    rules           JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rate_plans_property ON rate_plans(property_id);
```

### Payments

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
    gateway_data    JSONB NOT NULL DEFAULT '{}',
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payments_org ON payments(organisation_id);
CREATE INDEX idx_payments_reservation ON payments(reservation_id);
CREATE INDEX idx_payments_status ON payments(status);
```

### Guest Communication

```sql
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    reservation_id  UUID REFERENCES reservations(id),
    guest_id        UUID NOT NULL,
    channel_code    VARCHAR(50),
    subject         VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
    last_message_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_conversations_org ON conversations(organisation_id);
CREATE INDEX idx_conversations_reservation ON conversations(reservation_id);

CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id),
    sender_type     VARCHAR(20) NOT NULL,
    sender_id       UUID,
    channel         VARCHAR(50) NOT NULL,
    body            TEXT NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_messages_conversation ON messages(conversation_id);
CREATE INDEX idx_messages_created ON messages(created_at);
```

### Audit & Sync

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(50) NOT NULL,
    changes         JSONB,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);

CREATE TABLE channel_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id      UUID NOT NULL,
    direction       VARCHAR(10) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    error_message   TEXT,
    duration_ms     INTEGER,
    request_data    JSONB,
    response_data   JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sync_log_listing ON channel_sync_log(listing_id);
CREATE INDEX idx_sync_log_created ON channel_sync_log(created_at);
```

---

## Graph Layer — Relationship Intelligence

The graph layer stores entities as vertices and relationships as edges. Each vertex and edge has a label (type) and properties (key-value pairs). This layer is the source of truth for entity attributes and relationships; the relational layer references entity IDs from the graph.

### Implementation Options

| Option | Technology | Notes |
|--------|-----------|-------|
| **Option A** | Apache AGE (PostgreSQL extension) | Same database, Cypher queries via `ag_catalog`, zero operational overhead |
| **Option B** | Neo4j (dedicated graph database) | Mature, rich tooling, requires CDC sync from PostgreSQL |
| **Option C** | Amazon Neptune | Managed graph DB, supports both Gremlin and openCypher |

The schemas below use openCypher syntax (compatible with Apache AGE and Neo4j).

### Vertex Types (Nodes)

```cypher
// Organisation
CREATE (o:Organisation {
  id: 'uuid',
  name: 'Coastal Rentals LLC',
  slug: 'coastal-rentals',
  subscription_tier: 'professional',
  default_currency: 'USD',
  created_at: datetime('2024-01-15T00:00:00Z')
})

// Property
CREATE (p:Property {
  id: 'uuid',
  name: 'Oceanview Beach House',
  property_type: 'house',
  status: 'active',
  bedrooms: 3,
  bathrooms: 2.0,
  max_guests: 8,
  check_in_time: '15:00',
  check_out_time: '11:00',
  min_stay_nights: 2,
  timezone: 'America/Chicago',
  address_line_1: '123 Coastal Drive',
  city: 'Galveston',
  state_province: 'Texas',
  postal_code: '77550',
  country_code: 'US',
  latitude: 29.3013,
  longitude: -94.7977,
  area_sqft: 2200,
  description: 'Spacious beachfront home with panoramic ocean views...'
})

// Owner
CREATE (ow:Owner {
  id: 'uuid',
  full_name: 'Jane Smith',
  email: 'jane@example.com',
  phone: '+1-512-555-0100',
  tax_id: 'XX-XXXXXXX',
  payout_method: 'bank_transfer',
  commission_pct: 20.0
})

// Guest
CREATE (g:Guest {
  id: 'uuid',
  first_name: 'John',
  last_name: 'Doe',
  email: 'john.doe@email.com',
  phone: '+1-415-555-0200',
  nationality: 'US',
  preferred_language: 'en',
  id_verified: true,
  created_at: datetime('2025-06-01T00:00:00Z')
})

// Channel
CREATE (c:Channel {
  id: 'uuid',
  code: 'airbnb',
  name: 'Airbnb',
  api_type: 'rest'
})

// Listing (a property's presence on a channel)
CREATE (l:Listing {
  id: 'uuid',
  external_id: '12345678',
  status: 'active',
  title: 'Oceanview Beach House - Beachfront Paradise',
  listing_url: 'https://airbnb.com/rooms/12345678',
  sync_enabled: true,
  last_synced_at: datetime('2026-05-22T10:00:00Z')
})

// Amenity
CREATE (a:Amenity {
  id: 'uuid',
  name: 'pool',
  category: 'features',
  icon: 'pool'
})

// Cleaner (a user who performs cleaning tasks)
CREATE (cl:Cleaner {
  id: 'uuid',
  full_name: 'Maria Garcia',
  email: 'maria@cleaning.com',
  phone: '+1-512-555-0300',
  hourly_rate: 35.00,
  currency: 'USD',
  rating: 4.8,
  total_cleanings: 156
})

// Vendor (maintenance service provider)
CREATE (v:Vendor {
  id: 'uuid',
  company_name: 'ABC Plumbing',
  contact_name: 'Bob Johnson',
  phone: '+1-512-555-0400',
  email: 'info@abcplumbing.com',
  specialty: 'plumbing',
  rating: 4.5,
  is_preferred: true
})

// Jurisdiction
CREATE (j:Jurisdiction {
  id: 'uuid',
  name: 'City of Galveston',
  level: 'city',
  country_code: 'US',
  subdivision_code: 'US-TX',
  city_name: 'Galveston',
  requires_permit: true,
  max_nights_per_year: null,
  occupancy_tax_rate: 0.09
})

// Permit
CREATE (pm:Permit {
  id: 'uuid',
  permit_type: 'str_licence',
  permit_number: 'STR-2024-12345',
  status: 'active',
  issued_date: date('2024-01-15'),
  expiry_date: date('2025-01-15')
})
```

### Edge Types (Relationships)

```cypher
// Organisation structure
(o:Organisation)-[:MANAGES]->(p:Property)
(u:User)-[:BELONGS_TO {role: 'admin'}]->(o:Organisation)

// Property ownership (with temporal and financial properties)
(ow:Owner)-[:OWNS {
  ownership_pct: 60.0,
  effective_from: date('2020-01-01'),
  effective_to: null
}]->(p:Property)

// Property amenities
(p:Property)-[:HAS_AMENITY]->(a:Amenity)

// Channel listings
(p:Property)-[:LISTED_ON {
  listing_id: 'uuid',
  external_id: '12345678',
  status: 'active'
}]->(c:Channel)

// Guest stays (created from reservation data)
(g:Guest)-[:STAYED_AT {
  reservation_id: 'uuid',
  check_in: date('2026-07-15'),
  check_out: date('2026-07-20'),
  nights: 5,
  total_amount: 1662.50,
  currency: 'USD',
  rating_given: 4.5,
  channel: 'airbnb'
}]->(p:Property)

// Guest relationships
(g1:Guest)-[:TRAVELED_WITH {
  reservation_id: 'uuid',
  relationship: 'family'
}]->(g2:Guest)

// Guest preferences (inferred from booking history)
(g:Guest)-[:PREFERS {
  strength: 0.85,
  inferred_from: 'booking_history'
}]->(a:Amenity)

// Cleaning assignments
(cl:Cleaner)-[:CLEANS {
  frequency: 'regular',
  since: date('2024-03-01')
}]->(p:Property)

(cl:Cleaner)-[:COMPLETED_CLEANING {
  task_id: 'uuid',
  date: date('2026-07-20'),
  duration_minutes: 95,
  rating: 4,
  reservation_id: 'uuid'
}]->(p:Property)

// Vendor services
(v:Vendor)-[:SERVICES {
  service_type: 'plumbing',
  contract_since: date('2023-06-01')
}]->(p:Property)

(v:Vendor)-[:COMPLETED_REPAIR {
  task_id: 'uuid',
  date: date('2026-05-10'),
  cost: 425.00,
  currency: 'USD',
  category: 'plumbing'
}]->(p:Property)

// Compliance
(p:Property)-[:LOCATED_IN]->(j:Jurisdiction)
(j:Jurisdiction)-[:CHILD_OF]->(parent_j:Jurisdiction)
(p:Property)-[:HOLDS_PERMIT]->(pm:Permit)
(pm:Permit)-[:ISSUED_BY]->(j:Jurisdiction)

// Review relationships
(g:Guest)-[:REVIEWED {
  reservation_id: 'uuid',
  overall_rating: 4.5,
  body: 'Great location...',
  date: date('2026-07-22')
}]->(p:Property)

// Similar properties (computed by ML)
(p1:Property)-[:SIMILAR_TO {
  similarity_score: 0.87,
  shared_amenities: ['pool', 'beachfront', 'wifi'],
  computed_at: datetime('2026-05-22T00:00:00Z')
}]->(p2:Property)
```

---

## Graph Query Examples

### 1. Repeat Guest Detection

```cypher
// Find all guests who have stayed at more than one property in the organisation
MATCH (g:Guest)-[s:STAYED_AT]->(p:Property)<-[:MANAGES]-(o:Organisation {id: $org_id})
WITH g, count(DISTINCT p) AS property_count, collect(p.name) AS properties
WHERE property_count > 1
RETURN g.first_name, g.last_name, g.email, property_count, properties
ORDER BY property_count DESC
```

### 2. Guest Preference Analysis for Recommendations

```cypher
// Given a guest, find properties they haven't stayed at but that match their preferences
MATCH (g:Guest {id: $guest_id})-[:PREFERS]->(a:Amenity)<-[:HAS_AMENITY]-(p:Property)
WHERE NOT (g)-[:STAYED_AT]->(p)
  AND p.status = 'active'
WITH p, count(a) AS matching_amenities, collect(a.name) AS amenities
ORDER BY matching_amenities DESC
LIMIT 5
RETURN p.name, p.city, p.bedrooms, matching_amenities, amenities
```

### 3. Cleaner Workload and Coverage Analysis

```cypher
// Find cleaners who service properties without enough coverage
MATCH (p:Property)<-[:MANAGES]-(o:Organisation {id: $org_id})
OPTIONAL MATCH (cl:Cleaner)-[:CLEANS]->(p)
WITH p, collect(cl) AS cleaners, size(collect(cl)) AS cleaner_count
WHERE cleaner_count < 2
RETURN p.name, p.city, cleaner_count,
       [c IN cleaners | c.full_name] AS assigned_cleaners
ORDER BY cleaner_count ASC
```

### 4. Ownership Chain and Financial Exposure

```cypher
// Find all owners, their properties, and total portfolio value
MATCH (ow:Owner)-[owns:OWNS]->(p:Property)<-[:MANAGES]-(o:Organisation {id: $org_id})
OPTIONAL MATCH (g:Guest)-[s:STAYED_AT]->(p)
WHERE s.check_in >= date('2026-01-01')
WITH ow, p, owns.ownership_pct AS pct,
     sum(s.total_amount) AS gross_revenue_2026
RETURN ow.full_name, ow.email,
       collect({
         property: p.name,
         ownership_pct: pct,
         gross_revenue: gross_revenue_2026,
         owner_share: gross_revenue_2026 * pct / 100
       }) AS portfolio
ORDER BY ow.full_name
```

### 5. Fraud Detection — Connected Guest Accounts

```cypher
// Find guests who share email domains or phone numbers with other guests
// who have had chargebacks or negative reviews
MATCH (g1:Guest)-[s1:STAYED_AT]->(p:Property)
MATCH (g2:Guest)-[s2:STAYED_AT]->(p2:Property)
WHERE g1 <> g2
  AND (g1.email ENDS WITH split(g2.email, '@')[1]
       OR g1.phone = g2.phone)
  AND (s2.rating_given IS NOT NULL AND s2.rating_given < 2.0)
RETURN g1.first_name + ' ' + g1.last_name AS flagged_guest,
       g1.email AS flagged_email,
       g2.first_name + ' ' + g2.last_name AS connected_to,
       g2.email AS connected_email,
       'shared_domain_or_phone' AS connection_type
```

### 6. Jurisdiction Compliance Roll-up

```cypher
// Find all properties in a jurisdiction hierarchy and their permit status
MATCH path = (j:Jurisdiction {name: 'Texas'})<-[:CHILD_OF*0..3]-(child:Jurisdiction)
MATCH (p:Property)-[:LOCATED_IN]->(child)
OPTIONAL MATCH (p)-[:HOLDS_PERMIT]->(pm:Permit)
RETURN child.name AS jurisdiction,
       child.level AS level,
       p.name AS property,
       pm.permit_number AS permit,
       pm.status AS permit_status,
       pm.expiry_date AS expires
ORDER BY child.level, child.name
```

### 7. Similar Property Discovery for Pricing Intelligence

```cypher
// Find properties similar to a target and compare their pricing
MATCH (target:Property {id: $property_id})-[sim:SIMILAR_TO]->(comp:Property)
WHERE sim.similarity_score > 0.7
RETURN comp.name, comp.city, comp.bedrooms, comp.max_guests,
       sim.similarity_score, sim.shared_amenities
ORDER BY sim.similarity_score DESC
LIMIT 10
```

### 8. Vendor Network Optimization

```cypher
// Find the best vendor for a specific service type near a property
MATCH (p:Property {id: $property_id})-[:LOCATED_IN]->(j:Jurisdiction)
MATCH (v:Vendor)-[:SERVICES]->(nearby:Property)-[:LOCATED_IN]->(j)
WHERE v.specialty = $service_type
WITH v, count(nearby) AS properties_serviced,
     avg(v.rating) AS avg_rating
RETURN v.company_name, v.contact_name, v.phone,
       properties_serviced, avg_rating
ORDER BY avg_rating DESC, properties_serviced DESC
```

---

## Synchronization Between Layers

```sql
-- Outbox table for graph sync (transactional outbox pattern)
-- Application writes to this table within the same transaction as the relational write.
-- A background worker reads from this table and applies changes to the graph layer.
CREATE TABLE graph_sync_outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation       VARCHAR(20) NOT NULL,   -- create_vertex, update_vertex, delete_vertex,
                                            -- create_edge, update_edge, delete_edge
    vertex_label    VARCHAR(50),
    edge_label      VARCHAR(50),
    source_id       UUID,                   -- for edges: source vertex ID
    target_id       UUID,                   -- for edges: target vertex ID
    properties      JSONB NOT NULL,         -- vertex/edge properties to set
    processed       BOOLEAN NOT NULL DEFAULT false,
    processed_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_outbox_unprocessed ON graph_sync_outbox(processed, created_at)
    WHERE processed = false;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| **Relational Layer** | | |
| Identity & Configuration | 2 | organisations, users |
| Reservations | 1 | reservations |
| Calendar & Pricing | 2 | calendar_days, rate_plans |
| Payments | 1 | payments |
| Guest Communication | 2 | conversations, messages |
| Audit & Sync | 2 | audit_log, channel_sync_log |
| Graph Sync | 1 | graph_sync_outbox |
| **Relational Subtotal** | **11** | |
| **Graph Layer** | | |
| Vertex Types | 10 | Organisation, Property, Owner, Guest, Channel, Listing, Amenity, Cleaner, Vendor, Jurisdiction, Permit |
| Edge Types | 15+ | MANAGES, OWNS, LISTED_ON, STAYED_AT, TRAVELED_WITH, PREFERS, CLEANS, SERVICES, LOCATED_IN, HOLDS_PERMIT, REVIEWED, SIMILAR_TO, etc. |
| **Total** | **11 tables + graph schema** | |

---

## Key Design Decisions

1. **Clear separation of concerns** — the relational layer handles transactional operations where ACID matters (booking a reservation, processing a payment, updating calendar availability). The graph layer handles relationship queries where traversal performance matters (guest networks, ownership chains, vendor coverage, property recommendations).

2. **Graph as the entity source of truth** — Property, Guest, Owner, Cleaner, Vendor, and Amenity details live primarily in the graph. The relational layer references them by UUID but does not duplicate their attributes. This avoids dual-maintenance of entity data and makes the graph the single place to query "everything about this property."

3. **Relational for time-series data** — Calendar availability (one row per property per date) and rate plans are inherently tabular and benefit from B-tree indexes and range queries. These stay relational. Trying to model daily availability as graph edges would be both unnatural and inefficient.

4. **Transactional outbox for sync** — the `graph_sync_outbox` table ensures that relational writes and graph updates are eventually consistent. When a reservation is created, the application inserts the reservation row AND an outbox entry in the same transaction. A background worker processes outbox entries and creates the corresponding graph edges (STAYED_AT, REVIEWED, etc.).

5. **Temporal edges for historical analysis** — graph edges like STAYED_AT and COMPLETED_CLEANING carry date properties, enabling time-windowed traversals: "find all guests who stayed in the last 6 months" without scanning the full graph.

6. **Inferred relationship edges** — the PREFERS and SIMILAR_TO edges are computed by ML models and stored in the graph. These enable real-time recommendation queries without running ML inference at query time. The `computed_at` timestamp indicates freshness.

7. **Jurisdiction hierarchy as a graph** — the CHILD_OF edges between Jurisdiction vertices create a natural tree that can be traversed to answer "what regulations apply to this property?" by walking up from city to county to state to country. This replaces recursive CTEs in the relational model.

8. **Apache AGE for single-database deployment** — for teams that want the graph capability without a separate database, Apache AGE runs as a PostgreSQL extension. The same PostgreSQL instance serves both relational queries and Cypher graph queries, simplifying deployment for smaller operators.

9. **Neo4j for enterprise-scale deployments** — organisations with 200+ properties, thousands of guests, and complex vendor networks benefit from Neo4j's dedicated graph engine, visual graph explorer (Neo4j Bloom), and graph data science library for centrality, community detection, and similarity algorithms.

10. **AI agent traversal** — an AI agent (via MCP server) can traverse the graph to answer natural-language questions: "Which cleaners have the highest ratings for beachfront properties in Texas?" becomes a Cypher query that the agent constructs and executes. The graph structure makes these queries composable and fast, while a relational model would require the agent to construct multi-table JOINs.

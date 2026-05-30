# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Hotel Property Management System · Created: 2026-05-22

## Philosophy

This model uses PostgreSQL relational tables for core entities with stable, well-understood structures (reservations, rooms, folios, payments), but delegates variable, jurisdiction-specific, and rapidly evolving data to JSONB columns. The result is a leaner schema than a fully normalised model -- fewer tables, fewer migrations, faster time to MVP -- while retaining transactional integrity and relational query power for the fields that matter most.

The key insight is that hotel PMS data has two distinct zones. Zone 1 is universal: every hotel on earth has rooms, reservations, guests, rates, charges, and payments, and these entities share a common structure regardless of geography. Zone 2 is variable: tax rules differ by country and municipality, guest registration requirements vary by jurisdiction (some require passport scans, others require only a name), property amenity configurations are unique, and AI-driven features produce schema-fluid output (demand signals, pricing recommendations). JSONB handles Zone 2 without requiring schema migrations for every new country, tax regime, or AI model version.

This approach is used in production by platforms like Apaleo (which uses a lean core with extensible configuration) and by modern SaaS platforms that need to serve diverse markets from a single codebase. It is the natural fit for an independent-hotel PMS that must deploy globally without per-country schema forks.

**Best for:** Teams building a multi-market PMS that must ship fast, support diverse jurisdictions, and evolve its feature set rapidly without heavy migration overhead.

**Trade-offs:**
- (+) Fewer tables (~25-30) means simpler schema, fewer joins, faster queries
- (+) Jurisdiction-specific fields (tax rules, registration requirements, invoice formats) live in JSONB without schema changes
- (+) AI feature output (pricing signals, recommendations, forecasts) stored in JSONB without rigid column definitions
- (+) Faster MVP: add new fields by updating JSONB structure, not running ALTER TABLE
- (+) PostgreSQL JSONB has mature indexing (GIN, expression indexes) and query operators
- (-) JSONB fields lack foreign key constraints -- referential integrity is application-enforced
- (-) JSONB schema drift is a risk without validation (mitigated by JSON Schema or CHECK constraints)
- (-) Complex JSONB queries can be slower than indexed relational columns without careful index design
- (-) BI tools and report generators may struggle with nested JSONB structures
- (-) Type safety for JSONB contents depends on application-layer validation

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenTravel Alliance (OTA) | Core relational fields (reservation, rate plan, room type) align with OTA 2.0 object model; OTA-specific extensions stored in JSONB |
| ISO 3166-1/2 | Country and subdivision codes as relational CHAR columns on property and guest tables |
| ISO 4217 | Currency codes as relational CHAR(3) columns on all monetary fields |
| PCI DSS v4.0 | Payment data uses relational columns with tokenised references; no JSONB for sensitive card data |
| GDPR/CCPA | Guest PII in relational columns for controlled access; consent tracking in dedicated table; JSONB preferences are non-PII |
| JSON Schema (Draft 2020-12) | Application-layer validation of JSONB column structures using JSON Schema definitions |
| PostgreSQL 18 Temporal Constraints | `WITHOUT OVERLAPS` on reservation date ranges to prevent double-booking at the database level |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    billing_email   VARCHAR(255),
    subscription_tier VARCHAR(50) DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings JSONB:
    -- {
    --   "default_currency": "EUR",
    --   "date_format": "DD/MM/YYYY",
    --   "fiscal_year_start_month": 1,
    --   "features": {"ai_pricing": true, "guest_messaging": true}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE property (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(200) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    star_rating     SMALLINT CHECK (star_rating BETWEEN 1 AND 5),
    country_code    CHAR(2) NOT NULL,                     -- ISO 3166-1
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',      -- ISO 4217

    -- JSONB: property details that vary by type and jurisdiction
    address         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "line1": "123 Beach Road",
    --   "line2": "Suite 4",
    --   "city": "Miami",
    --   "state": "FL",
    --   "postal_code": "33139",
    --   "country": "US",
    --   "latitude": 25.7617,
    --   "longitude": -80.1918
    -- }

    contact         JSONB NOT NULL DEFAULT '{}',
    -- {"phone": "+1-305-555-0100", "email": "info@hotel.com", "website": "https://hotel.com"}

    -- Jurisdiction-specific configuration
    tax_config      JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"name": "State Sales Tax", "rate_pct": 6.0, "applies_to": ["ROOM","OTHER"], "inclusive": false},
    --   {"name": "Tourist Development Tax", "rate_pct": 6.0, "applies_to": ["ROOM"], "inclusive": false},
    --   {"name": "City Resort Tax", "rate_pct": 2.0, "applies_to": ["ROOM"], "inclusive": false}
    -- ]

    registration_config JSONB NOT NULL DEFAULT '{}',
    -- Jurisdiction-specific guest registration requirements:
    -- {
    --   "require_id_scan": true,
    --   "require_nationality": true,
    --   "require_date_of_birth": false,
    --   "require_next_destination": true,       -- required in Italy
    --   "police_reporting_enabled": true,        -- Alloggiati Web (Italy)
    --   "tourist_tax_per_night_cents": 350,      -- city tax (per person per night)
    --   "tourist_tax_max_nights": 10
    -- }

    operational_config JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "check_in_time": "15:00",
    --   "check_out_time": "11:00",
    --   "overbooking_pct": 5,
    --   "auto_assign_rooms": true,
    --   "housekeeping_inspection_required": true,
    --   "night_audit_time": "03:00"
    -- }

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE INDEX idx_property_tenant ON property(tenant_id);
```

---

## Room Inventory

```sql
CREATE TABLE room_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    code            VARCHAR(20) NOT NULL,
    name            VARCHAR(100) NOT NULL,
    max_occupancy   SMALLINT NOT NULL DEFAULT 2,
    max_adults      SMALLINT NOT NULL DEFAULT 2,
    max_children    SMALLINT NOT NULL DEFAULT 0,
    sort_order      SMALLINT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: room type details that vary by property
    details         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "description": "Spacious room with king bed and city views",
    --   "area_sqm": 35,
    --   "bed_type": "king",
    --   "bed_count": 1,
    --   "view_types": ["city", "pool"],
    --   "amenities": ["wifi", "minibar", "safe", "coffee_maker", "balcony"],
    --   "images": [
    --     {"url": "/img/room/kdlx-1.jpg", "caption": "Room overview", "sort": 1},
    --     {"url": "/img/room/kdlx-2.jpg", "caption": "Bathroom", "sort": 2}
    --   ],
    --   "ota_mapping": {
    --     "booking_com_room_type": "double_room",
    --     "expedia_room_type": "standard_king"
    --   }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, code)
);

CREATE TABLE room (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    room_type_id    UUID NOT NULL REFERENCES room_type(id),
    room_number     VARCHAR(10) NOT NULL,
    floor           VARCHAR(10),
    status          VARCHAR(20) NOT NULL DEFAULT 'AVAILABLE'
                    CHECK (status IN ('AVAILABLE','OCCUPIED','OUT_OF_ORDER',
                                      'OUT_OF_SERVICE','BLOCKED')),
    hk_status       VARCHAR(20) NOT NULL DEFAULT 'CLEAN'
                    CHECK (hk_status IN ('CLEAN','DIRTY','INSPECTED',
                                          'IN_PROGRESS','OUT_OF_ORDER')),

    -- JSONB: room-specific attributes
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "is_accessible": true,
    --   "is_smoking": false,
    --   "is_connecting": true,
    --   "connecting_to": "405",
    --   "features": ["sea_view", "balcony", "rollaway_available"],
    --   "maintenance_notes": "AC serviced 2026-04-15"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, room_number)
);

CREATE INDEX idx_room_property_type ON room(property_id, room_type_id);
CREATE INDEX idx_room_status ON room(property_id, status, hk_status);
```

---

## Guest Profiles

```sql
CREATE TABLE guest (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    -- Core PII in relational columns (queryable, indexable, GDPR-trackable)
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(30),
    nationality     CHAR(2),                              -- ISO 3166-1
    language        VARCHAR(10) DEFAULT 'en',             -- BCP 47
    vip_level       SMALLINT DEFAULT 0,
    company_name    VARCHAR(200),

    -- JSONB: extended profile data that varies by guest and jurisdiction
    profile         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "date_of_birth": "1985-03-15",
    --   "gender": "male",
    --   "id_documents": [
    --     {"type": "passport", "number": "AB1234567", "country": "GB",
    --      "expiry": "2030-01-01"}
    --   ],
    --   "address": {
    --     "line1": "10 Downing St", "city": "London",
    --     "postal_code": "SW1A 2AA", "country": "GB"
    --   },
    --   "loyalty": {
    --     "program_id": "GOLD-2026-001",
    --     "tier": "gold",
    --     "points_balance": 15420
    --   },
    --   "company": {
    --     "id": "uuid...",
    --     "name": "Acme Corp",
    --     "tax_id": "GB123456789"
    --   }
    -- }

    -- JSONB: guest preferences (non-PII, safe for AI processing)
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "room": {"floor": "high", "bed": "king", "view": "sea", "quiet": true},
    --   "pillow": "firm",
    --   "minibar": {"pre_stock": ["sparkling_water", "white_wine"]},
    --   "newspaper": "Financial Times",
    --   "dietary": "gluten_free",
    --   "communication": {"preferred_channel": "whatsapp", "quiet_hours": "22:00-08:00"}
    -- }

    -- Aggregated stats (updated by triggers or application)
    total_stays     INT NOT NULL DEFAULT 0,
    total_revenue_cents BIGINT NOT NULL DEFAULT 0,
    last_stay_date  DATE,

    -- GDPR
    gdpr_consent    BOOLEAN NOT NULL DEFAULT false,
    gdpr_consent_date TIMESTAMPTZ,
    data_retention_until DATE,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_guest_tenant ON guest(tenant_id);
CREATE INDEX idx_guest_email ON guest(tenant_id, email);
CREATE INDEX idx_guest_name ON guest(tenant_id, last_name, first_name);

-- GIN index on preferences for AI queries
-- e.g. "find all guests who prefer high floors"
CREATE INDEX idx_guest_preferences ON guest USING GIN (preferences jsonb_path_ops);

-- Expression index on loyalty tier from profile JSONB
CREATE INDEX idx_guest_loyalty_tier ON guest ((profile->>'loyalty'->>'tier'))
    WHERE profile->'loyalty' IS NOT NULL;
```

---

## Rate Plans & Pricing

```sql
CREATE TABLE rate_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    code            VARCHAR(30) NOT NULL,
    name            VARCHAR(200) NOT NULL,
    rate_type       VARCHAR(20) NOT NULL DEFAULT 'PUBLIC'
                    CHECK (rate_type IN ('PUBLIC','NEGOTIATED','PACKAGE',
                                         'PROMOTIONAL','MEMBER','WHOLESALE')),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    valid_from      DATE,
    valid_to        DATE,

    -- JSONB: rate plan rules and terms
    rules           JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "min_stay": 1,
    --   "max_stay": 30,
    --   "min_advance_days": 0,
    --   "max_advance_days": 365,
    --   "meal_plan": "BB",
    --   "is_refundable": true,
    --   "cancellation": {
    --     "deadline_hours": 48,
    --     "penalty": "first_night"
    --   },
    --   "applicable_room_types": ["KDLX", "QSTD", "STE"],
    --   "channel_restrictions": ["DIRECT", "BOOKING_COM"],
    --   "inclusions": ["breakfast", "parking", "wifi"],
    --   "derived_from": {
    --     "base_rate_plan": "BAR",
    --     "discount_pct": 15
    --   }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, code)
);

-- Daily rate amounts (ARI table -- lean, high-volume)
CREATE TABLE rate (
    rate_plan_id    UUID NOT NULL REFERENCES rate_plan(id),
    room_type_id    UUID NOT NULL REFERENCES room_type(id),
    stay_date       DATE NOT NULL,
    amount_cents    BIGINT NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    source          VARCHAR(20) NOT NULL DEFAULT 'manual'
                    CHECK (source IN ('manual','ai_engine','import','override')),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (rate_plan_id, room_type_id, stay_date)
);

-- Daily availability with restrictions
CREATE TABLE availability (
    property_id     UUID NOT NULL REFERENCES property(id),
    room_type_id    UUID NOT NULL REFERENCES room_type(id),
    stay_date       DATE NOT NULL,
    total_inventory SMALLINT NOT NULL,
    sold_count      SMALLINT NOT NULL DEFAULT 0,
    blocked_count   SMALLINT NOT NULL DEFAULT 0,

    -- JSONB: restrictions that vary by channel and date
    restrictions    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "is_closed": false,
    --   "min_stay": 2,
    --   "max_stay": null,
    --   "cta": false,
    --   "ctd": false,
    --   "stop_sell_channels": ["EXPEDIA"]
    -- }

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (property_id, room_type_id, stay_date)
);
```

---

## Reservations

```sql
CREATE TABLE reservation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    confirmation_number VARCHAR(20) NOT NULL,
    guest_id        UUID NOT NULL REFERENCES guest(id),
    room_type_id    UUID NOT NULL REFERENCES room_type(id),
    room_id         UUID REFERENCES room(id),
    rate_plan_id    UUID NOT NULL REFERENCES rate_plan(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'CONFIRMED'
                    CHECK (status IN ('PENDING','CONFIRMED','CHECKED_IN','CHECKED_OUT',
                                      'CANCELLED','NO_SHOW','WAITLISTED')),
    source          VARCHAR(30) NOT NULL DEFAULT 'DIRECT'
                    CHECK (source IN ('DIRECT','OTA','GDS','PHONE','WALK_IN',
                                      'CORPORATE','TRAVEL_AGENT','GROUP')),

    -- Core dates and counts (relational for querying)
    check_in_date   DATE NOT NULL,
    check_out_date  DATE NOT NULL,
    adults          SMALLINT NOT NULL DEFAULT 1,
    children        SMALLINT NOT NULL DEFAULT 0,

    -- Financials
    total_amount_cents BIGINT,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',

    -- Timestamps
    checked_in_at   TIMESTAMPTZ,
    checked_out_at  TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,

    -- JSONB: everything else that varies by source, channel, and booking type
    details         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "channel": {
    --     "code": "BOOKING_COM",
    --     "reservation_id": "BDC-9876543",
    --     "commission_pct": 15.0
    --   },
    --   "company": {
    --     "id": "uuid...",
    --     "name": "Acme Corp",
    --     "booking_reference": "PO-2026-001"
    --   },
    --   "travel_agent": {
    --     "id": "uuid...",
    --     "iata": "12345678",
    --     "commission_pct": 10.0
    --   },
    --   "group": {
    --     "id": "uuid...",
    --     "name": "Smith Wedding",
    --     "block_code": "GRP-2026-005"
    --   },
    --   "special_requests": "High floor, extra pillows, late check-in",
    --   "arrival_time": "18:00",
    --   "companions": [
    --     {"guest_id": "uuid...", "first_name": "Jane", "last_name": "Doe"}
    --   ],
    --   "daily_rates": [
    --     {"date": "2026-06-15", "amount_cents": 25000, "room_id": null},
    --     {"date": "2026-06-16", "amount_cents": 25000, "room_id": null},
    --     {"date": "2026-06-17", "amount_cents": 28000, "room_id": null}
    --   ],
    --   "cancellation_reason": null,
    --   "no_show_charge_applied": false
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, confirmation_number),
    CHECK (check_out_date > check_in_date)
);

CREATE INDEX idx_res_property_dates ON reservation(property_id, check_in_date, check_out_date);
CREATE INDEX idx_res_guest ON reservation(guest_id);
CREATE INDEX idx_res_status ON reservation(property_id, status);
CREATE INDEX idx_res_room ON reservation(room_id) WHERE room_id IS NOT NULL;

-- GIN index on details for channel-specific queries
CREATE INDEX idx_res_details ON reservation USING GIN (details jsonb_path_ops);
```

---

## Billing & Folios

```sql
CREATE TABLE folio (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    reservation_id  UUID REFERENCES reservation(id),
    guest_id        UUID NOT NULL REFERENCES guest(id),
    folio_number    VARCHAR(20) NOT NULL,
    folio_type      VARCHAR(20) NOT NULL DEFAULT 'GUEST'
                    CHECK (folio_type IN ('GUEST','MASTER','CITY_LEDGER',
                                          'ADVANCE_DEPOSIT','HOUSE')),
    status          VARCHAR(20) NOT NULL DEFAULT 'OPEN'
                    CHECK (status IN ('OPEN','SETTLED','TRANSFERRED','VOID')),
    balance_cents   BIGINT NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',

    -- JSONB: billing details that vary by payment arrangement
    billing_info    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "company_name": "Acme Corp",
    --   "company_tax_id": "US-12-3456789",
    --   "billing_address": {...},
    --   "payment_terms": "net_30",
    --   "purchase_order": "PO-2026-001",
    --   "routing_rules": [
    --     {"category": "ROOM", "target_folio": "COMPANY"},
    --     {"category": "FNB",  "target_folio": "GUEST"}
    --   ]
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, folio_number)
);

CREATE TABLE folio_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    folio_id        UUID NOT NULL REFERENCES folio(id),
    line_type       VARCHAR(10) NOT NULL
                    CHECK (line_type IN ('CHARGE','PAYMENT','CREDIT','TAX','ADJUSTMENT')),
    category        VARCHAR(20) NOT NULL,                 -- 'ROOM','FNB','SPA','PARKING','TAX','OTHER'
    description     VARCHAR(255) NOT NULL,
    amount_cents    BIGINT NOT NULL,                      -- positive=debit, negative=credit
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    line_date       DATE NOT NULL,
    is_void         BOOLEAN NOT NULL DEFAULT false,

    -- JSONB: line-specific details
    details         JSONB NOT NULL DEFAULT '{}',
    -- For charges:
    -- {"quantity": 2, "unit_price_cents": 4250, "pos_reference": "POS-001",
    --  "gl_account": "4100", "tax_breakdown": [{"name": "VAT", "amount_cents": 850}]}
    -- For payments:
    -- {"method": "credit_card", "card_brand": "visa", "card_last_four": "4242",
    --  "payment_token": "tok_...", "gateway_ref": "ch_...", "auth_code": "A12345"}

    posted_by       UUID,                                 -- staff ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_folio_reservation ON folio(reservation_id);
CREATE INDEX idx_folio_guest ON folio(guest_id);
CREATE INDEX idx_folio_line_folio ON folio_line(folio_id, line_date);
CREATE INDEX idx_folio_line_date ON folio_line(line_date, line_type);
```

---

## Housekeeping

```sql
CREATE TABLE housekeeping_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    room_id         UUID NOT NULL REFERENCES room(id),
    task_type       VARCHAR(20) NOT NULL
                    CHECK (task_type IN ('CHECKOUT','STAYOVER','DEEP_CLEAN',
                                         'INSPECTION','TURNDOWN','MAINTENANCE')),
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING'
                    CHECK (status IN ('PENDING','ASSIGNED','IN_PROGRESS',
                                      'COMPLETED','INSPECTED','SKIPPED')),
    priority        SMALLINT NOT NULL DEFAULT 5,
    assigned_to     UUID,                                 -- staff ID
    scheduled_date  DATE NOT NULL,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,

    -- JSONB: task-specific details and checklist
    details         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "checklist": [
    --     {"item": "Strip and remake bed", "done": true},
    --     {"item": "Clean bathroom", "done": true},
    --     {"item": "Vacuum carpet", "done": false},
    --     {"item": "Restock minibar", "done": false}
    --   ],
    --   "inspection": {
    --     "inspected_by": "uuid...",
    --     "inspected_at": "2026-06-15T14:30:00Z",
    --     "passed": true,
    --     "notes": "All good"
    --   },
    --   "ai_scheduling": {
    --     "predicted_checkout_time": "10:30",
    --     "urgency_score": 0.85,
    --     "sequence_position": 3
    --   }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hk_property_date ON housekeeping_task(property_id, scheduled_date, status);
CREATE INDEX idx_hk_assigned ON housekeeping_task(assigned_to, scheduled_date)
    WHERE assigned_to IS NOT NULL;
```

---

## Channel Distribution

```sql
CREATE TABLE channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    code            VARCHAR(30) NOT NULL,
    name            VARCHAR(100) NOT NULL,
    channel_type    VARCHAR(20) NOT NULL
                    CHECK (channel_type IN ('OTA','GDS','DIRECT','METASEARCH',
                                            'CORPORATE','WHOLESALE')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    commission_pct  NUMERIC(5,2) DEFAULT 0,

    -- JSONB: channel-specific configuration
    config          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "credentials_vault_ref": "vault://channels/booking_com/prop123",
    --   "external_property_id": "12345",
    --   "rate_mappings": [
    --     {"local_code": "BAR", "external_code": "SBX_STANDARD"},
    --     {"local_code": "CORP", "external_code": "SBX_CORPORATE"}
    --   ],
    --   "room_type_mappings": [
    --     {"local_code": "KDLX", "external_code": "KING_DLX"}
    --   ],
    --   "sync_settings": {
    --     "push_frequency_minutes": 15,
    --     "pull_frequency_minutes": 5,
    --     "auto_confirm": true
    --   }
    -- }

    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, code)
);

CREATE TABLE channel_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id      UUID NOT NULL REFERENCES channel(id),
    direction       VARCHAR(10) NOT NULL CHECK (direction IN ('PUSH','PULL')),
    sync_type       VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    records_count   INT DEFAULT 0,
    error_message   TEXT,
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sync_log ON channel_sync_log(channel_id, created_at DESC);
```

---

## Staff & Access Control

```sql
CREATE TABLE staff (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: role and permission assignments
    access          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "role": "front_desk_manager",
    --   "properties": ["uuid-prop-1", "uuid-prop-2"],
    --   "permissions": [
    --     "reservation.*",
    --     "folio.read", "folio.charge", "folio.payment",
    --     "guest.*",
    --     "housekeeping.read",
    --     "rate.read",
    --     "report.occupancy", "report.revenue"
    --   ],
    --   "mfa_enabled": true,
    --   "department": "front_office"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
```

---

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    property_id     UUID,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    actor_id        UUID,                                 -- staff ID
    changes         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "reservation.status": {"old": "CONFIRMED", "new": "CHECKED_IN"},
    --   "reservation.room_id": {"old": null, "new": "uuid..."}
    -- }
    context         JSONB NOT NULL DEFAULT '{}',
    -- {"ip": "192.168.1.1", "user_agent": "...", "source": "front_desk_app"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);

-- GDPR consent tracking (relational -- this data has legal significance)
CREATE TABLE consent_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guest_id        UUID NOT NULL REFERENCES guest(id),
    consent_type    VARCHAR(50) NOT NULL,
    is_granted      BOOLEAN NOT NULL,
    granted_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    source          VARCHAR(50),
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_guest ON consent_record(guest_id);
```

---

## AI Revenue Management

```sql
CREATE TABLE ai_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    event_type      VARCHAR(50) NOT NULL,
    -- 'price_recommendation', 'demand_forecast', 'upsell_offer',
    -- 'pricing_signal', 'anomaly_detected'

    -- JSONB: event-specific payload (schema varies by event type)
    payload         JSONB NOT NULL,
    -- For price_recommendation:
    -- {
    --   "room_type_code": "KDLX",
    --   "stay_date": "2026-07-04",
    --   "current_rate_cents": 25000,
    --   "recommended_rate_cents": 31500,
    --   "confidence": 0.87,
    --   "signals": {
    --     "occupancy_forecast": 0.94,
    --     "comp_set_avg_cents": 28000,
    --     "local_event": "Independence Day Weekend",
    --     "booking_pace_ratio": 1.35
    --   },
    --   "model_version": "rev-mgmt-v2.3",
    --   "status": "accepted",
    --   "decided_by": "uuid...",
    --   "decided_at": "2026-06-20T09:00:00Z"
    -- }
    --
    -- For demand_forecast:
    -- {
    --   "forecast_date": "2026-07-04",
    --   "predicted_occupancy_pct": 94.2,
    --   "confidence_interval": [88.5, 97.1],
    --   "model_version": "forecast-v1.1"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_event_property ON ai_event(property_id, event_type, created_at);
CREATE INDEX idx_ai_event_payload ON ai_event USING GIN (payload jsonb_path_ops);
```

---

## Guest Communication

```sql
CREATE TABLE message (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    guest_id        UUID NOT NULL REFERENCES guest(id),
    reservation_id  UUID REFERENCES reservation(id),
    direction       VARCHAR(10) NOT NULL CHECK (direction IN ('OUTBOUND','INBOUND')),
    channel         VARCHAR(20) NOT NULL
                    CHECK (channel IN ('EMAIL','SMS','WHATSAPP','IN_APP','PUSH')),
    status          VARCHAR(20) NOT NULL DEFAULT 'QUEUED'
                    CHECK (status IN ('QUEUED','SENT','DELIVERED','READ','FAILED','REPLIED')),

    -- JSONB: message content and metadata
    content         JSONB NOT NULL,
    -- {
    --   "subject": "Your stay at Grand Hotel starts tomorrow!",
    --   "body": "Dear Mr. Smith, we're looking forward to...",
    --   "template_id": "pre_arrival_v2",
    --   "template_vars": {"guest_name": "Mr. Smith", "check_in_date": "June 15"},
    --   "ai_generated": true,
    --   "ai_model": "guest-comms-v1.2",
    --   "delivery": {
    --     "sent_at": "2026-06-14T10:00:00Z",
    --     "delivered_at": "2026-06-14T10:00:05Z",
    --     "provider_id": "msg_abc123"
    --   }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_message_guest ON message(guest_id);
CREATE INDEX idx_message_reservation ON message(reservation_id);
```

---

## Example Queries

### Find all guests who prefer high floors and have stayed 3+ times

```sql
SELECT id, first_name, last_name, email, total_stays, preferences
FROM guest
WHERE tenant_id = '{{tenant_id}}'
  AND preferences @> '{"room": {"floor": "high"}}'
  AND total_stays >= 3
ORDER BY total_stays DESC;
```

### Get jurisdiction-specific tax breakdown for a property

```sql
SELECT
    t.value->>'name' AS tax_name,
    (t.value->>'rate_pct')::numeric AS rate_pct,
    t.value->>'applies_to' AS applies_to
FROM property p,
     jsonb_array_elements(p.tax_config) AS t(value)
WHERE p.id = '{{property_id}}';
```

### Find all Booking.com reservations with commission above 12%

```sql
SELECT id, confirmation_number, check_in_date, total_amount_cents, details
FROM reservation
WHERE property_id = '{{property_id}}'
  AND details @> '{"channel": {"code": "BOOKING_COM"}}'
  AND (details->'channel'->>'commission_pct')::numeric > 12.0;
```

### Aggregate AI pricing recommendation acceptance rate

```sql
SELECT
    payload->>'room_type_code' AS room_type,
    COUNT(*) AS total_recommendations,
    COUNT(*) FILTER (WHERE payload->>'status' = 'accepted') AS accepted,
    ROUND(
        COUNT(*) FILTER (WHERE payload->>'status' = 'accepted')::numeric / COUNT(*) * 100, 1
    ) AS acceptance_rate_pct
FROM ai_event
WHERE property_id = '{{property_id}}'
  AND event_type = 'price_recommendation'
  AND created_at >= now() - interval '30 days'
GROUP BY payload->>'room_type_code';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenant, property (with JSONB config) |
| Room Inventory | 2 | room_type, room (with JSONB attributes) |
| Guest Profiles | 1 | guest (with JSONB profile and preferences) |
| Rate Management | 2 | rate_plan (with JSONB rules), rate |
| Availability | 1 | availability (with JSONB restrictions) |
| Reservations | 1 | reservation (with JSONB details) |
| Billing & Folios | 2 | folio (with JSONB billing_info), folio_line (with JSONB details) |
| Housekeeping | 1 | housekeeping_task (with JSONB details) |
| Channel Distribution | 2 | channel (with JSONB config), channel_sync_log |
| Staff & Access | 1 | staff (with JSONB access) |
| Audit & Compliance | 2 | audit_log, consent_record |
| AI & Analytics | 1 | ai_event (with JSONB payload) |
| Guest Communication | 1 | message (with JSONB content) |
| **Total** | **19** | Significantly leaner than normalized model (~37 tables) |

---

## Key Design Decisions

1. **JSONB for variable data, relational for queryable/joinable data.** Every table has a clear boundary: columns that are frequently filtered, joined, or indexed are relational; columns that vary by jurisdiction, property type, or feature version are JSONB. This avoids the "JSONB-for-everything" anti-pattern while capturing the "relational-for-everything" overhead.

2. **Property-level tax configuration in JSONB** rather than a separate `tax_rule` table with per-rule rows. Tax regimes differ dramatically by country and municipality (city tourist taxes, VAT, GST, resort fees) and change infrequently. JSONB array with `jsonb_array_elements()` handles this cleanly without a five-table tax hierarchy.

3. **Reservation details JSONB absorbs channel, company, agent, and group data** that would otherwise require 3-4 separate tables and junction tables. For the majority of reservations (individual direct or OTA bookings), most of these fields are null. JSONB avoids sparse-column problems.

4. **Daily rates remain relational** despite the hybrid approach. The `rate` table is high-volume (365 rows x room_types x rate_plans per property per year) and is the primary integration surface for channel managers. Relational structure with a composite primary key ensures efficient ARI pushes and pulls.

5. **Guest preferences in a separate JSONB column from profile.** `profile` contains PII (address, ID documents, company); `preferences` contains non-PII behavioural data (room preferences, dietary needs). This separation enables AI systems to process preferences without accessing PII, supporting GDPR data minimisation.

6. **Staff RBAC in JSONB** rather than the classic five-table RBAC model (staff, role, permission, role_permission, staff_role). For a PMS with ~10 roles and ~50 permissions, the full relational model adds complexity without proportional value. JSONB permissions are validated at the application layer.

7. **AI events as a single table with typed JSONB payloads.** Price recommendations, demand forecasts, upsell offers, and anomalies all go into `ai_event` with an `event_type` discriminator. This avoids creating a new table for each AI feature and allows the ML team to evolve payload schemas without database migrations.

8. **GIN indexes on JSONB columns for operational queries.** PostgreSQL's `jsonb_path_ops` GIN index supports the containment operator (`@>`), enabling efficient queries like "find guests who prefer king beds" or "find reservations from Booking.com" without full table scans.

9. **Expression indexes for frequently queried JSONB paths.** Where a specific JSONB field is queried repeatedly (e.g., loyalty tier), a B-tree expression index on that path provides relational-speed lookups without extracting the field into a column.

10. **JSON Schema validation at the application layer, not the database.** PostgreSQL CHECK constraints can validate JSONB structure, but complex schemas are better enforced in application code using JSON Schema (Draft 2020-12). This keeps the database schema clean while ensuring payload consistency.

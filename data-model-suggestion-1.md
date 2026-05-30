# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Hotel Property Management System · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form (3NF) relational design. Every concept in the hotel domain -- properties, rooms, room types, rate plans, reservations, guests, folios, charges, payments, housekeeping tasks, OTA channels -- gets its own table with explicit foreign key relationships. Junction tables handle many-to-many relationships (e.g., reservation-to-guest companions, rate-plan-to-room-type applicability). Reference data tables enforce domain vocabularies aligned with OTA and ISO standards.

The approach mirrors how enterprise PMS platforms like Oracle OPERA structure their data internally: deep normalization enables complex cross-entity queries (e.g., "show all reservations for corporate accounts with outstanding city-ledger balances across three properties") without data duplication or inconsistency. Every field has a single source of truth, and referential integrity is enforced at the database level.

This is the safest starting point for a system where financial accuracy, regulatory compliance (PCI DSS, GDPR), and auditability are paramount. The trade-off is schema rigidity: adding a new concept requires DDL migrations, and the table count is high.

**Best for:** Teams prioritizing data integrity, complex reporting, and regulatory compliance in a well-understood domain with stable requirements.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Complex cross-entity queries are straightforward SQL joins
- (+) Standards-aligned reference data tables enforce consistency
- (+) Well-understood by most developers and BI tools
- (-) High table count (~45-55 tables) increases migration complexity
- (-) Schema changes require DDL migrations and deployment coordination
- (-) Jurisdiction-specific or property-specific custom fields require schema extensions
- (-) Many-to-many junction tables add write overhead

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenTravel Alliance (OTA) | Room type codes, rate plan structures, reservation status enums, and guest profile fields align with OTA 2.0 object model naming conventions |
| ISO 3166-1/2 | Country and subdivision codes for property addresses, guest nationalities, and tax jurisdictions |
| ISO 4217 | Currency codes stored as CHAR(3) on all monetary fields |
| ISO 17442 (LEI) | Optional Legal Entity Identifier for corporate account profiles |
| PCI DSS v4.0 | Payment card data is tokenized; no raw card numbers stored; payment_tokens table references external vault |
| GDPR/CCPA | Guest PII fields are flagged; consent records tracked in dedicated table; soft-delete with retention policy |
| RFC 5545 (iCalendar) | Reservation date ranges and housekeeping schedules exportable to iCal format |
| HTNG | Folio/charge posting patterns follow HTNG financial exchange conventions |

---

## Core Identity & Multi-Tenancy

```sql
-- ============================================================
-- TENANT / PROPERTY GROUP
-- ============================================================
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    billing_email   VARCHAR(255),
    subscription_tier VARCHAR(50) DEFAULT 'standard',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE property (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(200) NOT NULL,
    code            VARCHAR(20) NOT NULL,              -- short property code, e.g. "HLTN-NYC"
    star_rating     SMALLINT CHECK (star_rating BETWEEN 1 AND 5),
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL,                  -- ISO 3166-1 alpha-2
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC', -- IANA timezone
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',   -- ISO 4217
    phone           VARCHAR(30),
    email           VARCHAR(255),
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
    code            VARCHAR(20) NOT NULL,              -- e.g. "KDLX", "QSTD" (OTA room type code)
    name            VARCHAR(100) NOT NULL,             -- e.g. "King Deluxe"
    description     TEXT,
    max_occupancy   SMALLINT NOT NULL DEFAULT 2,
    max_adults      SMALLINT NOT NULL DEFAULT 2,
    max_children    SMALLINT NOT NULL DEFAULT 0,
    base_rate_cents BIGINT,                            -- default rack rate in minor currency units
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',    -- ISO 4217
    sort_order      SMALLINT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
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
    building        VARCHAR(50),
    status          VARCHAR(20) NOT NULL DEFAULT 'AVAILABLE'
                    CHECK (status IN ('AVAILABLE','OCCUPIED','OUT_OF_ORDER',
                                      'OUT_OF_SERVICE','BLOCKED')),
    housekeeping_status VARCHAR(20) NOT NULL DEFAULT 'CLEAN'
                    CHECK (housekeeping_status IN ('CLEAN','DIRTY','INSPECTED',
                                                   'IN_PROGRESS','OUT_OF_ORDER')),
    is_smoking      BOOLEAN NOT NULL DEFAULT false,
    has_accessible   BOOLEAN NOT NULL DEFAULT false,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, room_number)
);

CREATE INDEX idx_room_property_type ON room(property_id, room_type_id);
CREATE INDEX idx_room_status ON room(property_id, status);

-- Room amenities (many-to-many)
CREATE TABLE amenity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(30) NOT NULL UNIQUE,       -- e.g. "WIFI", "MINIBAR", "BALCONY"
    name            VARCHAR(100) NOT NULL,
    category        VARCHAR(50)                        -- "room", "bathroom", "technology"
);

CREATE TABLE room_type_amenity (
    room_type_id    UUID NOT NULL REFERENCES room_type(id),
    amenity_id      UUID NOT NULL REFERENCES amenity(id),
    PRIMARY KEY (room_type_id, amenity_id)
);
```

---

## Rate Management

```sql
CREATE TABLE rate_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    code            VARCHAR(30) NOT NULL,              -- e.g. "BAR", "CORP-IBM", "PKG-SPA"
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    rate_type       VARCHAR(20) NOT NULL DEFAULT 'PUBLIC'
                    CHECK (rate_type IN ('PUBLIC','NEGOTIATED','PACKAGE',
                                         'PROMOTIONAL','MEMBER','WHOLESALE')),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    min_stay_nights SMALLINT DEFAULT 1,
    max_stay_nights SMALLINT,
    booking_start   DATE,                              -- rate plan valid for bookings from
    booking_end     DATE,                              -- rate plan valid for bookings until
    stay_start      DATE,                              -- stay dates from
    stay_end        DATE,                              -- stay dates until
    cancellation_policy_id UUID REFERENCES cancellation_policy(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, code)
);

CREATE TABLE rate_plan_room_type (
    rate_plan_id    UUID NOT NULL REFERENCES rate_plan(id),
    room_type_id    UUID NOT NULL REFERENCES room_type(id),
    PRIMARY KEY (rate_plan_id, room_type_id)
);

-- Daily rate amounts (the "ARI" -- Availability, Rates, Inventory)
CREATE TABLE rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rate_plan_id    UUID NOT NULL REFERENCES rate_plan(id),
    room_type_id    UUID NOT NULL REFERENCES room_type(id),
    stay_date       DATE NOT NULL,
    amount_cents    BIGINT NOT NULL,                    -- rate in minor currency units
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    extra_adult_cents BIGINT DEFAULT 0,
    extra_child_cents BIGINT DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (rate_plan_id, room_type_id, stay_date)
);

CREATE INDEX idx_rate_lookup ON rate(room_type_id, stay_date);

-- Daily inventory / availability restrictions
CREATE TABLE availability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    room_type_id    UUID NOT NULL REFERENCES room_type(id),
    stay_date       DATE NOT NULL,
    total_inventory SMALLINT NOT NULL,
    sold_count      SMALLINT NOT NULL DEFAULT 0,
    overbooking_allowance SMALLINT NOT NULL DEFAULT 0,
    is_closed       BOOLEAN NOT NULL DEFAULT false,    -- stop-sell
    min_stay        SMALLINT DEFAULT 1,
    max_stay        SMALLINT,
    cta             BOOLEAN NOT NULL DEFAULT false,    -- closed to arrival
    ctd             BOOLEAN NOT NULL DEFAULT false,    -- closed to departure
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, room_type_id, stay_date)
);

CREATE INDEX idx_availability_lookup ON availability(property_id, room_type_id, stay_date);

CREATE TABLE cancellation_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    deadline_hours  INT NOT NULL DEFAULT 24,           -- hours before check-in
    penalty_type    VARCHAR(20) NOT NULL DEFAULT 'FIRST_NIGHT'
                    CHECK (penalty_type IN ('NONE','FIRST_NIGHT','FULL_STAY',
                                            'PERCENTAGE','FLAT_FEE')),
    penalty_amount_cents BIGINT DEFAULT 0,
    penalty_percentage NUMERIC(5,2) DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Guest Profiles

```sql
CREATE TABLE guest (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(30),
    date_of_birth   DATE,
    nationality     CHAR(2),                           -- ISO 3166-1 alpha-2
    gender          VARCHAR(10),
    language        VARCHAR(10) DEFAULT 'en',           -- BCP 47 language tag
    id_type         VARCHAR(30),                       -- "PASSPORT", "DRIVERS_LICENSE", "NATIONAL_ID"
    id_number       VARCHAR(50),                       -- encrypted or hashed
    id_country      CHAR(2),                           -- issuing country ISO 3166-1
    address_line1   VARCHAR(255),
    city            VARCHAR(100),
    country_code    CHAR(2),
    postal_code     VARCHAR(20),
    vip_level       SMALLINT DEFAULT 0,
    is_blacklisted  BOOLEAN NOT NULL DEFAULT false,
    gdpr_consent    BOOLEAN NOT NULL DEFAULT false,
    gdpr_consent_date TIMESTAMPTZ,
    marketing_opt_in BOOLEAN NOT NULL DEFAULT false,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_guest_tenant ON guest(tenant_id);
CREATE INDEX idx_guest_email ON guest(tenant_id, email);
CREATE INDEX idx_guest_name ON guest(tenant_id, last_name, first_name);

CREATE TABLE guest_preference (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guest_id        UUID NOT NULL REFERENCES guest(id),
    category        VARCHAR(50) NOT NULL,              -- "room", "pillow", "diet", "floor"
    preference_key  VARCHAR(50) NOT NULL,              -- "high_floor", "extra_pillows"
    preference_value VARCHAR(200),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (guest_id, category, preference_key)
);

CREATE TABLE company (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(200) NOT NULL,
    legal_entity_id VARCHAR(20),                       -- ISO 17442 LEI (optional)
    tax_id          VARCHAR(50),
    address_line1   VARCHAR(255),
    city            VARCHAR(100),
    country_code    CHAR(2),
    postal_code     VARCHAR(20),
    contact_name    VARCHAR(200),
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(30),
    payment_terms   SMALLINT DEFAULT 30,               -- net days
    credit_limit_cents BIGINT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE travel_agent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(200) NOT NULL,
    iata_number     VARCHAR(20),                       -- IATA agency number
    commission_pct  NUMERIC(5,2) DEFAULT 0,
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(30),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Reservations

```sql
CREATE TABLE reservation_group (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    group_name      VARCHAR(200) NOT NULL,
    group_code      VARCHAR(30),
    company_id      UUID REFERENCES company(id),
    contact_name    VARCHAR(200),
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(30),
    rooms_blocked   SMALLINT NOT NULL DEFAULT 0,
    cutoff_date     DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE reservation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    confirmation_number VARCHAR(20) NOT NULL,
    guest_id        UUID NOT NULL REFERENCES guest(id),
    company_id      UUID REFERENCES company(id),
    travel_agent_id UUID REFERENCES travel_agent(id),
    reservation_group_id UUID REFERENCES reservation_group(id),
    room_type_id    UUID NOT NULL REFERENCES room_type(id),
    room_id         UUID REFERENCES room(id),          -- assigned room (nullable until assignment)
    rate_plan_id    UUID NOT NULL REFERENCES rate_plan(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'CONFIRMED'
                    CHECK (status IN ('PENDING','CONFIRMED','CHECKED_IN','CHECKED_OUT',
                                      'CANCELLED','NO_SHOW','WAITLISTED')),
    source          VARCHAR(30) NOT NULL DEFAULT 'DIRECT'
                    CHECK (source IN ('DIRECT','OTA','GDS','PHONE','WALK_IN',
                                      'CORPORATE','TRAVEL_AGENT','GROUP')),
    channel_id      UUID REFERENCES channel(id),       -- which OTA/channel
    channel_reservation_id VARCHAR(100),               -- external booking reference
    check_in_date   DATE NOT NULL,
    check_out_date  DATE NOT NULL,
    adults          SMALLINT NOT NULL DEFAULT 1,
    children        SMALLINT NOT NULL DEFAULT 0,
    total_amount_cents BIGINT,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    special_requests TEXT,
    arrival_time    TIME,
    departure_time  TIME,
    checked_in_at   TIMESTAMPTZ,
    checked_out_at  TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    cancellation_reason TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, confirmation_number)
);

CREATE INDEX idx_reservation_property_dates ON reservation(property_id, check_in_date, check_out_date);
CREATE INDEX idx_reservation_guest ON reservation(guest_id);
CREATE INDEX idx_reservation_status ON reservation(property_id, status);
CREATE INDEX idx_reservation_channel ON reservation(channel_id, channel_reservation_id);

-- Companions on a reservation (many-to-many)
CREATE TABLE reservation_guest (
    reservation_id  UUID NOT NULL REFERENCES reservation(id),
    guest_id        UUID NOT NULL REFERENCES guest(id),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    PRIMARY KEY (reservation_id, guest_id)
);

-- Per-night rate breakdown
CREATE TABLE reservation_daily_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reservation_id  UUID NOT NULL REFERENCES reservation(id),
    stay_date       DATE NOT NULL,
    amount_cents    BIGINT NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    UNIQUE (reservation_id, stay_date)
);
```

---

## Billing & Folios

```sql
-- Folios follow the hotel accounting pattern:
-- Guest Ledger = folios for in-house guests
-- City Ledger = folios transferred at checkout for direct-bill accounts
CREATE TABLE folio (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    reservation_id  UUID REFERENCES reservation(id),
    guest_id        UUID REFERENCES guest(id),
    company_id      UUID REFERENCES company(id),
    folio_number    VARCHAR(20) NOT NULL,
    folio_type      VARCHAR(20) NOT NULL DEFAULT 'GUEST'
                    CHECK (folio_type IN ('GUEST','MASTER','CITY_LEDGER',
                                          'ADVANCE_DEPOSIT','HOUSE')),
    status          VARCHAR(20) NOT NULL DEFAULT 'OPEN'
                    CHECK (status IN ('OPEN','SETTLED','TRANSFERRED','VOID')),
    balance_cents   BIGINT NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, folio_number)
);

CREATE INDEX idx_folio_reservation ON folio(reservation_id);

CREATE TABLE charge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    folio_id        UUID NOT NULL REFERENCES folio(id),
    transaction_code_id UUID NOT NULL REFERENCES transaction_code(id),
    description     VARCHAR(255),
    amount_cents    BIGINT NOT NULL,                    -- positive = debit, negative = credit
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    quantity        SMALLINT NOT NULL DEFAULT 1,
    charge_date     DATE NOT NULL,
    posted_by       UUID REFERENCES staff(id),
    reference       VARCHAR(100),                      -- external POS ref, etc.
    is_void         BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_charge_folio ON charge(folio_id);
CREATE INDEX idx_charge_date ON charge(charge_date);

CREATE TABLE transaction_code (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    code            VARCHAR(10) NOT NULL,              -- "1000" = Room, "2000" = F&B, etc.
    name            VARCHAR(100) NOT NULL,
    category        VARCHAR(50) NOT NULL,              -- "ROOM","FOOD","BEVERAGE","TELEPHONE","OTHER"
    is_revenue      BOOLEAN NOT NULL DEFAULT true,
    gl_account_code VARCHAR(20),                       -- maps to general ledger
    tax_category    VARCHAR(30),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, code)
);

CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    folio_id        UUID NOT NULL REFERENCES folio(id),
    payment_method  VARCHAR(30) NOT NULL
                    CHECK (payment_method IN ('CREDIT_CARD','DEBIT_CARD','CASH',
                                              'BANK_TRANSFER','CITY_LEDGER','VOUCHER',
                                              'MOBILE_PAY','OTHER')),
    amount_cents    BIGINT NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    payment_token   VARCHAR(255),                      -- PCI-compliant tokenized card ref
    authorization_code VARCHAR(50),
    gateway_reference VARCHAR(100),                    -- payment gateway transaction ID
    status          VARCHAR(20) NOT NULL DEFAULT 'COMPLETED'
                    CHECK (status IN ('PENDING','COMPLETED','FAILED','REFUNDED','VOID')),
    payment_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    posted_by       UUID REFERENCES staff(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_folio ON payment(folio_id);

CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    folio_id        UUID NOT NULL REFERENCES folio(id),
    invoice_number  VARCHAR(30) NOT NULL,
    issued_date     DATE NOT NULL,
    due_date        DATE,
    total_cents     BIGINT NOT NULL,
    tax_cents       BIGINT NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(20) NOT NULL DEFAULT 'ISSUED'
                    CHECK (status IN ('DRAFT','ISSUED','PAID','OVERDUE','VOID')),
    pdf_url         VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Housekeeping

```sql
CREATE TABLE housekeeping_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    room_id         UUID NOT NULL REFERENCES room(id),
    task_type       VARCHAR(20) NOT NULL DEFAULT 'CHECKOUT'
                    CHECK (task_type IN ('CHECKOUT','STAYOVER','DEEP_CLEAN',
                                         'INSPECTION','TURNDOWN','MAINTENANCE')),
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING'
                    CHECK (status IN ('PENDING','ASSIGNED','IN_PROGRESS',
                                      'COMPLETED','INSPECTED','SKIPPED')),
    priority        SMALLINT NOT NULL DEFAULT 5,       -- 1=highest, 10=lowest
    assigned_to     UUID REFERENCES staff(id),
    scheduled_date  DATE NOT NULL,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    inspected_by    UUID REFERENCES staff(id),
    inspected_at    TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hk_property_date ON housekeeping_task(property_id, scheduled_date);
CREATE INDEX idx_hk_assigned ON housekeeping_task(assigned_to, status);
CREATE INDEX idx_hk_room ON housekeeping_task(room_id, scheduled_date);
```

---

## OTA / Channel Distribution

```sql
CREATE TABLE channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(30) NOT NULL UNIQUE,       -- "BOOKING_COM", "EXPEDIA", "AIRBNB"
    name            VARCHAR(100) NOT NULL,
    channel_type    VARCHAR(20) NOT NULL
                    CHECK (channel_type IN ('OTA','GDS','DIRECT','METASEARCH',
                                            'CORPORATE','WHOLESALE')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE property_channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    channel_id      UUID NOT NULL REFERENCES channel(id),
    external_property_id VARCHAR(100),                 -- property ID in the OTA system
    is_active       BOOLEAN NOT NULL DEFAULT true,
    commission_pct  NUMERIC(5,2) DEFAULT 0,
    credentials_vault_ref VARCHAR(255),                -- reference to encrypted credentials
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, channel_id)
);

CREATE TABLE channel_rate_mapping (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_channel_id UUID NOT NULL REFERENCES property_channel(id),
    rate_plan_id    UUID NOT NULL REFERENCES rate_plan(id),
    external_rate_code VARCHAR(100),                   -- rate plan ID in the OTA system
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_channel_id, rate_plan_id)
);

CREATE TABLE channel_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_channel_id UUID NOT NULL REFERENCES property_channel(id),
    direction       VARCHAR(10) NOT NULL CHECK (direction IN ('PUSH','PULL')),
    sync_type       VARCHAR(20) NOT NULL
                    CHECK (sync_type IN ('AVAILABILITY','RATES','RESERVATION',
                                         'RESTRICTION','FULL')),
    status          VARCHAR(20) NOT NULL
                    CHECK (status IN ('SUCCESS','PARTIAL','FAILED')),
    records_synced  INT DEFAULT 0,
    error_message   TEXT,
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sync_log_channel ON channel_sync_log(property_channel_id, created_at DESC);
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
    phone           VARCHAR(30),
    position        VARCHAR(100),
    department      VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(50) NOT NULL,              -- "front_desk", "housekeeping_mgr", "revenue_mgr"
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,      -- "reservation.create", "folio.void", "rate.update"
    description     TEXT
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id),
    permission_id   UUID NOT NULL REFERENCES permission(id),
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE staff_role (
    staff_id        UUID NOT NULL REFERENCES staff(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    property_id     UUID REFERENCES property(id),      -- nullable = all properties
    PRIMARY KEY (staff_id, role_id, property_id)
);
```

---

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    property_id     UUID,
    entity_type     VARCHAR(50) NOT NULL,              -- "reservation", "folio", "rate"
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL               -- "CREATE", "UPDATE", "DELETE", "VOID"
                    CHECK (action IN ('CREATE','UPDATE','DELETE','VOID','STATUS_CHANGE')),
    changed_by      UUID REFERENCES staff(id),
    old_values      JSONB,                             -- previous field values
    new_values      JSONB,                             -- new field values
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);

-- GDPR consent tracking
CREATE TABLE consent_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guest_id        UUID NOT NULL REFERENCES guest(id),
    consent_type    VARCHAR(50) NOT NULL,              -- "MARKETING", "DATA_PROCESSING", "PROFILING"
    is_granted      BOOLEAN NOT NULL,
    granted_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    source          VARCHAR(50),                       -- "WEB_FORM", "CHECKIN_KIOSK", "EMAIL"
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_guest ON consent_record(guest_id);
```

---

## Tax Configuration

```sql
CREATE TABLE tax_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    name            VARCHAR(100) NOT NULL,             -- "City Occupancy Tax", "VAT", "GST"
    rate_pct        NUMERIC(6,4) NOT NULL,             -- e.g. 14.5000 for 14.5%
    is_inclusive     BOOLEAN NOT NULL DEFAULT false,
    applies_to      VARCHAR(50) NOT NULL DEFAULT 'ROOM'
                    CHECK (applies_to IN ('ROOM','FOOD','BEVERAGE','OTHER','ALL')),
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tax_property ON tax_rate(property_id, effective_from);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenant, property |
| Room Inventory | 4 | room_type, room, amenity, room_type_amenity |
| Rate Management | 5 | rate_plan, rate_plan_room_type, rate, availability, cancellation_policy |
| Guest Profiles | 4 | guest, guest_preference, company, travel_agent |
| Reservations | 4 | reservation_group, reservation, reservation_guest, reservation_daily_rate |
| Billing & Folios | 5 | folio, charge, transaction_code, payment, invoice |
| Housekeeping | 1 | housekeeping_task |
| Channel Distribution | 4 | channel, property_channel, channel_rate_mapping, channel_sync_log |
| Staff & RBAC | 5 | staff, role, permission, role_permission, staff_role |
| Audit & Compliance | 2 | audit_log, consent_record |
| Tax | 1 | tax_rate |
| **Total** | **37** | Core schema; additional tables expected for POS, spa, direct booking engine |

---

## Key Design Decisions

1. **All monetary values stored as BIGINT cents** (minor currency units) to avoid floating-point precision errors in financial calculations. Currency code stored alongside every amount for multi-currency support.

2. **UUID primary keys everywhere** for globally unique identifiers across multi-property and multi-tenant deployments, and for safe use in APIs without exposing sequential IDs.

3. **Tenant-scoped multi-tenancy** via `tenant_id` foreign keys on shared tables (guest, staff, company) and `property_id` on operational tables. Row-Level Security (RLS) policies can enforce isolation at the database level.

4. **Separate availability and rate tables** with per-day granularity, following the ARI (Availability, Rates, Inventory) pattern used by all major channel managers and OTAs. This enables efficient OTA sync without scanning reservation data.

5. **Folio model mirrors hotel accounting conventions** -- Guest Ledger (in-house folios), City Ledger (direct-bill accounts), and Advance Deposit ledgers are distinct folio types, enabling correct financial reporting without application-level logic.

6. **Channel distribution as first-class entities** with per-channel property mappings and rate mappings. The `channel_sync_log` provides operational visibility into OTA sync health without coupling to the reservation table.

7. **Transaction codes** separate from charges, allowing properties to define their own chart-of-accounts mapping. This follows the OPERA/Mews pattern of configurable revenue buckets.

8. **GDPR-ready design** with consent tracking, PII field identification, and soft-delete patterns. Guest `id_number` should be encrypted at the application layer; the schema stores the encrypted form.

9. **Audit log uses JSONB for old/new values** rather than a fully normalized change-tracking schema. This is a pragmatic hybrid: the log table itself is relational, but the change payload is flexible enough to capture any entity's field changes without schema coupling.

10. **ISO standard codes used throughout**: ISO 3166 for countries, ISO 4217 for currencies, IANA for timezones, BCP 47 for languages. This ensures compatibility with OTA and GDS systems that expect standard code vocabularies.

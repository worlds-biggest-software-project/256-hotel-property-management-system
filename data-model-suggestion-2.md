# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Hotel Property Management System · Created: 2026-05-22

## Philosophy

This model uses event sourcing as its foundational principle: every change to the hotel's state -- a reservation created, a room assigned, a charge posted, a rate updated -- is captured as an immutable event in an append-only event store. The current state of any entity (a reservation, a folio, a room) is derived by replaying its event stream. Materialised read models (projections) are maintained separately for efficient querying, following the CQRS (Command Query Responsibility Segregation) pattern.

This architecture is inspired by financial ledger systems and has direct precedent in hospitality: Mews bills itself as a "ledger-first" PMS, and the hotel folio itself is conceptually an event log (a running sequence of charges and payments). Event sourcing extends this principle to the entire system. Every reservation status change, every rate modification, every housekeeping state transition becomes a first-class, queryable, replayable event.

The approach is particularly powerful for an AI-native PMS because the event stream is a natural training dataset. Revenue management ML models can consume rate-change and booking-pace events directly. Guest preference modelling can replay historical stay events. Predictive housekeeping can analyse check-out timing events. The event store also provides an automatic, complete audit trail that satisfies PCI DSS and GDPR requirements without any additional audit-logging infrastructure.

**Best for:** Teams building an AI-native PMS where full audit trails, temporal queries ("what was the rate on March 5th?"), and ML-consumable event streams are core requirements.

**Trade-offs:**
- (+) Complete, immutable audit trail by design -- no separate audit log needed
- (+) Temporal queries are trivial: replay events to any point in time
- (+) Natural ML training data: the event stream is a rich, structured dataset
- (+) Decoupled read/write paths allow independent scaling
- (+) Easy to add new read models (projections) without schema migration
- (-) Higher implementation complexity: event handlers, projectors, and eventual consistency
- (-) Read model rebuild can be slow if event volume is high
- (-) Eventual consistency between writes and reads requires careful UX handling
- (-) Debugging requires understanding event replay, not just inspecting current state
- (-) Storage grows faster than a mutable-state model (events are never deleted)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenTravel Alliance (OTA) | Event payloads for reservation and rate events use OTA-aligned field names and enums |
| ISO 3166-1/2 | Country and subdivision codes in property and guest event data |
| ISO 4217 | Currency codes on all monetary event payloads |
| PCI DSS v4.0 | Payment events store only tokenized references; card data never enters the event store |
| GDPR/CCPA | Guest PII in event payloads is encrypted; GDPR erasure uses crypto-shredding (destroy the encryption key) |
| OCSF (Open Cybersecurity Schema Framework) | Event metadata structure (timestamp, actor, action, target) aligns with OCSF event patterns |
| CloudEvents (CNCF) | Event envelope structure follows CloudEvents v1.0 specification for interoperability |

---

## Event Store (Source of Truth)

```sql
-- ============================================================
-- CORE EVENT STORE
-- The single source of truth. All state is derived from here.
-- ============================================================
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                     -- aggregate root ID (e.g., reservation ID)
    stream_type     VARCHAR(50) NOT NULL,              -- "reservation", "folio", "room", "rate_plan"
    event_type      VARCHAR(100) NOT NULL,             -- "ReservationCreated", "ChargePosted", etc.
    event_version   INT NOT NULL,                      -- sequential version within the stream
    tenant_id       UUID NOT NULL,
    property_id     UUID,
    payload         JSONB NOT NULL,                    -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',       -- actor, IP, correlation ID, causation ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)                  -- optimistic concurrency control
);

-- Primary query pattern: get all events for an aggregate
CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);

-- Query by type for projectors and event handlers
CREATE INDEX idx_event_type ON event_store(event_type, created_at);

-- Tenant-scoped queries
CREATE INDEX idx_event_tenant ON event_store(tenant_id, created_at);

-- Property-scoped queries for operational dashboards
CREATE INDEX idx_event_property ON event_store(property_id, event_type, created_at);

-- GIN index for JSONB payload queries (e.g., find all events for a guest)
CREATE INDEX idx_event_payload ON event_store USING GIN (payload jsonb_path_ops);

-- ============================================================
-- EVENT SNAPSHOTS
-- Periodic snapshots avoid full replay for long-lived aggregates
-- ============================================================
CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INT NOT NULL,                     -- event_version at snapshot time
    state           JSONB NOT NULL,                    -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

### Event Type Taxonomy

```
-- Reservation Events
ReservationCreated          -- new booking with all initial details
ReservationModified         -- dates, room type, guest count changed
ReservationCancelled        -- cancellation with reason
ReservationCheckedIn        -- guest arrived, room assigned
ReservationCheckedOut       -- guest departed
ReservationNoShow           -- guest did not arrive
RoomAssigned                -- specific room assigned to reservation
RoomChanged                 -- room swap during stay
CompanionAdded              -- additional guest linked
CompanionRemoved            -- guest unlinked

-- Rate & Availability Events
RatePlanCreated             -- new rate plan defined
RatePlanUpdated             -- rate plan terms modified
RateSet                     -- daily rate amount set for room type + date
AvailabilityUpdated         -- inventory count or restrictions changed
StopSellApplied             -- channel closed for date range
StopSellLifted              -- channel reopened

-- Folio & Billing Events
FolioCreated                -- new folio opened
ChargePosted                -- debit added to folio
CreditPosted                -- credit/adjustment added
PaymentReceived             -- payment applied to folio
PaymentRefunded             -- refund processed
FolioTransferred            -- charges moved to city ledger / master folio
InvoiceGenerated            -- invoice PDF created
FolioSettled                -- balance zeroed, folio closed

-- Housekeeping Events
HousekeepingTaskCreated     -- cleaning task generated
HousekeepingTaskAssigned    -- staff member assigned
HousekeepingStarted         -- cleaning in progress
HousekeepingCompleted       -- cleaning finished
RoomInspected               -- supervisor inspection done
RoomStatusChanged           -- clean/dirty/OOO/OOS status change

-- Guest Profile Events
GuestProfileCreated         -- new guest record
GuestProfileUpdated         -- contact or preference changed
GuestConsentGranted         -- GDPR consent recorded
GuestConsentRevoked         -- GDPR consent withdrawn
GuestMerged                 -- duplicate profiles merged

-- Channel Distribution Events
ChannelSyncPushed           -- ARI data sent to OTA
ChannelSyncPulled           -- reservations pulled from OTA
ChannelSyncFailed           -- sync error recorded

-- AI/ML Events
DynamicPriceRecommended     -- AI engine suggested a rate change
DynamicPriceApplied         -- recommended rate was accepted
DemandForecastGenerated     -- forecast model output recorded
UpsellOffered               -- upgrade/add-on offered to guest
UpsellAccepted              -- guest accepted upsell
UpsellDeclined              -- guest declined upsell
```

### Example Event Payloads

```json
-- ReservationCreated
{
  "confirmation_number": "RES-2026-00142",
  "guest_id": "a1b2c3d4-...",
  "room_type_code": "KDLX",
  "rate_plan_code": "BAR",
  "check_in": "2026-06-15",
  "check_out": "2026-06-18",
  "adults": 2,
  "children": 0,
  "source": "OTA",
  "channel_code": "BOOKING_COM",
  "channel_reservation_id": "BDC-9876543",
  "total_amount_cents": 75000,
  "currency": "USD",
  "special_requests": "High floor, late check-in"
}

-- ChargePosted
{
  "folio_id": "f1e2d3c4-...",
  "transaction_code": "2010",
  "description": "Restaurant - Dinner",
  "amount_cents": 8500,
  "currency": "USD",
  "quantity": 1,
  "charge_date": "2026-06-16",
  "pos_reference": "POS-2026-0891"
}

-- DynamicPriceRecommended (AI event)
{
  "room_type_code": "KDLX",
  "stay_date": "2026-07-04",
  "current_rate_cents": 25000,
  "recommended_rate_cents": 31500,
  "confidence": 0.87,
  "demand_signals": {
    "occupancy_forecast": 0.94,
    "comp_set_avg_cents": 28000,
    "local_event": "Independence Day Weekend",
    "booking_pace_vs_avg": 1.35
  },
  "model_version": "rev-mgmt-v2.3"
}
```

---

## Read Models (Projections)

These tables are **derived state**, rebuilt from the event store. They can be dropped and reconstructed at any time.

```sql
-- ============================================================
-- PROJECTION: Current Reservation State
-- ============================================================
CREATE TABLE v_reservation (
    id              UUID PRIMARY KEY,                  -- same as stream_id
    property_id     UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    confirmation_number VARCHAR(20) NOT NULL,
    guest_id        UUID NOT NULL,
    guest_name      VARCHAR(200),                      -- denormalized for display
    company_id      UUID,
    room_type_code  VARCHAR(20) NOT NULL,
    room_number     VARCHAR(10),
    rate_plan_code  VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    source          VARCHAR(30) NOT NULL,
    channel_code    VARCHAR(30),
    check_in_date   DATE NOT NULL,
    check_out_date  DATE NOT NULL,
    adults          SMALLINT NOT NULL DEFAULT 1,
    children        SMALLINT NOT NULL DEFAULT 0,
    total_amount_cents BIGINT,
    currency_code   CHAR(3) NOT NULL,
    special_requests TEXT,
    checked_in_at   TIMESTAMPTZ,
    checked_out_at  TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    last_event_version INT NOT NULL,                   -- tracks projection currency
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, confirmation_number)
);

CREATE INDEX idx_vres_property_dates ON v_reservation(property_id, check_in_date, check_out_date);
CREATE INDEX idx_vres_status ON v_reservation(property_id, status);
CREATE INDEX idx_vres_guest ON v_reservation(guest_id);

-- ============================================================
-- PROJECTION: Room Status Board (front desk view)
-- ============================================================
CREATE TABLE v_room_status (
    room_id         UUID PRIMARY KEY,
    property_id     UUID NOT NULL,
    room_number     VARCHAR(10) NOT NULL,
    room_type_code  VARCHAR(20) NOT NULL,
    floor           VARCHAR(10),
    occupancy_status VARCHAR(20) NOT NULL,             -- VACANT, OCCUPIED, DUE_OUT, DUE_IN
    housekeeping_status VARCHAR(20) NOT NULL,          -- CLEAN, DIRTY, IN_PROGRESS, INSPECTED
    current_guest_name VARCHAR(200),
    current_reservation_id UUID,
    check_out_date  DATE,
    next_reservation_id UUID,
    next_check_in_date DATE,
    assigned_housekeeper VARCHAR(200),
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vroom_property ON v_room_status(property_id, occupancy_status);

-- ============================================================
-- PROJECTION: Folio / Billing State
-- ============================================================
CREATE TABLE v_folio (
    id              UUID PRIMARY KEY,
    property_id     UUID NOT NULL,
    reservation_id  UUID,
    guest_id        UUID,
    guest_name      VARCHAR(200),
    folio_number    VARCHAR(20) NOT NULL,
    folio_type      VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    total_charges_cents BIGINT NOT NULL DEFAULT 0,
    total_payments_cents BIGINT NOT NULL DEFAULT 0,
    balance_cents   BIGINT NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL,
    last_event_version INT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE v_folio_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    folio_id        UUID NOT NULL REFERENCES v_folio(id),
    line_type       VARCHAR(10) NOT NULL,              -- "CHARGE", "PAYMENT", "CREDIT"
    transaction_code VARCHAR(10),
    description     VARCHAR(255),
    amount_cents    BIGINT NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    line_date       DATE NOT NULL,
    reference       VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_vfolio_line_folio ON v_folio_line(folio_id, line_date);

-- ============================================================
-- PROJECTION: Daily Availability & Rates (ARI)
-- ============================================================
CREATE TABLE v_availability (
    property_id     UUID NOT NULL,
    room_type_code  VARCHAR(20) NOT NULL,
    stay_date       DATE NOT NULL,
    total_inventory SMALLINT NOT NULL,
    sold_count      SMALLINT NOT NULL DEFAULT 0,
    available_count SMALLINT GENERATED ALWAYS AS (total_inventory - sold_count) STORED,
    is_closed       BOOLEAN NOT NULL DEFAULT false,
    min_stay        SMALLINT DEFAULT 1,
    cta             BOOLEAN NOT NULL DEFAULT false,
    ctd             BOOLEAN NOT NULL DEFAULT false,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (property_id, room_type_code, stay_date)
);

CREATE TABLE v_rate (
    property_id     UUID NOT NULL,
    rate_plan_code  VARCHAR(30) NOT NULL,
    room_type_code  VARCHAR(20) NOT NULL,
    stay_date       DATE NOT NULL,
    amount_cents    BIGINT NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (property_id, rate_plan_code, room_type_code, stay_date)
);

-- ============================================================
-- PROJECTION: Guest Profile (CRM view)
-- ============================================================
CREATE TABLE v_guest (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(30),
    nationality     CHAR(2),
    language        VARCHAR(10),
    vip_level       SMALLINT DEFAULT 0,
    total_stays     INT NOT NULL DEFAULT 0,
    total_revenue_cents BIGINT NOT NULL DEFAULT 0,
    last_stay_date  DATE,
    last_property_id UUID,
    gdpr_consent    BOOLEAN NOT NULL DEFAULT false,
    preferences     JSONB DEFAULT '{}',
    -- Example preferences JSONB:
    -- {
    --   "room": {"floor": "high", "bed": "king", "view": "city"},
    --   "pillow": "firm",
    --   "diet": "vegetarian",
    --   "minibar": "pre-stock wine"
    -- }
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vguest_tenant ON v_guest(tenant_id);
CREATE INDEX idx_vguest_email ON v_guest(tenant_id, email);

-- ============================================================
-- PROJECTION: Revenue & Occupancy Dashboard
-- ============================================================
CREATE TABLE v_daily_stats (
    property_id     UUID NOT NULL,
    business_date   DATE NOT NULL,
    total_rooms     SMALLINT NOT NULL,
    rooms_sold      SMALLINT NOT NULL DEFAULT 0,
    rooms_available SMALLINT NOT NULL DEFAULT 0,
    occupancy_pct   NUMERIC(5,2) DEFAULT 0,
    adr_cents       BIGINT DEFAULT 0,                  -- average daily rate
    revpar_cents    BIGINT DEFAULT 0,                  -- revenue per available room
    total_room_revenue_cents BIGINT DEFAULT 0,
    total_fb_revenue_cents BIGINT DEFAULT 0,
    total_other_revenue_cents BIGINT DEFAULT 0,
    currency_code   CHAR(3) NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (property_id, business_date)
);
```

---

## Projection Infrastructure

```sql
-- ============================================================
-- PROJECTION TRACKING
-- Tracks which events each projector has processed
-- ============================================================
CREATE TABLE projection_checkpoint (
    projector_name  VARCHAR(100) PRIMARY KEY,           -- "reservation_projector", "folio_projector"
    last_event_id   UUID,
    last_event_at   TIMESTAMPTZ,
    events_processed BIGINT NOT NULL DEFAULT 0,
    is_rebuilding   BOOLEAN NOT NULL DEFAULT false,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- DEAD LETTER QUEUE
-- Events that failed projection processing
-- ============================================================
CREATE TABLE projection_dead_letter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,
    projector_name  VARCHAR(100) NOT NULL,
    error_message   TEXT,
    retry_count     SMALLINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Reference Data (Shared)

```sql
-- These tables are NOT event-sourced; they are slowly-changing
-- configuration managed via standard CRUD with audit events.

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE property (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(200) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE room (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    room_number     VARCHAR(10) NOT NULL,
    room_type_code  VARCHAR(20) NOT NULL,
    floor           VARCHAR(10),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (property_id, room_number)
);

CREATE TABLE transaction_code (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    code            VARCHAR(10) NOT NULL,
    name            VARCHAR(100) NOT NULL,
    category        VARCHAR(50) NOT NULL,
    gl_account_code VARCHAR(20),
    UNIQUE (property_id, code)
);

CREATE TABLE channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(30) NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL,
    channel_type    VARCHAR(20) NOT NULL
);
```

---

## Example Queries

### Rebuild a reservation's current state from events

```sql
SELECT event_type, event_version, payload, created_at
FROM event_store
WHERE stream_id = '{{reservation_id}}'
  AND stream_type = 'reservation'
ORDER BY event_version ASC;
```

### "What was the rate for King Deluxe on July 4th when it was last changed?"

```sql
SELECT payload, created_at
FROM event_store
WHERE event_type = 'RateSet'
  AND property_id = '{{property_id}}'
  AND payload @> '{"room_type_code": "KDLX", "stay_date": "2026-07-04"}'
ORDER BY created_at DESC
LIMIT 1;
```

### Get all AI pricing recommendations for a date range

```sql
SELECT payload, created_at
FROM event_store
WHERE event_type = 'DynamicPriceRecommended'
  AND property_id = '{{property_id}}'
  AND created_at BETWEEN '2026-06-01' AND '2026-06-30'
ORDER BY created_at;
```

### GDPR "right to be forgotten" via crypto-shredding

```sql
-- Guest PII in event payloads is encrypted with a per-guest key.
-- To erase, destroy the key:
DELETE FROM guest_encryption_key WHERE guest_id = '{{guest_id}}';

-- The events remain in the store (immutability preserved),
-- but PII fields become unreadable without the key.
-- Projections are then rebuilt with the PII fields nulled out.
```

---

## Crypto-Shredding Support

```sql
CREATE TABLE guest_encryption_key (
    guest_id        UUID PRIMARY KEY,
    encryption_key  BYTEA NOT NULL,                    -- AES-256 key, itself encrypted with master key
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | event_store, event_snapshot |
| Projection Infrastructure | 2 | projection_checkpoint, projection_dead_letter |
| Read Models (Projections) | 9 | v_reservation, v_room_status, v_folio, v_folio_line, v_availability, v_rate, v_guest, v_daily_stats |
| Reference Data | 5 | tenant, property, room, transaction_code, channel |
| Security | 1 | guest_encryption_key |
| **Total** | **19** | Projections are disposable; can be dropped and rebuilt from event store |

---

## Key Design Decisions

1. **Single event_store table as source of truth.** All domain state changes flow through this table. The `stream_id` + `event_version` unique constraint provides optimistic concurrency control -- if two commands try to modify the same aggregate simultaneously, one will fail and can be retried.

2. **JSONB payloads for event data.** Each event type has its own payload schema, but all are stored in a single JSONB column. This avoids needing a separate table per event type while retaining full queryability via GIN indexes and JSONB containment operators.

3. **Snapshots for performance.** Long-lived aggregates (e.g., a folio with hundreds of charges) would be slow to rebuild from scratch. Periodic snapshots capture the computed state at a point in time, so replay only needs to process events after the snapshot.

4. **Projections are disposable.** Every `v_*` table can be dropped and rebuilt by replaying the event store. This means read-model schema changes do not require data migration -- just rebuild. New reporting dimensions can be added by creating a new projector.

5. **Crypto-shredding for GDPR erasure.** Event sourcing's immutability conflicts with GDPR's right to erasure. The solution: encrypt guest PII in event payloads with a per-guest key. To "erase" a guest, delete their encryption key. Events remain intact but PII becomes unreadable.

6. **CloudEvents-aligned metadata.** The `metadata` JSONB field follows CloudEvents conventions: `source`, `subject`, `correlation_id`, `causation_id`, `actor_id`, `actor_ip`. This enables distributed tracing and audit compliance.

7. **AI events as first-class citizens.** Revenue management recommendations, demand forecasts, upsell offers, and their outcomes are stored as events in the same store. This creates a feedback loop: the AI can query its own past recommendations and outcomes to improve future predictions.

8. **Event type taxonomy is hierarchical by domain.** Event types use PascalCase naming (e.g., `ReservationCreated`, `ChargePosted`) and are grouped by domain area. New event types can be added without schema changes.

9. **Projection checkpointing enables resumable processing.** Each projector tracks its last processed event. If a projector crashes, it resumes from its checkpoint rather than replaying all events. The dead letter queue captures failed projections for investigation.

10. **Separation of reference data from event-sourced data.** Slowly-changing configuration (properties, rooms, transaction codes) is managed via traditional CRUD tables. Changes to reference data emit events for audit purposes, but the reference tables are the canonical source. This pragmatic hybrid avoids over-engineering configuration management.

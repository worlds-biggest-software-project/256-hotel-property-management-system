# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Hotel Property Management System · Created: 2026-05-22

## Philosophy

This model combines a conventional relational schema for day-to-day CRUD operations (reservations, folios, housekeeping) with a property-graph layer that models the rich, many-to-many relationships in the hospitality domain: guest-to-guest connections (travel companions, corporate contacts, family groups), guest-to-property affinity (loyalty, preference history across stays), corporate ownership hierarchies (parent company -> subsidiary -> department -> traveller), and channel attribution chains (OTA -> metasearch -> direct conversion). The graph layer lives in PostgreSQL itself, using a `graph_node` / `graph_edge` table pair with JSONB properties, avoiding the need for a separate Neo4j deployment.

Hotels are fundamentally relationship businesses. A PMS that can answer "which guests who stayed with us also stayed at our sister property?", "which corporate accounts generate the most cross-property revenue?", or "which OTA bookings later converted to direct?" has a structural advantage for loyalty programmes, corporate sales, and distribution optimisation. These queries are awkward and expensive in a pure relational model (recursive CTEs, multiple self-joins) but natural in a graph.

The graph layer also unlocks AI use cases that require relationship traversal: guest clustering for personalised marketing, fraud detection (linked payment tokens across guest profiles), corporate travel pattern analysis, and recommendation engines ("guests like you also enjoyed..."). A Fortune 200 hospitality company already uses Neo4j in production to manage 650,000+ rate programmes -- this model brings that capability into the PMS itself.

**Best for:** Teams building a PMS with strong loyalty, CRM, corporate sales, or AI personalisation features where understanding relationships between guests, companies, properties, and bookings is a core differentiator.

**Trade-offs:**

- (+) Natural modelling of guest relationships, corporate hierarchies, and loyalty networks
- (+) Enables "guests like you" recommendations, cross-property affinity, and travel-pattern analysis
- (+) Fraud detection via linked payment tokens and identity graph traversal
- (+) Corporate sales insights: map entire corporate travel programmes across properties
- (+) Graph queries (shortest path, connected components, PageRank-style influence) are trivial
- (-) Additional complexity: two data access patterns (relational + graph) to maintain
- (-) Graph queries in PostgreSQL (recursive CTEs on edge tables) are slower than native graph DBs for deep traversals
- (-) Graph layer requires careful edge-type taxonomy and property-schema discipline
- (-) Developers must understand both relational and graph-thinking paradigms
- (-) Graph data must be kept in sync with relational operational data

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenTravel Alliance (OTA) | Relational operational tables follow OTA 2.0 object model for reservations, rates, and profiles |
| ISO 3166-1/2 | Country and subdivision codes on property and guest tables |
| ISO 4217 | Currency codes on all monetary fields |
| ISO 17442 (LEI) | Legal Entity Identifier stored on company graph nodes for corporate hierarchy traversal |
| PCI DSS v4.0 | Payment tokens stored relationally; graph edges link guests to payment methods without exposing card data |
| GDPR/CCPA | Guest PII in relational tables with consent tracking; graph nodes store only non-PII relationship data or encrypted references |
| W3C RDF/OWL concepts | Edge types and node types follow a lightweight ontology inspired by W3C semantic web patterns |
| HTNG Customer Profile | Guest node properties align with HTNG customer profile specification fields |

---

## Relational Layer (Operational CRUD)

The relational layer handles all transactional operations. It is structurally similar to Model 1 (normalized relational) but leaner, because relationship-heavy data moves to the graph layer.

### Core Tables

```sql
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
    star_rating     SMALLINT CHECK (star_rating BETWEEN 1 AND 5),
    country_code    CHAR(2) NOT NULL,                     -- ISO 3166-1
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',      -- ISO 4217
    address         JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE room_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    code            VARCHAR(20) NOT NULL,
    name            VARCHAR(100) NOT NULL,
    max_occupancy   SMALLINT NOT NULL DEFAULT 2,
    max_adults      SMALLINT NOT NULL DEFAULT 2,
    max_children    SMALLINT NOT NULL DEFAULT 0,
    details         JSONB NOT NULL DEFAULT '{}',
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
    status          VARCHAR(20) NOT NULL DEFAULT 'AVAILABLE'
                    CHECK (status IN ('AVAILABLE','OCCUPIED','OUT_OF_ORDER',
                                      'OUT_OF_SERVICE','BLOCKED')),
    hk_status       VARCHAR(20) NOT NULL DEFAULT 'CLEAN'
                    CHECK (hk_status IN ('CLEAN','DIRTY','INSPECTED',
                                          'IN_PROGRESS','OUT_OF_ORDER')),
    attributes      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, room_number)
);

CREATE INDEX idx_room_property_type ON room(property_id, room_type_id);
CREATE INDEX idx_room_status ON room(property_id, status, hk_status);
```

### Guest Profiles

```sql
CREATE TABLE guest (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(30),
    nationality     CHAR(2),                              -- ISO 3166-1
    language        VARCHAR(10) DEFAULT 'en',
    vip_level       SMALLINT DEFAULT 0,
    profile         JSONB NOT NULL DEFAULT '{}',
    preferences     JSONB NOT NULL DEFAULT '{}',
    total_stays     INT NOT NULL DEFAULT 0,
    total_revenue_cents BIGINT NOT NULL DEFAULT 0,
    last_stay_date  DATE,
    gdpr_consent    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_guest_tenant ON guest(tenant_id);
CREATE INDEX idx_guest_email ON guest(tenant_id, email);
CREATE INDEX idx_guest_name ON guest(tenant_id, last_name, first_name);
CREATE INDEX idx_guest_preferences ON guest USING GIN (preferences jsonb_path_ops);

CREATE TABLE company (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(200) NOT NULL,
    legal_entity_id VARCHAR(20),                          -- ISO 17442 LEI
    tax_id          VARCHAR(50),
    country_code    CHAR(2),
    contact_email   VARCHAR(255),
    payment_terms   SMALLINT DEFAULT 30,
    credit_limit_cents BIGINT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Reservations

```sql
CREATE TABLE reservation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    confirmation_number VARCHAR(20) NOT NULL,
    guest_id        UUID NOT NULL REFERENCES guest(id),
    company_id      UUID REFERENCES company(id),
    room_type_id    UUID NOT NULL REFERENCES room_type(id),
    room_id         UUID REFERENCES room(id),
    rate_plan_id    UUID NOT NULL REFERENCES rate_plan(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'CONFIRMED'
                    CHECK (status IN ('PENDING','CONFIRMED','CHECKED_IN','CHECKED_OUT',
                                      'CANCELLED','NO_SHOW','WAITLISTED')),
    source          VARCHAR(30) NOT NULL DEFAULT 'DIRECT',
    channel_code    VARCHAR(30),
    channel_reservation_id VARCHAR(100),
    check_in_date   DATE NOT NULL,
    check_out_date  DATE NOT NULL,
    adults          SMALLINT NOT NULL DEFAULT 1,
    children        SMALLINT NOT NULL DEFAULT 0,
    total_amount_cents BIGINT,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    details         JSONB NOT NULL DEFAULT '{}',
    checked_in_at   TIMESTAMPTZ,
    checked_out_at  TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, confirmation_number),
    CHECK (check_out_date > check_in_date)
);

CREATE INDEX idx_res_property_dates ON reservation(property_id, check_in_date, check_out_date);
CREATE INDEX idx_res_guest ON reservation(guest_id);
CREATE INDEX idx_res_status ON reservation(property_id, status);
```

### Rate Plans & Availability

```sql
CREATE TABLE rate_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    code            VARCHAR(30) NOT NULL,
    name            VARCHAR(200) NOT NULL,
    rate_type       VARCHAR(20) NOT NULL DEFAULT 'PUBLIC',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    rules           JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    valid_from      DATE,
    valid_to        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, code)
);

CREATE TABLE rate (
    rate_plan_id    UUID NOT NULL REFERENCES rate_plan(id),
    room_type_id    UUID NOT NULL REFERENCES room_type(id),
    stay_date       DATE NOT NULL,
    amount_cents    BIGINT NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    source          VARCHAR(20) NOT NULL DEFAULT 'manual',
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (rate_plan_id, room_type_id, stay_date)
);

CREATE TABLE availability (
    property_id     UUID NOT NULL REFERENCES property(id),
    room_type_id    UUID NOT NULL REFERENCES room_type(id),
    stay_date       DATE NOT NULL,
    total_inventory SMALLINT NOT NULL,
    sold_count      SMALLINT NOT NULL DEFAULT 0,
    restrictions    JSONB NOT NULL DEFAULT '{}',
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (property_id, room_type_id, stay_date)
);
```

### Billing & Folios

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
    billing_info    JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (property_id, folio_number)
);

CREATE TABLE folio_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    folio_id        UUID NOT NULL REFERENCES folio(id),
    line_type       VARCHAR(10) NOT NULL
                    CHECK (line_type IN ('CHARGE','PAYMENT','CREDIT','TAX','ADJUSTMENT')),
    category        VARCHAR(20) NOT NULL,
    description     VARCHAR(255) NOT NULL,
    amount_cents    BIGINT NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    line_date       DATE NOT NULL,
    is_void         BOOLEAN NOT NULL DEFAULT false,
    details         JSONB NOT NULL DEFAULT '{}',
    posted_by       UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_folio_reservation ON folio(reservation_id);
CREATE INDEX idx_folio_guest ON folio(guest_id);
CREATE INDEX idx_folio_line_folio ON folio_line(folio_id, line_date);
```

### Housekeeping

```sql
CREATE TABLE housekeeping_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id     UUID NOT NULL REFERENCES property(id),
    room_id         UUID NOT NULL REFERENCES room(id),
    task_type       VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    priority        SMALLINT NOT NULL DEFAULT 5,
    assigned_to     UUID,
    scheduled_date  DATE NOT NULL,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_hk_property_date ON housekeeping_task(property_id, scheduled_date, status);
```

### Staff & Audit

```sql
CREATE TABLE staff (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    access          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    property_id     UUID,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    actor_id        UUID,
    changes         JSONB NOT NULL DEFAULT '{}',
    context         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);

CREATE TABLE consent_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guest_id        UUID NOT NULL REFERENCES guest(id),
    consent_type    VARCHAR(50) NOT NULL,
    is_granted      BOOLEAN NOT NULL,
    granted_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    source          VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_guest ON consent_record(guest_id);
```

---

## Graph Layer

The graph layer lives in PostgreSQL using two tables: `graph_node` and `graph_edge`. This avoids a separate database while providing graph-style query capabilities via recursive CTEs and the `ltree` extension.

### Graph Schema

```sql
-- ============================================================
-- GRAPH NODES
-- Each node represents an entity in the relationship graph.
-- Nodes reference relational entities via entity_type + entity_id.
-- ============================================================
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    node_type       VARCHAR(30) NOT NULL,
    -- Node types:
    -- 'guest'       -- links to guest table
    -- 'company'     -- links to company table
    -- 'property'    -- links to property table
    -- 'reservation' -- links to reservation table
    -- 'channel'     -- distribution channel
    -- 'payment_method' -- tokenised payment instrument
    -- 'group'       -- guest group (family, corporate team, wedding party)

    entity_id       UUID,                                 -- FK to relational table (contextual)
    label           VARCHAR(200) NOT NULL,                -- human-readable label for display
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Node properties vary by type:
    --
    -- guest node:
    -- {"email": "j.smith@example.com", "vip_level": 2, "total_stays": 5,
    --  "home_country": "US", "segments": ["business", "frequent"]}
    --
    -- company node:
    -- {"lei": "5493001KJTIIGC8Y1R12", "industry": "technology",
    --  "annual_room_nights": 450, "preferred_properties": ["uuid1","uuid2"]}
    --
    -- property node:
    -- {"code": "HLTN-NYC", "city": "New York", "star_rating": 4,
    --  "room_count": 250, "country": "US"}
    --
    -- payment_method node:
    -- {"token_prefix": "tok_visa_4242", "card_brand": "visa",
    --  "card_last_four": "4242", "first_seen": "2025-01-15"}

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gnode_tenant_type ON graph_node(tenant_id, node_type);
CREATE INDEX idx_gnode_entity ON graph_node(entity_id) WHERE entity_id IS NOT NULL;
CREATE INDEX idx_gnode_properties ON graph_node USING GIN (properties jsonb_path_ops);

-- ============================================================
-- GRAPH EDGES
-- Each edge represents a typed, directional relationship
-- between two nodes. Edges carry properties (weight, dates, etc.).
-- ============================================================
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    from_node_id    UUID NOT NULL REFERENCES graph_node(id),
    to_node_id      UUID NOT NULL REFERENCES graph_node(id),
    edge_type       VARCHAR(50) NOT NULL,
    -- Edge types:
    -- 'STAYED_AT'          -- guest -> property (with dates, revenue)
    -- 'BOOKED_VIA'         -- reservation -> channel
    -- 'TRAVELLED_WITH'     -- guest -> guest (same reservation or overlapping dates)
    -- 'WORKS_FOR'          -- guest -> company
    -- 'MANAGES'            -- company -> company (parent-subsidiary)
    -- 'MEMBER_OF'          -- guest -> group
    -- 'REFERRED_BY'        -- guest -> guest (referral tracking)
    -- 'CONVERTED_FROM'     -- reservation(direct) -> reservation(ota) (channel conversion)
    -- 'PAYS_WITH'          -- guest -> payment_method
    -- 'PREFERS'            -- guest -> property (affinity score)
    -- 'PART_OF_GROUP'      -- reservation -> group block
    -- 'UPSOLD'             -- reservation -> reservation (upgrade chain)
    -- 'SISTER_PROPERTY'    -- property -> property

    properties      JSONB NOT NULL DEFAULT '{}',
    -- Edge properties vary by type:
    --
    -- STAYED_AT:
    -- {"check_in": "2026-06-15", "check_out": "2026-06-18", "nights": 3,
    --  "revenue_cents": 75000, "room_type": "KDLX", "satisfaction_score": 4.5}
    --
    -- TRAVELLED_WITH:
    -- {"reservation_id": "uuid...", "relationship": "spouse",
    --  "co_stay_count": 3, "first_co_stay": "2024-12-20"}
    --
    -- WORKS_FOR:
    -- {"department": "Engineering", "title": "VP Engineering",
    --  "travel_authority": true, "booking_approver": false}
    --
    -- PAYS_WITH:
    -- {"usage_count": 12, "first_used": "2024-06-01", "last_used": "2026-05-10",
    --  "total_charged_cents": 340000}
    --
    -- PREFERS:
    -- {"affinity_score": 0.87, "visit_count": 5, "avg_rating": 4.6,
    --  "last_visit": "2026-03-10", "preferred_room_type": "STE"}

    weight          NUMERIC(8,4) DEFAULT 1.0,             -- for graph algorithms (PageRank, etc.)
    valid_from      DATE,                                 -- temporal edge validity
    valid_to        DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gedge_from ON graph_edge(from_node_id, edge_type);
CREATE INDEX idx_gedge_to ON graph_edge(to_node_id, edge_type);
CREATE INDEX idx_gedge_type ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_gedge_properties ON graph_edge USING GIN (properties jsonb_path_ops);

-- Composite index for temporal edge queries
CREATE INDEX idx_gedge_temporal ON graph_edge(edge_type, valid_from, valid_to)
    WHERE valid_from IS NOT NULL;
```

### Graph Sync Triggers

```sql
-- ============================================================
-- Automatic graph node creation when relational entities are created.
-- These triggers keep the graph layer in sync with operational data.
-- ============================================================

-- Example: create a guest graph node when a guest record is inserted
CREATE OR REPLACE FUNCTION sync_guest_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties)
    VALUES (
        NEW.tenant_id,
        'guest',
        NEW.id,
        NEW.first_name || ' ' || NEW.last_name,
        jsonb_build_object(
            'email', NEW.email,
            'vip_level', NEW.vip_level,
            'nationality', NEW.nationality,
            'total_stays', NEW.total_stays
        )
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_guest_graph_sync
    AFTER INSERT ON guest
    FOR EACH ROW
    EXECUTE FUNCTION sync_guest_to_graph();

-- Example: create STAYED_AT edge when a reservation checks out
CREATE OR REPLACE FUNCTION sync_checkout_to_graph()
RETURNS TRIGGER AS $$
DECLARE
    guest_node_id UUID;
    property_node_id UUID;
BEGIN
    IF NEW.status = 'CHECKED_OUT' AND OLD.status = 'CHECKED_IN' THEN
        SELECT id INTO guest_node_id FROM graph_node
            WHERE entity_id = NEW.guest_id AND node_type = 'guest';
        SELECT id INTO property_node_id FROM graph_node
            WHERE entity_id = NEW.property_id AND node_type = 'property';

        IF guest_node_id IS NOT NULL AND property_node_id IS NOT NULL THEN
            INSERT INTO graph_edge (
                tenant_id, from_node_id, to_node_id, edge_type, properties,
                valid_from, valid_to, weight
            ) VALUES (
                (SELECT tenant_id FROM property WHERE id = NEW.property_id),
                guest_node_id,
                property_node_id,
                'STAYED_AT',
                jsonb_build_object(
                    'reservation_id', NEW.id,
                    'check_in', NEW.check_in_date,
                    'check_out', NEW.check_out_date,
                    'nights', NEW.check_out_date - NEW.check_in_date,
                    'revenue_cents', NEW.total_amount_cents,
                    'room_type', (SELECT code FROM room_type WHERE id = NEW.room_type_id),
                    'source', NEW.source
                ),
                NEW.check_in_date,
                NEW.check_out_date,
                (NEW.check_out_date - NEW.check_in_date)::numeric  -- weight by stay length
            );
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_checkout_graph_sync
    AFTER UPDATE ON reservation
    FOR EACH ROW
    WHEN (NEW.status = 'CHECKED_OUT' AND OLD.status = 'CHECKED_IN')
    EXECUTE FUNCTION sync_checkout_to_graph();
```

---

## Example Graph Queries

### Find all guests who have stayed at the same property as a given guest

```sql
-- "Who else stayed at properties where Guest X has stayed?"
WITH guest_properties AS (
    SELECT DISTINCT e.to_node_id AS property_node_id
    FROM graph_edge e
    JOIN graph_node gn ON gn.id = e.from_node_id
    WHERE gn.entity_id = '{{guest_id}}'
      AND gn.node_type = 'guest'
      AND e.edge_type = 'STAYED_AT'
)
SELECT
    g.id AS guest_id,
    g.first_name,
    g.last_name,
    g.total_stays,
    COUNT(DISTINCT gp.property_node_id) AS shared_properties
FROM graph_edge e
JOIN graph_node gn ON gn.id = e.from_node_id AND gn.node_type = 'guest'
JOIN guest g ON g.id = gn.entity_id
JOIN guest_properties gp ON gp.property_node_id = e.to_node_id
WHERE e.edge_type = 'STAYED_AT'
  AND gn.entity_id != '{{guest_id}}'
GROUP BY g.id, g.first_name, g.last_name, g.total_stays
ORDER BY shared_properties DESC, g.total_stays DESC
LIMIT 20;
```

### Map a corporate travel hierarchy (recursive)

```sql
-- "Show the full corporate hierarchy from parent company down to individual travellers"
WITH RECURSIVE corp_tree AS (
    -- Start from the parent company node
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.node_type,
        gn.properties,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.entity_id = '{{parent_company_id}}'
      AND gn.node_type = 'company'

    UNION ALL

    -- Traverse MANAGES edges (company -> subsidiary) and WORKS_FOR edges (guest -> company)
    SELECT
        child.id,
        child.label,
        child.node_type,
        child.properties,
        ct.depth + 1,
        ct.path || child.id
    FROM corp_tree ct
    JOIN graph_edge e ON (
        (e.from_node_id = ct.node_id AND e.edge_type = 'MANAGES')
        OR
        (e.to_node_id = ct.node_id AND e.edge_type = 'WORKS_FOR')
    )
    JOIN graph_node child ON child.id = CASE
        WHEN e.edge_type = 'MANAGES' THEN e.to_node_id
        WHEN e.edge_type = 'WORKS_FOR' THEN e.from_node_id
    END
    WHERE child.id != ALL(ct.path)  -- prevent cycles
      AND ct.depth < 10             -- max depth guard
)
SELECT
    depth,
    REPEAT('  ', depth) || label AS hierarchy,
    node_type,
    properties->>'department' AS department,
    properties->>'annual_room_nights' AS room_nights
FROM corp_tree
ORDER BY path;
```

### Detect linked identities via shared payment methods

```sql
-- "Find all guest profiles linked by the same payment token (potential duplicates or fraud)"
WITH payment_links AS (
    SELECT
        e1.from_node_id AS guest_node_1,
        e2.from_node_id AS guest_node_2,
        pm.label AS payment_method,
        pm.properties->>'card_last_four' AS card_last_four
    FROM graph_edge e1
    JOIN graph_edge e2 ON e1.to_node_id = e2.to_node_id
        AND e1.from_node_id < e2.from_node_id  -- avoid duplicates
    JOIN graph_node pm ON pm.id = e1.to_node_id AND pm.node_type = 'payment_method'
    WHERE e1.edge_type = 'PAYS_WITH'
      AND e2.edge_type = 'PAYS_WITH'
)
SELECT
    g1.first_name || ' ' || g1.last_name AS guest_1,
    g1.email AS email_1,
    g2.first_name || ' ' || g2.last_name AS guest_2,
    g2.email AS email_2,
    pl.payment_method,
    pl.card_last_four
FROM payment_links pl
JOIN graph_node gn1 ON gn1.id = pl.guest_node_1
JOIN graph_node gn2 ON gn2.id = pl.guest_node_2
JOIN guest g1 ON g1.id = gn1.entity_id
JOIN guest g2 ON g2.id = gn2.entity_id;
```

### Calculate guest affinity scores across properties

```sql
-- "Rank properties by affinity for a given guest based on stay history and preferences"
SELECT
    p.name AS property_name,
    p.code AS property_code,
    COUNT(*) AS stay_count,
    SUM((e.properties->>'nights')::int) AS total_nights,
    SUM((e.properties->>'revenue_cents')::bigint) AS total_revenue_cents,
    AVG(e.weight) AS avg_stay_weight,
    -- Composite affinity score
    ROUND(
        (COUNT(*)::numeric * 0.3 +
         SUM((e.properties->>'nights')::int)::numeric * 0.3 +
         LOG(SUM((e.properties->>'revenue_cents')::bigint)::numeric + 1) * 0.4
        ), 2
    ) AS affinity_score
FROM graph_edge e
JOIN graph_node gn ON gn.id = e.from_node_id AND gn.node_type = 'guest'
JOIN graph_node pn ON pn.id = e.to_node_id AND pn.node_type = 'property'
JOIN property p ON p.id = pn.entity_id
WHERE gn.entity_id = '{{guest_id}}'
  AND e.edge_type = 'STAYED_AT'
GROUP BY p.id, p.name, p.code
ORDER BY affinity_score DESC;
```

### Find channel conversion paths (OTA to direct)

```sql
-- "Which OTA-sourced guests later booked direct? What was the conversion path?"
SELECT
    g.first_name || ' ' || g.last_name AS guest_name,
    r_ota.confirmation_number AS ota_booking,
    r_ota.source AS ota_source,
    r_ota.channel_code AS ota_channel,
    r_ota.check_in_date AS ota_stay_date,
    r_direct.confirmation_number AS direct_booking,
    r_direct.check_in_date AS direct_stay_date,
    r_direct.check_in_date - r_ota.check_out_date AS days_to_convert
FROM graph_edge e
JOIN graph_node rn_from ON rn_from.id = e.from_node_id AND rn_from.node_type = 'reservation'
JOIN graph_node rn_to ON rn_to.id = e.to_node_id AND rn_to.node_type = 'reservation'
JOIN reservation r_direct ON r_direct.id = rn_from.entity_id
JOIN reservation r_ota ON r_ota.id = rn_to.entity_id
JOIN guest g ON g.id = r_direct.guest_id
WHERE e.edge_type = 'CONVERTED_FROM'
  AND r_direct.source = 'DIRECT'
  AND r_ota.source = 'OTA'
ORDER BY days_to_convert ASC;
```

---

## AI-Specific Graph Features

```sql
-- ============================================================
-- GUEST SEGMENTS (computed via graph clustering algorithms)
-- ============================================================
CREATE TABLE guest_segment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,                -- 'high_value_business', 'leisure_family'
    description     TEXT,
    criteria        JSONB NOT NULL,
    -- {
    --   "min_stays": 3,
    --   "min_revenue_cents": 100000,
    --   "preferred_sources": ["DIRECT", "CORPORATE"],
    --   "graph_centrality_min": 0.5
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE guest_segment_membership (
    guest_id        UUID NOT NULL REFERENCES guest(id),
    segment_id      UUID NOT NULL REFERENCES guest_segment(id),
    score           NUMERIC(6,4) NOT NULL DEFAULT 1.0,    -- membership strength
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (guest_id, segment_id)
);

-- ============================================================
-- RECOMMENDATION EDGES (AI-generated relationship suggestions)
-- ============================================================
-- These are stored as regular graph edges with edge_type = 'AI_RECOMMENDS'
-- and properties containing the recommendation details:
-- {
--   "recommendation_type": "room_upgrade",
--   "from_room_type": "QSTD",
--   "to_room_type": "KDLX",
--   "confidence": 0.82,
--   "model_version": "upsell-v1.3",
--   "reason": "Guest prefers king beds and has accepted 3 of 4 prior upgrades"
-- }
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenant, property |
| Room Inventory | 2 | room_type, room |
| Guest & Company | 2 | guest, company |
| Rate Management | 2 | rate_plan, rate |
| Availability | 1 | availability |
| Reservations | 1 | reservation |
| Billing & Folios | 2 | folio, folio_line |
| Housekeeping | 1 | housekeeping_task |
| Staff & Security | 1 | staff |
| Audit & Compliance | 2 | audit_log, consent_record |
| Graph Layer | 2 | graph_node, graph_edge |
| AI Features | 2 | guest_segment, guest_segment_membership |
| **Total** | **20** | Plus 2 graph tables that model all relationships |

---

## Key Design Decisions

1. **Graph in PostgreSQL, not a separate database.** Using `graph_node` and `graph_edge` tables with JSONB properties avoids the operational complexity of running Neo4j or Amazon Neptune alongside PostgreSQL. Recursive CTEs handle traversals up to 5-10 hops efficiently; for deeper traversals or graph-algorithm-heavy workloads, the same schema can be mirrored to a dedicated graph DB later.

2. **Graph nodes reference relational entities.** Each graph node has an `entity_id` that links to the corresponding relational table (`guest`, `company`, `property`, `reservation`). This avoids data duplication: the graph layer stores relationship structure and computed properties, while the relational layer remains the system of record for operational data.

3. **Trigger-based graph sync.** PostgreSQL triggers automatically create graph nodes when relational entities are inserted and create graph edges when state transitions occur (e.g., checkout creates a `STAYED_AT` edge). This keeps the graph layer consistent without requiring application-level dual-write logic.

4. **Edge types as a controlled vocabulary.** The `edge_type` column uses a defined set of relationship types (`STAYED_AT`, `TRAVELLED_WITH`, `WORKS_FOR`, etc.) that form a lightweight ontology. New edge types can be added without schema changes, but the vocabulary is documented and enforced at the application layer.

5. **Temporal edges with `valid_from` / `valid_to`.** Many relationships in hospitality are temporal: a guest's employment at a company, a stay at a property, a corporate rate agreement. Temporal edges enable time-windowed graph queries ("who worked for Acme Corp in Q1 2026?") and historical relationship analysis.

6. **Edge weight for graph algorithms.** The `weight` column enables weighted graph algorithms: stay length for affinity scoring, revenue for value ranking, recency for recommendation freshness. Weights are computed at edge creation time by triggers and can be recalculated in batch.

7. **Payment method as a graph node, not a relational table.** Modelling payment instruments as graph nodes with `PAYS_WITH` edges enables identity-linking queries (find all guest profiles sharing a payment token) that would require complex self-joins in a relational model. This is the foundation for duplicate detection and fraud prevention.

8. **Guest segmentation as a graph-computed feature.** The `guest_segment` and `guest_segment_membership` tables store the output of graph clustering algorithms (community detection, centrality analysis). Segments are computed periodically in batch and used by the AI personalisation engine for targeted offers and room assignments.

9. **Channel conversion tracking via `CONVERTED_FROM` edges.** When a guest who previously booked via an OTA later books directly, a `CONVERTED_FROM` edge links the two reservations. This enables distribution ROI analysis: "which OTA channels produce guests who convert to direct?" -- a question that is nearly impossible to answer in a relational model without complex multi-pass queries.

10. **Relational layer kept lean.** Because the graph layer handles relationship complexity, the relational layer uses the hybrid JSONB pattern (similar to Model 3) rather than full normalisation. This keeps the operational schema at ~18 relational tables while the 2 graph tables absorb all relationship modelling. Total table count remains low without sacrificing query power.

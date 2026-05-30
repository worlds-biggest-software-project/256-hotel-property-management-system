# Hotel Property Management System — Phased Development Plan

> Project: 256-hotel-property-management-system · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into a concrete, phased build. The product is an **AI-native, self-hostable, multi-tenant, multi-property hotel PMS** unifying reservations, front desk, housekeeping, rate/inventory distribution, billing, and embedded AI revenue management.

---

## Core Requirements (synthesised)

- **What it does**: A cloud-native PMS for independent hotels and boutique groups. Manages reservations, front-desk check-in/out, room inventory and availability, rate plans, housekeeping, folios/billing/payments, OTA channel distribution, a direct booking engine, operational reporting (occupancy, ADR, RevPAR), and an embedded AI dynamic-pricing/forecasting engine. Exposes an OpenAPI 3.1 REST API with OAuth 2.0, a webhook framework, and (later) an MCP server for AI agents.
- **Who uses it**: Independent hotel owners/operators, front-office managers (check-in/out, folios), housekeeping staff (mobile task lists), revenue managers (pricing/forecasting), boutique-group operations/IT directors (multi-property), and ISV developers (API/integrations).
- **Key differentiators**: (1) AI revenue management built into the core, not a third-party RMS bolt-on; (2) transparent open-source/self-hostable deployment with enterprise depth; (3) MCP-ready architecture for AI agents; (4) prescriptive analytics ("RevPAR is 12% below comp set — here's why").
- **MVP scope** (features.md "Must-have"): reservations, availability engine with overbooking prevention, front desk, housekeeping, billing/folios/payments with PCI-safe tokenisation, basic direct booking engine, core reporting.
- **Should-have (v1.1)**: AI dynamic pricing + forecasting, guest CRM, automated guest comms, multi-property dashboard, staff mobile app, full public REST API + OAuth, webhook/marketplace framework.
- **Backlog**: AI upselling, staff scheduling, automated PCI/GDPR posture reporting, NL-query BI, digital keys, MCP server.
- **Deployment**: Cloud-native SaaS **and** self-hostable via Docker Compose (first-class).
- **Integration surface**: OTA channel managers (Booking.com / Expedia / Airbnb via channel-manager intermediaries, OTA 2.0 JSON / HTNG-aligned models), payment gateway (Stripe + tokenisation vault), LLM provider (pricing/comms/forecasting), webhooks out, MCP server.
- **Standards** (standards.md): OpenAPI 3.1, OAuth 2.0 (RFC 6749) + JWT (RFC 7519) + PKCE (RFC 9700), OTA 2.0 JSON / HTNG Express for distribution, PCI DSS v4.0 (tokenisation, no PAN at rest), GDPR/CCPA (consent, erasure, retention), OWASP API Security Top 10, ISO 4217 / ISO 3166 / IANA TZ / BCP 47 code vocabularies, RFC 5545 (iCal export), MCP.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | **Python 3.12** | The product's defining advantage is embedded AI (pricing, forecasting, guest comms). Python has the richest ML/forecasting (statsmodels, Prophet, scikit-learn) and LLM SDK ecosystem, and is first-class for MCP servers. The non-AI CRUD surface is well served by modern async Python. |
| API framework | **FastAPI** | Generates OpenAPI 3.1 automatically (a hard requirement from standards.md and the README's API-first positioning), native Pydantic validation maps directly to JSON Schema 2020-12, async I/O suits OTA/payment/LLM calls, and built-in OAuth2 + JWT support. |
| ASGI server | **Uvicorn** (behind Gunicorn workers in prod) | Standard production ASGI stack for FastAPI. |
| Database | **PostgreSQL 16** | Required for JSONB (the hybrid data model), `gen_random_uuid()`, GIN indexes, Row-Level Security for multi-tenancy, range types/exclusion constraints for overbooking prevention, and `tstzrange` temporal queries. Financial accuracy needs ACID transactions. |
| ORM / migrations | **SQLAlchemy 2.0 (async) + Alembic** | Mature async ORM with typed models; Alembic gives versioned, reviewable migrations (needed because the relational core is stable and migration discipline matters for a financial system). |
| Data model | **Hybrid Relational + JSONB (data-model-suggestion-3)**, with the normalised financial/audit core from suggestion-1 | Suggestion 3 ships fastest, supports multi-jurisdiction tax/registration and schema-fluid AI output in JSONB, and matches the Apaleo "lean core + extensible config" pattern. Folios, charges, payments, and audit use the strict relational shape from suggestion-1 for financial integrity. |
| Task queue / async jobs | **Celery + Redis** | OTA sync, payment capture, LLM calls, nightly audit, AI pricing recompute, and webhook delivery are long-running/retriable. Celery is the de-facto Python task system; Redis doubles as broker and cache. |
| Cache / rate-limit store | **Redis 7** | Availability lookups, OAuth token introspection cache, API rate limiting (OWASP API4), Celery broker/result backend. |
| Frontend | **Next.js 15 (App Router) + TypeScript + shadcn/ui + Tailwind** | Two surfaces needed: a desktop staff/admin dashboard and a mobile-friendly housekeeping/front-desk PWA. Next.js server components keep the dashboard fast; the same app ships a responsive PWA for staff mobile. Consumes the same public REST API. |
| Booking engine widget | **Preact + Vite (embeddable bundle)** | The embeddable direct-booking widget must be a tiny, dependency-light script droppable into any hotel website; Preact keeps the bundle small. A standalone booking page reuses the Next.js app. |
| LLM access | **Provider-agnostic gateway (LiteLLM)** | Pricing rationale, guest comms, and NL-BI must not hard-bind to one vendor (self-hosters may use local models). LiteLLM gives one interface over OpenAI/Anthropic/local. |
| Forecasting | **statsmodels + scikit-learn (+ optional Prophet)** | Demand forecasting and booking-pace models are classical time-series/regression problems; these libraries are deterministic, testable, and run without external API calls. |
| Payments | **Stripe (gateway) + tokenisation only** | PCI DSS v4.0: never store PANs. Stripe handles card capture (Stripe.js/Elements), the PMS stores only `payment_token` + gateway references. Gateway is abstracted behind a `PaymentGateway` interface so self-hosters can swap. |
| MCP server | **Official Python MCP SDK** | Exposes reservations, availability, rates, and operational actions as MCP tools/resources (standards.md note 3; README roadmap). |
| Auth | **OAuth 2.0 (Authorization Code + PKCE, Client Credentials) + JWT access tokens** | RFC 6749/7519/9700. Staff login uses Authorization Code + PKCE; machine/ISV integrations use Client Credentials with scopes. WebAuthn for staff/kiosk added later. |
| Containerisation | **Docker + Docker Compose** | Self-hostable deployment is first-class (README); Compose bundles API, worker, Postgres, Redis, and frontend for one-command self-host. |
| Testing | **pytest + pytest-asyncio + httpx + Testcontainers; Playwright (E2E UI); k6 (load)** | pytest is the Python standard; Testcontainers spins real Postgres/Redis for integration tests; Playwright drives the dashboard/widget; k6 validates availability-engine throughput. |
| Code quality | **Ruff (lint+format) + mypy (strict) + pre-commit** | Ruff replaces flake8/black/isort; mypy enforces types on a financial codebase. Frontend uses ESLint + Prettier + `tsc --noEmit`. |
| Package management | **uv (Python) + pnpm (frontend)** | uv is the fastest, reproducible Python resolver/installer; pnpm for the JS workspace. |
| Schema validation (wire) | **Pydantic v2** | Request/response models double as the JSON Schema 2020-12 components in the OpenAPI doc. |
| Observability | **OpenTelemetry + structured JSON logging + Prometheus metrics** | Required for OTA-sync health, payment tracing, and SLA monitoring; OTEL exports to any backend. |

### Project Structure

```
hotel-pms/
├── README.md
├── docker-compose.yml                # api + worker + postgres + redis + frontend
├── docker-compose.dev.yml            # hot-reload overrides
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.frontend
├── .env.example
├── pyproject.toml                    # uv project, ruff/mypy/pytest config
├── alembic.ini
├── Makefile                          # dev shortcuts (up, test, lint, migrate, seed)
│
├── backend/
│   ├── pms/
│   │   ├── __init__.py
│   │   ├── main.py                   # FastAPI app factory, router registration
│   │   ├── config.py                 # Pydantic Settings (env-driven)
│   │   ├── db/
│   │   │   ├── base.py               # async engine, session, Base
│   │   │   ├── rls.py                # tenant RLS session helpers
│   │   │   └── migrations/           # alembic versions
│   │   ├── models/                   # SQLAlchemy ORM (one module per domain)
│   │   │   ├── tenant.py  property.py  room.py  rate.py
│   │   │   ├── guest.py  reservation.py  folio.py
│   │   │   ├── housekeeping.py  channel.py  staff.py  audit.py
│   │   ├── schemas/                  # Pydantic request/response models
│   │   ├── api/
│   │   │   ├── deps.py               # auth, tenant, db, pagination deps
│   │   │   ├── v1/
│   │   │   │   ├── reservations.py  frontdesk.py  rooms.py  rates.py
│   │   │   │   ├── availability.py  guests.py  folios.py  payments.py
│   │   │   │   ├── housekeeping.py  channels.py  reports.py
│   │   │   │   ├── pricing.py  webhooks.py  auth.py  oauth.py
│   │   ├── services/                 # business logic (framework-agnostic)
│   │   │   ├── availability.py       # ARI engine, overbooking guard
│   │   │   ├── reservation.py  folio.py  payment.py  housekeeping.py
│   │   │   ├── reporting.py  pricing/  forecasting/  comms/
│   │   ├── integrations/
│   │   │   ├── channel/              # OTA channel-manager adapters (base + impls)
│   │   │   ├── payment/              # PaymentGateway interface + Stripe impl
│   │   │   ├── llm.py                # LiteLLM wrapper
│   │   │   └── webhooks.py           # outbound delivery
│   │   ├── workers/
│   │   │   ├── celery_app.py
│   │   │   └── tasks/                # sync, pricing, comms, webhook, nightaudit
│   │   ├── security/                 # oauth, jwt, password, rbac, encryption
│   │   ├── mcp/                      # MCP server (later phase)
│   │   └── utils/                    # money, dates, ids, pagination
│   └── tests/
│       ├── conftest.py               # Testcontainers fixtures, factories
│       ├── unit/  integration/  e2e/  fixtures/
│
├── frontend/                         # Next.js dashboard + staff PWA
│   ├── package.json
│   ├── app/  components/  lib/api/   # generated typed client from OpenAPI
│   └── tests/                        # Playwright
│
└── widget/                           # embeddable booking widget (Preact + Vite)
    ├── package.json  src/  dist/
```

The structure is grouped by concern (models / schemas / api / services / integrations / workers), so every phase adds modules without restructuring.

---

## Phase 1: Foundation, Multi-Tenancy & Core Schema

### Purpose
Establish the runnable skeleton: project tooling, configuration, the database with the hybrid schema and tenant isolation, the FastAPI app factory, health checks, and the test harness. After this phase the system boots, connects to Postgres/Redis, serves a health endpoint, runs migrations, and enforces tenant isolation at the database level — the foundation every other phase builds on.

### Tasks

#### 1.1 — Project scaffolding & tooling

**What**: Initialise the uv project, Docker Compose stack, linting/typing, and CI-ready test commands.

**Design**:
- `pyproject.toml` with deps: `fastapi`, `uvicorn[standard]`, `sqlalchemy[asyncio]`, `asyncpg`, `alembic`, `pydantic`, `pydantic-settings`, `celery`, `redis`, `httpx`; dev: `pytest`, `pytest-asyncio`, `testcontainers`, `ruff`, `mypy`.
- `pms/config.py` — Pydantic `Settings`:
  ```python
  class Settings(BaseSettings):
      database_url: str
      redis_url: str = "redis://redis:6379/0"
      jwt_secret: str
      jwt_access_ttl_seconds: int = 900
      jwt_refresh_ttl_seconds: int = 1_209_600
      environment: Literal["dev", "test", "prod"] = "dev"
      stripe_secret_key: str | None = None
      llm_provider: str = "openai"
      model_config = SettingsConfigDict(env_file=".env", env_prefix="PMS_")
  ```
- `docker-compose.yml`: services `api`, `worker`, `postgres:16`, `redis:7`, `frontend`; named volume for Postgres; healthchecks gating `api` on Postgres readiness.
- `Makefile`: `up`, `down`, `test`, `lint`, `typecheck`, `migrate`, `revision`, `seed`.
- Ruff (line length 100, select `E,F,I,UP,B,ASYNC,S`) and mypy `strict = true`.

**Testing**:
- `Unit: Settings loads from env → fields populated, defaults applied`
- `Unit: missing PMS_DATABASE_URL → ValidationError naming the field`
- `Integration: docker compose up → /health returns 200 within 30s`
- `CI: ruff check and mypy pass on empty skeleton`

#### 1.2 — Database base, async session, and RLS multi-tenancy

**What**: Async SQLAlchemy engine/session, declarative `Base`, and tenant Row-Level Security.

**Design**:
- `db/base.py`: async engine from `settings.database_url`, `async_sessionmaker`, `DeclarativeBase` subclass `Base` with shared mixins:
  ```python
  class TimestampMixin:
      created_at: Mapped[datetime] = mapped_column(server_default=func.now())
      updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())
  class UUIDMixin:
      id: Mapped[uuid.UUID] = mapped_column(primary_key=True, server_default=text("gen_random_uuid()"))
  ```
- `db/rls.py`: `set_tenant(session, tenant_id)` issues `SET LOCAL app.current_tenant = :tid`; RLS policies on tenant-scoped tables use `USING (tenant_id = current_setting('app.current_tenant')::uuid)`.
- Decision: enable Postgres RLS as defence-in-depth; application also filters by `tenant_id` (belt and braces, mitigates OWASP API1/API5 BOLA).

**Testing**:
- `Integration (Testcontainers): insert two tenants' rows → query under tenant A returns only A's rows`
- `Integration: query with no tenant set → RLS returns zero rows on tenant-scoped table`
- `Unit: set_tenant emits SET LOCAL with the given UUID`

#### 1.3 — Core schema migrations (hybrid model)

**What**: Alembic migrations creating identity, inventory, rate, guest, reservation, folio, housekeeping, channel, staff, audit, and tax tables per the hybrid design.

**Design**:
- Adopt **data-model-suggestion-3** as the base shape with these decisions carried from suggestion-1: all money as `BIGINT` cents + `currency_code CHAR(3)` (ISO 4217); UUID PKs everywhere; `tenant_id` on shared tables, `property_id` on operational tables.
- JSONB "Zone 2" columns: `property.settings`, `room_type.attributes`, `guest.profile_ext`, `reservation.metadata`, `tax_rule.definition` (jurisdiction-specific tax/registration rules), `rate.ai_signals` (pricing-model output), `pricing_recommendation.payload`.
- Overbooking-critical constraint on `availability`: `UNIQUE (property_id, room_type_id, stay_date)` plus a `CHECK (sold_count <= total_inventory + overbooking_allowance)`.
- Enums implemented as `VARCHAR` + `CHECK` (not native PG enums) so values evolve without enum-migration pain — matches suggestion-1 status sets (`reservation.status`, `room.status`, `housekeeping_task.status`, etc.).
- GIN indexes on each JSONB column actively queried (`property.settings`, `tax_rule.definition`).
- Tables (≈30): `tenant, property, room_type, room, amenity, room_type_amenity, rate_plan, rate_plan_room_type, rate, availability, cancellation_policy, guest, guest_preference, company, travel_agent, reservation_group, reservation, reservation_guest, reservation_daily_rate, folio, charge, transaction_code, payment, invoice, housekeeping_task, channel, property_channel, channel_rate_mapping, channel_sync_log, staff, role, permission, role_permission, staff_role, audit_log, consent_record, tax_rule`.

**Testing**:
- `Integration: alembic upgrade head → all tables exist; downgrade base → all dropped cleanly`
- `Integration: insert availability with sold_count > total + allowance → CHECK violation raised`
- `Integration: duplicate (property_id, confirmation_number) → UNIQUE violation`
- `Integration: GIN index used for property.settings @> query (EXPLAIN shows index scan)`

#### 1.4 — Test harness & data factories

**What**: pytest fixtures providing a migrated Postgres + Redis via Testcontainers, transactional rollback per test, and factory helpers.

**Design**:
- `conftest.py`: session-scoped Testcontainers Postgres/Redis; function-scoped session wrapped in a nested transaction rolled back after each test.
- Factories: `make_tenant`, `make_property`, `make_room_type`, `make_room`, `make_guest`, `make_reservation` returning persisted ORM objects with sensible defaults.
- `app_client` fixture: `httpx.AsyncClient` bound to the FastAPI app with dependency overrides for db/auth.

**Testing**:
- `Meta: two tests creating the same email don't collide (rollback isolation verified)`
- `Meta: app_client GET /health → 200`

---

## Phase 2: Inventory, Rates & the Availability Engine

### Purpose
Build the ARI (Availability, Rates, Inventory) core — the heart of any PMS. This phase delivers room-type/room CRUD, rate plans with per-day rates, and a real-time availability engine that prevents overbooking. Everything downstream (reservations, booking engine, OTA sync, AI pricing) reads and writes through this engine.

### Tasks

#### 2.1 — Room type & room management

**What**: CRUD for `room_type`, `room`, and amenities, scoped to a property.

**Design**:
- Pydantic schemas `RoomTypeCreate/Update/Read`, `RoomCreate/Update/Read` mirroring the ORM with JSONB `attributes` typed as `dict[str, Any]`.
- Endpoints (all under `/v1/properties/{property_id}`):
  - `POST /room-types` → 201 `RoomTypeRead`
  - `GET /room-types` (paginated, RFC 8288 `Link` header) → `Page[RoomTypeRead]`
  - `PATCH /room-types/{id}`, `DELETE` (soft via `is_active=false`)
  - `POST /rooms`, `GET /rooms?status=&room_type_id=`, `PATCH /rooms/{id}`
- Pagination dep: cursor on `(sort_order, id)`; returns `Link: <...>; rel="next"`.

**Testing**:
- `Unit: RoomTypeCreate rejects max_occupancy=0 (>=1 validator)`
- `Integration: POST room-type then GET list → item present, tenant-scoped`
- `Integration: create room referencing room_type from another property → 422/404`
- `Integration: pagination → Link rel=next present when results exceed page size`

#### 2.2 — Rate plans & daily rates

**What**: Rate plan CRUD, room-type applicability, per-day `rate` rows, and cancellation policies.

**Design**:
- `rate_plan` with `rate_type` enum, booking/stay windows, `min/max_stay`, FK to `cancellation_policy`.
- Bulk daily-rate upsert: `PUT /room-types/{id}/rates` body `{ "rate_plan_id", "rates": [{stay_date, amount_cents}] }` → idempotent upsert on `(rate_plan_id, room_type_id, stay_date)`.
- Rate lookup service `get_rate(room_type_id, rate_plan_id, date) -> Money`, falling back to `room_type.base_rate_cents` when no daily rate exists.

**Testing**:
- `Unit: rate plan with stay_end < stay_start → ValidationError`
- `Integration: bulk upsert 30 rates then re-upsert 5 → 5 updated, 25 unchanged, no duplicates`
- `Unit: get_rate falls back to base_rate when no daily row`

#### 2.3 — Availability engine & overbooking prevention

**What**: The authoritative availability calculator and inventory mutation guard.

**Design**:
- `services/availability.py`:
  ```python
  async def get_availability(property_id, room_type_id, start: date, end: date) -> list[DayAvailability]:
      """For each night in [start, end): total_inventory - sold_count + overbooking_allowance,
         honouring is_closed (stop-sell), cta, ctd, min_stay, max_stay."""

  async def reserve_inventory(session, property_id, room_type_id, start, end, qty=1) -> None:
      """Atomically increment sold_count for each night via SELECT ... FOR UPDATE on the
         availability rows; raise NoAvailabilityError if any night would exceed
         total_inventory + overbooking_allowance. Auto-create availability rows from
         room_type room count if missing."""

  async def release_inventory(session, property_id, room_type_id, start, end, qty=1) -> None:
  ```
- Concurrency: row locks + the `CHECK` constraint from 1.3 make double-booking impossible even under race.
- `DayAvailability` dataclass: `stay_date, available, closed, cta, ctd, min_stay, max_stay, lowest_rate_cents`.
- Endpoint `GET /v1/properties/{id}/availability?start=&end=&room_type_id=&guests=` → list of `DayAvailability` (also feeds the booking widget).

**Testing**:
- `Unit: get_availability subtracts sold from total + allowance per night`
- `Integration: reserve 1 room when 1 left → success; concurrent second reserve → NoAvailabilityError`
- `Integration (real DB, parallel): 10 concurrent reserves on 3-room inventory → exactly 3 succeed`
- `Unit: stop-sell (is_closed) night → available reported 0 regardless of inventory`
- `Integration: release_inventory after reserve → sold_count restored`

---

## Phase 3: Reservations & Front Desk

### Purpose
Deliver the central operational workflow: creating reservations against the availability engine, managing their lifecycle, and running the front desk (check-in, room assignment, check-out). This is the primary daily surface for hotel staff and the core value-delivery phase.

### Tasks

#### 3.1 — Reservation lifecycle

**What**: Create/modify/cancel reservations with a strict status state machine, holding inventory atomically.

**Design**:
- State machine: `PENDING → CONFIRMED → CHECKED_IN → CHECKED_OUT`; side paths `CONFIRMED → CANCELLED`, `CONFIRMED → NO_SHOW`, `PENDING → WAITLISTED`. Transitions validated by `services/reservation.py::transition(res, target)`.
- `create_reservation`: in one DB transaction — validate dates, call `reserve_inventory`, generate `confirmation_number` (property-prefixed, collision-checked), write `reservation` + `reservation_daily_rate` rows from rate lookup, open a `GUEST` folio, post room charges (deferred to Phase 5 night audit but folio created now).
- Cancellation applies `cancellation_policy` to compute penalty and releases inventory.
- Endpoints: `POST /v1/properties/{id}/reservations`, `GET` (filters: status, dates, guest, confirmation_number), `GET /{rid}`, `PATCH /{rid}` (date/room-type change re-runs availability), `POST /{rid}/cancel`.

**Testing**:
- `Integration: create reservation → inventory decremented, folio opened, daily rates written`
- `Unit: transition CHECKED_OUT → CONFIRMED → InvalidTransitionError`
- `Integration: cancel within deadline → no penalty, inventory released`
- `Integration: cancel after deadline (FIRST_NIGHT policy) → penalty charge posted`
- `Integration: modify dates to unavailable range → 409, original reservation unchanged`

#### 3.2 — Front desk: check-in & room assignment

**What**: Assign a physical room and check a guest in.

**Design**:
- `POST /v1/reservations/{rid}/assign-room` body `{room_id}` — validates room belongs to the reservation's room_type and property, is `AVAILABLE`, and housekeeping_status is `CLEAN`/`INSPECTED` (configurable override with permission `room.assign_dirty`).
- `POST /v1/reservations/{rid}/check-in` — requires assigned room; sets `status=CHECKED_IN`, `checked_in_at`, room `status=OCCUPIED`; emits `reservation.checked_in` event (webhooks, Phase 9).
- Walk-in helper `POST /v1/properties/{id}/walk-in` creates guest + reservation + assignment + check-in in one call.

**Testing**:
- `Integration: assign clean room then check-in → room OCCUPIED, reservation CHECKED_IN`
- `Integration: assign dirty room without permission → 403`
- `Integration: check-in without assigned room → 409`
- `Integration: walk-in flow → reservation CHECKED_IN, inventory decremented`

#### 3.3 — Front desk: check-out

**What**: Settle and close out a stay.

**Design**:
- `POST /v1/reservations/{rid}/check-out` — requires folio balance ≤ 0 (or `folio.settle_with_balance` permission to transfer remaining to city ledger); sets `status=CHECKED_OUT`, `checked_out_at`, room `status=AVAILABLE`, `housekeeping_status=DIRTY`; auto-creates a `CHECKOUT` housekeeping task; releases the night that frees up.
- Returns the final folio snapshot.

**Testing**:
- `Integration: check-out with zero balance → CHECKED_OUT, room DIRTY, housekeeping task created`
- `Integration: check-out with outstanding balance, no permission → 409`
- `Integration: check-out transfers balance to CITY_LEDGER with permission → city-ledger folio created`

---

## Phase 4: Guest Profiles & CRM

### Purpose
Add guest identity, preferences, companies/travel agents, and GDPR consent — the data backbone for personalisation, repeat-guest recognition, and the later AI preference-modelling and comms features.

### Tasks

#### 4.1 — Guest profiles & preferences

**What**: Guest CRUD with stay history, preferences, and PII encryption.

**Design**:
- `guest` with encrypted `id_number` (AES-GCM via `security/encryption.py`, key from env/KMS); JSONB `profile_ext` for jurisdiction-specific registration data.
- `GET /v1/guests/{id}/stay-history` aggregates reservations (count, nights, revenue, last_stay).
- Duplicate detection on create: fuzzy match on `(last_name, email|phone)` → returns `409` with candidate matches unless `force=true`.

**Testing**:
- `Unit: id_number stored encrypted; round-trips on read`
- `Integration: create guest with existing email → 409 with candidate list`
- `Integration: stay-history aggregates nights and revenue correctly`

#### 4.2 — Companies, travel agents & GDPR consent

**What**: Corporate/agent accounts and consent lifecycle.

**Design**:
- `company` (credit limit, payment terms, optional ISO 17442 LEI), `travel_agent` (IATA, commission), linkable on reservations.
- Consent: `POST /v1/guests/{id}/consent` writes `consent_record` (type, granted/revoked, source, IP); `gdpr_consent` flag derived.
- GDPR erasure: `DELETE /v1/guests/{id}?mode=erase` anonymises PII (hash name/email, null `id_number`) while preserving financial records for audit/tax retention — implements GDPR right-to-erasure vs. legal-retention balance.

**Testing**:
- `Integration: grant then revoke marketing consent → two consent_records, current state revoked`
- `Integration: erase guest → PII anonymised, reservations/folios retained, audit_log entry written`

---

## Phase 5: Billing, Folios & Payments

### Purpose
Implement the financial heart: folios, charge posting, transaction codes, tax computation, PCI-safe payments via a pluggable gateway, invoices, and the nightly room-charge posting (night audit). Financial correctness, auditability, and PCI DSS v4.0 compliance are the priorities.

### Tasks

#### 5.1 — Folios, charges & transaction codes

**What**: Folio ledger operations with running balance integrity.

**Design**:
- `services/folio.py::post_charge(folio_id, transaction_code_id, amount_cents, qty, description)` — inserts `charge`, recomputes `folio.balance_cents` in the same transaction; voiding sets `is_void` and reverses balance.
- Transaction codes seeded per property (Room=1000, F&B=2000, Tax=9000…); `is_revenue` and `gl_account_code` drive reporting.
- Balance is always derived as `SUM(charges) - SUM(completed payments)`; stored `balance_cents` is a cached projection re-asserted on every mutation (a consistency assertion test guards drift).
- Endpoints: `POST /v1/folios/{id}/charges`, `POST /charges/{cid}/void`, `GET /v1/folios/{id}` (with line items), `POST /v1/folios/{id}/transfer` (move charges between folios).

**Testing**:
- `Unit: post two charges then a payment → balance = sum(charges) - payment`
- `Integration: void a charge → balance reverses; charge marked is_void`
- `Integration: transfer charge between folios → both balances correct, total conserved`
- `Property test: random sequence of charges/payments/voids → stored balance == derived balance`

#### 5.2 — Tax engine (JSONB rules)

**What**: Compute taxes from jurisdiction-specific JSONB rules.

**Design**:
- `tax_rule.definition` JSONB, e.g. `{"type":"percentage","rate":14.5,"applies_to":["ROOM"],"inclusive":false,"compound":false}`; supports flat, percentage, per-night, and compound rules.
- `services/tax.py::compute_taxes(property_id, charges, stay_dates) -> list[TaxLine]` posts tax charges against the Tax transaction code.

**Testing**:
- `Unit: 14.5% exclusive room tax on $200 → $29.00 tax line`
- `Unit: compound tax (tax-on-tax) ordering correct`
- `Unit: per-night flat city tax × 3 nights → 3× amount`

#### 5.3 — Payment gateway abstraction (PCI DSS v4.0)

**What**: Pluggable payment capture storing only tokens.

**Design**:
- Interface:
  ```python
  class PaymentGateway(Protocol):
      async def tokenize(self, card_ref: str) -> TokenResult: ...
      async def authorize(self, token: str, amount: Money) -> AuthResult: ...
      async def capture(self, auth_id: str, amount: Money) -> CaptureResult: ...
      async def refund(self, capture_id: str, amount: Money) -> RefundResult: ...
  ```
- `StripeGateway` implementation; PMS persists only `payment_token`, `authorization_code`, `gateway_reference` — **never a PAN** (PCI DSS v4.0). Card data captured client-side via Stripe Elements; backend receives a token.
- `payment` rows linked to folios; capture posts a payment, recompute balance; refunds post negative payments.
- Webhook endpoint `POST /v1/payments/webhook` verifies Stripe signature, updates payment status (async settlement).

**Testing**:
- `Integration (mocked gateway): authorize+capture → payment COMPLETED, folio balance reduced`
- `Integration (mocked): capture fails → payment FAILED, balance unchanged`
- `Unit: no PAN ever persisted (assert payment row has only token fields)`
- `Integration: webhook with invalid signature → 401, no state change`

#### 5.4 — Invoices & night audit

**What**: PDF invoice generation and the nightly batch that posts room charges and rolls the business date.

**Design**:
- `POST /v1/folios/{id}/invoice` generates an `invoice` row + PDF (via `weasyprint`), stores `pdf_url`.
- Night audit Celery task `run_night_audit(property_id, business_date)`: for every `CHECKED_IN` reservation, post that night's room charge + taxes; flag unpaid departures as `NO_SHOW` where applicable; snapshot daily KPIs (Phase 7); idempotent per `(property_id, business_date)`.

**Testing**:
- `Integration: night audit posts room charge for each in-house reservation`
- `Integration: night audit run twice for same date → charges posted once (idempotent)`
- `Integration: invoice generation → invoice row + non-empty PDF`

---

## Phase 6: AuthN/AuthZ, RBAC & Public REST API

### Purpose
Secure the system and turn the internal API into a public, OpenAPI-described, OAuth-protected surface — the API-first differentiator. Implements staff login, OAuth 2.0 for ISV integrations, fine-grained RBAC, and rate limiting, aligned with OWASP API Security Top 10.

### Tasks

#### 6.1 — Authentication (OAuth 2.0 + JWT)

**What**: Staff login and machine-to-machine auth.

**Design**:
- Staff: Authorization Code + PKCE (RFC 9700) issuing short-lived JWT access (15 min) + rotating refresh tokens (RFC 7519).
- Integrations: Client Credentials flow issuing scoped tokens.
- `oauth.py`: `/oauth/authorize`, `/oauth/token`, `/oauth/revoke`; `oauth_client` table for registered ISV apps (id, secret hash, redirect URIs, allowed scopes).
- JWT claims: `sub, tenant_id, scopes[], property_ids[], exp, iat, jti`; verified per request; `jti` denylist in Redis for revocation.

**Testing**:
- `Integration: auth-code+PKCE happy path → access+refresh tokens`
- `Unit: tampered JWT signature → 401`
- `Integration: client-credentials with disallowed scope → scope omitted from token`
- `Integration: revoked jti → subsequent request 401`

#### 6.2 — RBAC & tenant/property scoping

**What**: Permission enforcement per endpoint.

**Design**:
- Permissions seeded (`reservation.create`, `folio.void`, `rate.update`, `room.assign_dirty`, …); roles aggregate permissions; `staff_role` scopes a role to a property (null = all).
- FastAPI dep `require(permission: str)`; checks JWT scopes ∩ role permissions and that the path's `property_id` ∈ token `property_ids`. Mitigates OWASP API1 (BOLA) and API5 (BFLA).

**Testing**:
- `Integration: housekeeping role calling folio.void → 403`
- `Integration: front-desk at property A accessing property B reservation → 403`
- `Unit: require() allows when scope present and property in scope`

#### 6.3 — OpenAPI spec, rate limiting & API hardening

**What**: Polished OpenAPI 3.1 doc, rate limiting, and security headers.

**Design**:
- FastAPI auto-OpenAPI customised: servers, security schemes (OAuth2 + Bearer), tags, examples; spec served at `/openapi.json`, docs at `/docs`.
- Redis token-bucket rate limiter middleware (per client_id/IP), default 600 req/min, `429` + `Retry-After` (OWASP API4).
- Response models use `response_model` everywhere to prevent excessive data exposure (OWASP API3); PII fields gated by scope.
- Typed TS client generated from `/openapi.json` for the frontend.

**Testing**:
- `Integration: exceed rate limit → 429 with Retry-After`
- `Contract: /openapi.json validates as OpenAPI 3.1`
- `Integration: list reservations response excludes guest id_number unless pii.read scope`

---

## Phase 7: Reporting & Analytics

### Purpose
Deliver the operational metrics hoteliers run their business on — occupancy %, ADR, RevPAR, revenue by segment — plus the daily KPI snapshots the AI engine and dashboard consume. Lays groundwork for prescriptive AI analytics.

### Tasks

#### 7.1 — KPI computation & daily snapshots

**What**: Core metric calculators and a daily snapshot table.

**Design**:
- `services/reporting.py`: `occupancy(property, date_range)`, `adr` (room revenue / rooms sold), `revpar` (room revenue / available room-nights), `revenue_by_segment(by=source|channel|rate_plan)`.
- `daily_kpi_snapshot` table (property_id, business_date, rooms_available, rooms_sold, room_revenue_cents, adr_cents, revpar_cents, occupancy_pct) populated by night audit (5.4).
- Endpoints: `GET /v1/properties/{id}/reports/kpis?start=&end=&granularity=day|week|month`, `GET .../reports/revenue?by=segment`.

**Testing**:
- `Unit: 80/100 rooms sold → occupancy 80%`
- `Unit: ADR = room_revenue / rooms_sold; RevPAR = room_revenue / available`
- `Integration: KPI endpoint aggregates snapshots over a date range`
- `Edge: zero rooms sold → ADR returns null, not divide-by-zero`

#### 7.2 — Multi-property consolidated dashboard data

**What**: Cross-property roll-ups for boutique groups.

**Design**:
- `GET /v1/reports/portfolio?property_ids=&start=&end=` returns per-property + consolidated KPIs; respects token `property_ids` scope.

**Testing**:
- `Integration: portfolio report sums room revenue across two properties`
- `Integration: property outside token scope excluded from portfolio response`

---

## Phase 8: AI Revenue Management — Dynamic Pricing & Forecasting

### Purpose
Deliver the project's flagship differentiator: an embedded AI engine that forecasts demand and recommends/auto-applies dynamic rates at room-type and day granularity — eliminating the third-party RMS that competitors require. This is the AI-native heart of the product.

### Tasks

#### 8.1 — Demand forecasting

**What**: Probabilistic occupancy/booking-pace forecasts per room-type per future date.

**Design**:
- `services/forecasting/`: train per-room-type models on historical `daily_kpi_snapshot` + reservation booking-pace (lead-time curves), using statsmodels (SARIMAX/ETS) with seasonality and day-of-week; fall back to naive seasonal model when data is thin (<90 days).
- Output `DemandForecast(stay_date, room_type_id, expected_occupancy, p10, p50, p90, pickup_remaining)` stored in `forecast` table; retrained nightly via Celery.

**Testing**:
- `Unit: synthetic seasonal series → forecast captures weekly seasonality (MAPE < threshold)`
- `Unit: <90 days history → falls back to naive model, no exception`
- `Integration: nightly retrain task writes forecasts for next 365 days`

#### 8.2 — Dynamic pricing engine

**What**: Convert forecasts + demand signals into rate recommendations.

**Design**:
- `services/pricing/engine.py::recommend(property_id, room_type_id, date) -> PriceRecommendation`:
  - Inputs: forecast occupancy, current pickup vs. pace, comp-set rates (ingested via channel/comp-set feed, JSONB `rate.ai_signals`), events/weather signals (pluggable signal providers), floor/ceiling guardrails per rate plan.
  - Algorithm: base rate × demand multiplier (from forecasted occupancy band) × pace adjustment, clamped to `[floor, ceiling]`. Returns `recommended_amount_cents`, `confidence`, and a human rationale generated by the LLM gateway (LiteLLM) summarising the drivers.
- `pricing_recommendation` table (room_type_id, stay_date, current_cents, recommended_cents, rationale, confidence, status: SUGGESTED|APPLIED|DISMISSED, applied_by).
- Modes: **advisory** (recommendations only) or **auto-apply** (writes to `rate` within guardrails, audit-logged). Per-property config in `property.settings`.
- Endpoints: `GET /v1/properties/{id}/pricing/recommendations?start=&end=`, `POST .../recommendations/{rid}/apply`, `POST .../recommendations/{rid}/dismiss`.

**Testing**:
- `Unit: high forecast occupancy → recommended > base, clamped at ceiling`
- `Unit: low occupancy → recommended < base, clamped at floor`
- `Unit: recommendation never exceeds rate-plan ceiling or floor`
- `Integration (mocked LLM): rationale text returned and stored`
- `Integration: auto-apply mode writes rate row + audit_log entry`

#### 8.3 — Prescriptive analytics ("why")

**What**: Natural-language explanation of KPI movements.

**Design**:
- `POST /v1/properties/{id}/insights` body `{question}` — assembles relevant KPI snapshots + forecasts into a structured context, queries the LLM gateway for a grounded, cited answer ("RevPAR down 12% — ADR held but occupancy fell on Tue, comp-set dropped rates 8%"). Answers must reference only retrieved data (RAG over the warehouse), not hallucinate.

**Testing**:
- `Integration (mocked LLM): insight prompt includes the correct KPI numbers in context`
- `Unit: insight context builder selects the queried date range`

---

## Phase 9: OTA Channel Distribution & Webhooks

### Purpose
Connect the PMS to the outside world: synchronise availability/rates/restrictions out to OTAs and pull reservations in (via channel-manager intermediaries, the standard approach per standards.md note 5), and provide an outbound webhook framework for the integration marketplace.

### Tasks

#### 9.1 — Channel adapter framework

**What**: Pluggable channel-manager adapters with a common interface.

**Design**:
- `integrations/channel/base.py`:
  ```python
  class ChannelAdapter(Protocol):
      async def push_ari(self, mapping: PropertyChannel, updates: list[ARIUpdate]) -> SyncResult: ...
      async def pull_reservations(self, mapping: PropertyChannel, since: datetime) -> list[InboundReservation]: ...
  ```
- ARI payloads modelled on OTA 2.0 JSON / HTNG-aligned structures. A `MockChannelAdapter` ships for tests/demos; a `SiteMinderAdapter` stub defines the real integration contract.
- Credentials stored as `credentials_vault_ref` (encrypted), never plaintext.

**Testing**:
- `Unit: ARIUpdate serialises to OTA-aligned JSON`
- `Integration (mock adapter): push_ari → channel_sync_log SUCCESS row written`

#### 9.2 — Sync orchestration

**What**: Scheduled and event-driven ARI push + reservation pull.

**Design**:
- Celery tasks: `push_availability(property_channel_id)` triggered on any availability/rate change (debounced); `pull_reservations(property_channel_id)` polled every N minutes.
- Inbound reservations mapped to internal `create_reservation` with `source=OTA`, `channel_id`, `channel_reservation_id`; idempotent on `(channel_id, channel_reservation_id)`.
- `channel_sync_log` records every sync (direction, type, status, records, errors) for the dashboard sync-health view.

**Testing**:
- `Integration (mock): availability change enqueues debounced push task`
- `Integration (mock): pull duplicate channel_reservation_id → no duplicate reservation`
- `Integration: failed push → channel_sync_log FAILED with error_message`

#### 9.3 — Outbound webhooks

**What**: Webhook subscriptions and signed delivery for the marketplace.

**Design**:
- `webhook_subscription` table (tenant_id, url, event_types[], secret, is_active).
- Events emitted across the system (`reservation.created/checked_in/checked_out/cancelled`, `payment.completed`, `room.status_changed`). Delivery via Celery with HMAC-SHA256 signature header, exponential-backoff retries, and a dead-letter after N attempts.
- Endpoints to manage subscriptions; `GET /v1/webhooks/deliveries` for delivery audit.

**Testing**:
- `Integration: reservation.created → delivery attempted to subscribed URL with valid HMAC`
- `Integration: endpoint 500 → retried with backoff; after max attempts → dead-lettered`
- `Unit: subscription filters by event_type`

---

## Phase 10: Direct Booking Engine & Staff Frontend

### Purpose
Provide the guest-facing direct-booking channel (embeddable widget + standalone page) and the staff-facing dashboard/PWA — turning the API into usable products for hoteliers and their guests.

### Tasks

#### 10.1 — Public booking API & embeddable widget

**What**: Unauthenticated, rate-limited booking endpoints and a tiny embeddable widget.

**Design**:
- Public endpoints (separate router, scoped to one property by API key): `GET /booking/{property}/availability`, `POST /booking/{property}/quote`, `POST /booking/{property}/reserve` (creates guest + `source=DIRECT` reservation + payment authorization).
- `widget/` Preact + Vite bundle (<50 KB gzipped) embeddable via `<script>` + `<div data-pms-property="...">`; calls the public API; standalone page reuses the Next.js app.

**Testing**:
- `Integration: public availability returns only bookable inventory`
- `E2E (Playwright): widget search → select → guest details → pay (test gateway) → confirmation`
- `Integration: public reserve respects overbooking guard`

#### 10.2 — Staff dashboard & housekeeping PWA

**What**: Next.js dashboard for front desk/reservations/rates/reporting + mobile housekeeping view.

**Design**:
- Dashboard pages: reservations calendar (drag-and-drop room assignment), front-desk arrivals/departures, rate calendar with AI recommendations inline, folio view, KPI dashboard, multi-property switcher.
- Housekeeping PWA: room-status board, "my tasks" list, one-tap status updates (`DIRTY→IN_PROGRESS→CLEAN→INSPECTED`), offline-tolerant via service worker.
- Auth via the OAuth flow (6.1); typed client from OpenAPI (6.3).

**Testing**:
- `E2E (Playwright): login → create reservation → check-in → check-out flow`
- `E2E: housekeeper updates room status → reflected in front-desk board`
- `E2E: rate calendar apply AI recommendation → rate updated`

---

## Phase 11: Housekeeping Optimisation & AI Guest Communication

### Purpose
Layer the remaining AI-native differentiators onto the working system: predictive housekeeping scheduling and agentic guest communication across the stay journey.

### Tasks

#### 11.1 — Predictive housekeeping scheduling

**What**: Auto-sequence and assign cleaning tasks from occupancy/checkout predictions.

**Design**:
- `services/housekeeping.py::generate_schedule(property_id, date)`: predicts checkout times (from historical checkout patterns + flagged late checkouts), groups by floor/section, assigns to available staff balancing workload, and orders tasks to minimise idle time. Writes/updates `housekeeping_task` rows with `priority` and `assigned_to`.
- Endpoints: `POST /v1/properties/{id}/housekeeping/auto-schedule`, plus existing task CRUD and mobile status updates.

**Testing**:
- `Unit: checkout rooms prioritised over stayover; balanced across staff`
- `Integration: auto-schedule assigns all dirty checkout rooms to active housekeepers`
- `Edge: more rooms than staff → tasks queued, none dropped`

#### 11.2 — Agentic guest communication

**What**: Automated, context-aware pre-arrival / in-stay / post-stay messaging.

**Design**:
- `services/comms/`: trigger rules (e.g. 48h pre-arrival, post-checkout) enqueue Celery jobs that build per-guest context (reservation, preferences, property info) and generate messages via the LLM gateway; templates constrain tone and inject upsell offers where eligible.
- `message_log` table records sent messages; inbound replies (email/SMS webhook) optionally routed to staff or an agent loop.
- Channels: email (SMTP/SES) at MVP; SMS pluggable.

**Testing**:
- `Integration (mocked LLM + SMTP): 48h-before trigger → pre-arrival message generated and logged`
- `Unit: upsell offer only included when room upgrade available`
- `Integration: opted-out guest (no marketing consent) → no promotional message sent`

---

## Phase 12: MCP Server, Compliance Tooling & Hardening

### Purpose
Complete the AI-native and enterprise story: expose PMS data/actions to AI agents via MCP, add automated compliance posture reporting, and harden for production self-hosting.

### Tasks

#### 12.1 — MCP server

**What**: An MCP server exposing read/act tools over the PMS.

**Design**:
- `mcp/server.py` (official Python MCP SDK) exposing resources (`reservations`, `availability`, `rates`, `kpis`, `guests`) and tools (`search_availability`, `create_reservation`, `apply_price_recommendation`, `get_kpis`, `assign_housekeeping`). Auth via scoped OAuth client-credentials token; every action is RBAC-checked and audit-logged exactly as the REST path.

**Testing**:
- `Integration: MCP search_availability tool returns same data as REST availability`
- `Integration: MCP create_reservation respects RBAC scope and writes audit_log`
- `Unit: tool schemas validate against MCP spec`

#### 12.2 — Compliance posture reporting

**What**: Automated PCI/GDPR posture checks and reports.

**Design**:
- `services/compliance.py`: scheduled checks (no PAN in DB, TLS enforced, MFA on privileged roles, retention-policy execution on stale guest PII, consent coverage) producing a `compliance_report` with pass/fail findings mapped to PCI DSS v4.0 and GDPR articles.
- Retention job anonymises guests past the configured retention window without active reservations (GDPR storage limitation).

**Testing**:
- `Integration: planted raw card-like string in a text field → PCI check flags finding`
- `Integration: guest past retention with no active reservation → anonymised by retention job`
- `Unit: report maps each finding to a control reference`

#### 12.3 — Production hardening & self-host packaging

**What**: Observability, backups, and a turnkey self-host bundle.

**Design**:
- OpenTelemetry tracing across API/worker/DB; Prometheus metrics (`pms_availability_query_seconds`, `pms_otasync_failures_total`, `pms_payment_capture_total`); structured JSON logs.
- `docker-compose.yml` production profile with healthchecks, automatic Alembic migration on boot, seed/admin-bootstrap command, and documented backup/restore for Postgres.
- k6 load test for the availability engine (target p95 < 100 ms at expected QPS).

**Testing**:
- `Integration: app boot runs pending migrations automatically`
- `Load (k6): availability endpoint p95 < 100 ms at target concurrency`
- `Integration: /metrics exposes Prometheus counters; traces emitted on a sample request`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Multi-Tenancy & Schema ─── required by everything
    │
Phase 2: Inventory, Rates & Availability Engine ─── requires Phase 1
    │
Phase 3: Reservations & Front Desk ─── requires Phase 2
    ├── Phase 4: Guest Profiles & CRM ─── requires Phase 3 (can parallel with 5)
    └── Phase 5: Billing, Folios & Payments ─── requires Phase 3 (can parallel with 4)
         │
Phase 6: AuthN/AuthZ, RBAC & Public REST API ─── requires Phases 3–5
    ├── Phase 7: Reporting & Analytics ─── requires Phase 5 (can parallel with 9)
    │     │
    │     └── Phase 8: AI Revenue Management ─── requires Phase 7
    └── Phase 9: OTA Channel Distribution & Webhooks ─── requires Phases 2, 3, 6 (can parallel with 7/8)
         │
Phase 10: Booking Engine & Staff Frontend ─── requires Phase 6 (+ 7,8 for full UI) (can parallel with 9)
    │
Phase 11: Housekeeping AI & Guest Comms ─── requires Phases 3, 8, 10
    │
Phase 12: MCP Server, Compliance & Hardening ─── requires all prior phases
```

**Parallelism opportunities**
- Phases **4** and **5** can be built concurrently once Phase 3 lands.
- Phase **7** (reporting) and Phase **9** (channels/webhooks) can proceed in parallel after Phase 6.
- Phase **10** (frontend/widget) can be developed in parallel with Phase 9 against the Phase 6 API.
- The frontend team can begin Phase 10 scaffolding against the OpenAPI contract as soon as Phase 6 publishes `/openapi.json`.

---

## Definition of Done (per phase)

A phase is complete only when every item below is satisfied:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`make test`).
3. Ruff lint + format pass with zero violations.
4. mypy (strict) passes on `backend/pms`; frontend `tsc --noEmit` passes for UI phases.
5. `docker compose up` builds and starts the affected services successfully.
6. The phase's feature works end-to-end (verified by an integration or E2E test, not just units).
7. New configuration options are added to `.env.example` and documented.
8. New/changed API endpoints appear correctly in `/openapi.json` and validate as OpenAPI 3.1.
9. Database changes ship as reviewed Alembic migrations that `upgrade head` and `downgrade` cleanly.
10. Money handling uses integer cents + ISO 4217 currency codes (no floats) in any new financial code.
11. New endpoints enforce RBAC + tenant/property scope and use explicit `response_model` (no excessive data exposure).
12. No PAN or other prohibited cardholder data is persisted (PCI DSS v4.0) in any new payment code path.
13. Mutations of financial/guest entities write an `audit_log` entry.
```

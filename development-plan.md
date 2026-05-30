# Mining Operations Management — Phased Development Plan

> Project: 370-mining-operations-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model suggestions. It adopts **Data Model Suggestion 1 (Entity-Centric Normalized Relational)** as the canonical schema — chosen because the platform must serve a production multi-site operation with strict regulatory audit boundaries (MSHA, ISO 45001, ISO 14001, GRI 14) and three distinct dashboard access patterns (fleet, maintenance, EHS/ESG), all of which are best served by indexed relational queries with clean per-domain tables. Time-series partitioning for `sensor_readings` and `audit_log` is taken from that suggestion directly. The JSONB-on-asset latest-telemetry idea from Suggestion 2 is borrowed as a denormalised `assets.latest_telemetry` cache to keep fleet dashboards fast.

---

## Core Requirements Synthesis

**What it does:** A single AI-native, open-source operations platform for mine operators that unifies asset/fleet tracking, maintenance (CMMS), safety & compliance (EHS), ore processing/production, and environmental/ESG reporting — domains that today require integrating multiple proprietary stacks (Maximo, Cat MineStar, Cority, RPMGlobal).

**Primary personas:** operations manager (multi-site KPI view), maintenance planner (work orders, parts, PM), maintenance technician (mobile, offline field work), operator (pre-shift inspections, telemetry source), safety officer (incidents, permits, contractors), environmental/ESG officer (emissions, tailings, GRI 14 reports).

**Key differentiators:** (1) one platform covering all five domains; (2) accessible to mid-tier/junior miners (no 6–18 month implementation); (3) AI-native — predictive failure scoring on mining-specific sensor profiles, AI-drafted SOPs, ESG narrative generation, natural-language operational querying; (4) open standards first — ISO 15143-3 (AEMP 2.0) mixed-fleet ingestion, OPC-UA/MQTT telemetry, GRI 14/ICMM ESG mapping.

**Deployment model:** Self-hostable SaaS. Containerised (Docker Compose for single-node; the same images deploy to Kubernetes for scale). Web dashboard + REST/WebSocket API + offline-capable mobile PWA + an edge ingestion agent for remote sites.

**Integration surface:** ISO 15143-3 OEM telematics APIs (Cat, Komatsu), OPC-UA/MQTT from SCADA/PLC, OAuth2/OIDC enterprise SSO (Azure AD, Okta), LLM provider (OpenAI/Anthropic/local), object storage for photos/attachments, outbound webhooks.

**Standards to implement:** ISO 15143-3 (AEMP 2.0) ingestion + normalisation; OPC-UA (IEC 62541) & MQTT v5.0 transports; OpenAPI 3.1 + AsyncAPI 3.0 for API description; JSON Schema 2020-12 validation; GeoJSON (RFC 7946) for spatial; OAuth 2.0 / OIDC + JWT (RFC 7519) auth; OWASP API Security Top 10 controls; GRI 14 & ICMM mapping; MSHA 7000-1 / 7000-2 form fields; GISTM tailings fields; ISO 55000 asset hierarchy.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **Python 3.12** | The heart of the product is AI/ML — predictive failure scoring, LLM orchestration for SOPs/ESG/NL-query. Python has the richest ML/LLM ecosystem (scikit-learn, river, openai/anthropic SDKs) and first-class OPC-UA (`asyncua`) and MQTT (`aiomqtt`) clients. |
| API framework | **FastAPI** | Async-native (needed for concurrent telemetry ingestion + LLM calls), generates **OpenAPI 3.1** automatically (a standards requirement), and uses Pydantic v2 for JSON Schema 2020-12 request/response validation. |
| ASGI server | **Uvicorn** behind **Gunicorn** workers | Standard production ASGI stack for FastAPI. |
| Database | **PostgreSQL 16** + **TimescaleDB** extension | Suggestion 1 is a relational model with FK integrity for regulatory auditability. TimescaleDB hypertables provide the time-series partitioning the model calls for on `sensor_readings` and `audit_log` without hand-rolled partition management. PostGIS-compatible geometry covers GeoJSON needs. |
| Spatial | **PostGIS** extension | Geofences, pit boundaries, haul-road networks, blast exclusion zones (GeoJSON / RFC 7946). |
| ORM / migrations | **SQLAlchemy 2.0 (async)** + **Alembic** | Async ORM matches FastAPI; Alembic gives versioned, reviewable migrations required for a regulated system. |
| Task queue | **Celery** + **Redis** broker | Long-running async work: LLM calls, AEMP polling, ESG report generation, predictive model scoring batches, webhook delivery. Redis doubles as cache and Celery result backend. |
| Real-time push | **WebSockets** (FastAPI) described by **AsyncAPI 3.0** | Live fleet positions, sensor alerts, dashboard updates — matches Modular Mining's REST+AsyncAPI pattern from standards.md. |
| Telemetry ingestion | **`asyncua`** (OPC-UA), **`aiomqtt`** (MQTT v5.0), **`httpx`** (AEMP REST polling) | Direct support for the three telemetry transports in standards.md. |
| ML — predictive maintenance | **scikit-learn** (batch models) + **river** (online/incremental) | Per-equipment-class failure scoring; `river` enables retraining-on-failure-event without full retrain. |
| LLM orchestration | **Anthropic + OpenAI SDKs** behind a provider abstraction; **Model Context Protocol (MCP)** server | SOP generation, ESG narratives, NL querying, incident root-cause. Provider abstraction avoids lock-in; MCP exposes operational data to AI assistants (per Suggestion 1 standards alignment). |
| Frontend | **Next.js 15 (React, TypeScript)** + **TanStack Query** + **MapLibre GL** | Role-based dashboards; MapLibre renders GeoJSON fleet maps without proprietary map SDK. Server components for fast dashboard loads. |
| Mobile | **PWA** (same Next.js app, offline service worker + IndexedDB) | features.md requires offline field use; a PWA avoids dual native codebases and works on ruggedised Android devices. Native shell deferred to backlog. |
| AuthN/AuthZ | **OAuth 2.0 / OIDC** via **Authlib**; JWT (RFC 7519) access tokens; RBAC by `users.role` | Enterprise SSO (Azure AD/Okta) is standard in mining; RBAC enforces object-level authorisation (OWASP API #1/#5). |
| Edge agent | **Python** standalone container (`store-and-forward`) | Remote sites have unreliable connectivity; the agent buffers telemetry locally (SQLite WAL) and syncs when the link returns. |
| Object storage | **S3-compatible** (MinIO self-host / AWS S3) | Inspection photos, attachments, generated ESG report PDFs. |
| Validation | **Pydantic v2** + **JSON Schema 2020-12** | Telemetry normalisation from heterogeneous OEM sources needs strict schema validation. |
| Testing | **pytest** + **pytest-asyncio** + **testcontainers** (Postgres/Redis) + **Playwright** (e2e) + **schemathesis** (OpenAPI fuzz) | Layered testing; schemathesis property-tests the API against its own OpenAPI spec. |
| Code quality | **ruff** (lint+format), **mypy** (type check), **bandit** (security), **pip-audit** | OWASP/secure-by-default toolchain. |
| Package manager | **uv** | Fast, reproducible Python dependency resolution and lockfiles. |
| Containerisation | **Docker** + **docker-compose.yml** | Self-host deployment model. |
| Reporting / PDF | **WeasyPrint** (HTML→PDF) | GRI 14 / MSHA report generation from templated HTML. |

### Project Structure

```
mining-ops/
├── pyproject.toml
├── uv.lock
├── Dockerfile                      # API image
├── Dockerfile.edge                 # edge agent image
├── docker-compose.yml              # postgres+timescale, redis, minio, api, worker, web
├── alembic.ini
├── .env.example
├── README.md
├── openapi/                        # exported OpenAPI 3.1 + AsyncAPI 3.0 specs (CI-generated)
│   ├── openapi.json
│   └── asyncapi.yaml
├── migrations/                     # Alembic versions
│   └── versions/
├── src/
│   └── mining_ops/
│       ├── main.py                 # FastAPI app factory, router registration
│       ├── config.py               # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py          # async engine, session dependency
│       │   ├── base.py             # DeclarativeBase
│       │   └── models/             # SQLAlchemy models (one file per domain)
│       │       ├── site.py
│       │       ├── asset.py
│       │       ├── user.py
│       │       ├── work_order.py
│       │       ├── inventory.py
│       │       ├── sensor.py
│       │       ├── incident.py
│       │       ├── inspection.py
│       │       ├── permit.py
│       │       ├── production.py
│       │       ├── environmental.py
│       │       ├── ai_suggestion.py
│       │       └── audit.py
│       ├── schemas/                # Pydantic request/response models per domain
│       ├── api/
│       │   ├── deps.py             # auth, db, RBAC dependencies
│       │   ├── routers/            # one router module per domain
│       │   └── ws.py               # WebSocket endpoints
│       ├── services/               # business logic (domain services)
│       │   ├── assets.py
│       │   ├── work_orders.py
│       │   ├── incidents.py
│       │   ├── permits.py
│       │   ├── production.py
│       │   ├── environmental.py
│       │   └── kpi.py
│       ├── auth/                   # OIDC, JWT, RBAC
│       ├── ingestion/
│       │   ├── normaliser.py       # OEM → canonical telemetry
│       │   ├── aemp.py             # ISO 15143-3 poller
│       │   ├── opcua_adapter.py
│       │   ├── mqtt_adapter.py
│       │   └── pipeline.py         # write path + alert evaluation
│       ├── ml/
│       │   ├── features.py         # feature extraction from sensor_readings
│       │   ├── failure_model.py    # train/score predictive failure
│       │   └── registry.py         # model versioning/loading
│       ├── ai/
│       │   ├── provider.py         # LLM provider abstraction
│       │   ├── sop.py              # SOP generation
│       │   ├── esg.py              # GRI 14 narrative generation
│       │   ├── nl_query.py         # natural-language → SQL/answer
│       │   └── mcp_server.py       # MCP tool server
│       ├── reporting/
│       │   ├── gri14.py            # GRI 14 / ICMM report builder
│       │   ├── msha.py             # 7000-1 / 7000-2 form builders
│       │   └── templates/          # HTML templates for WeasyPrint
│       ├── tasks/                  # Celery tasks
│       ├── audit/                  # audit-log writer (middleware + helper)
│       └── webhooks/               # outbound webhook delivery
├── edge/
│   └── agent/                      # standalone edge ingestion agent
├── web/                            # Next.js app (TypeScript)
│   ├── app/                        # role-based dashboards
│   ├── components/
│   └── lib/api/                    # generated TS client from openapi.json
└── tests/
    ├── unit/
    ├── integration/
    ├── e2e/
    └── fixtures/                   # sample AEMP payloads, OPC-UA traces, MSHA cases
```

The structure groups by concern (db / api / services / ingestion / ml / ai / reporting) so every phase adds modules without restructuring.

---

## Phase 1: Foundation — Project Skeleton, Config, Database Core

### Purpose
Stand up the runnable skeleton: dependency management, configuration, the Postgres/TimescaleDB/Redis/MinIO compose stack, the SQLAlchemy base, Alembic, and the core `sites` + `users` tables with health checks. After this phase the API boots, connects to a real database, and a developer can run migrations and the test suite. Everything else builds on this.

### Tasks

#### 1.1 — Repository, tooling, and dependency setup

**What:** Initialise the Python project with `uv`, ruff, mypy, bandit, pytest, and the Docker/compose stack.

**Design:**
- `pyproject.toml` declares dependencies (fastapi, uvicorn[standard], gunicorn, sqlalchemy[asyncio], asyncpg, alembic, pydantic, pydantic-settings, authlib, celery[redis], redis, httpx, asyncua, aiomqtt, scikit-learn, river, anthropic, openai, weasyprint, boto3, python-jose) and dev deps (pytest, pytest-asyncio, testcontainers, schemathesis, ruff, mypy, bandit, pip-audit, playwright).
- `config.py` — Pydantic `Settings`:
```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="MINEOPS_")
    database_url: str
    redis_url: str = "redis://redis:6379/0"
    s3_endpoint: str | None = None
    s3_bucket: str = "mineops"
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    access_token_ttl_seconds: int = 3600
    oidc_issuer: str | None = None
    oidc_client_id: str | None = None
    oidc_client_secret: str | None = None
    llm_provider: Literal["anthropic", "openai", "none"] = "none"
    llm_api_key: str | None = None
    log_level: str = "INFO"
    environment: Literal["dev", "test", "prod"] = "dev"
```
- `docker-compose.yml` services: `db` (timescale/timescaledb-ha:pg16 with PostGIS), `redis`, `minio`, `api`, `worker` (celery), `web`. Healthchecks on each.
- `Dockerfile` multi-stage (uv install → slim runtime).
- CI step exports OpenAPI spec to `openapi/openapi.json` and fails if it diverges from committed copy.

**Testing:**
- `Unit: Settings loads from env → all fields populated with defaults applied` (missing optional → None, missing required `database_url` → ValidationError naming the field).
- `Integration (testcontainers): app factory boots against a real Postgres container → /health returns 200 {"status":"ok","db":"ok"}`.
- `CI smoke: docker compose up → api container healthcheck passes within 60s`.

#### 1.2 — Database base, session management, Alembic

**What:** Async SQLAlchemy engine/session, declarative base with shared mixins, and Alembic configured for async + TimescaleDB/PostGIS extensions.

**Design:**
- `db/base.py`: `class Base(DeclarativeBase)`; `TimestampMixin` (`created_at`, `updated_at` server-default `now()`); `UUIDPkMixin` (`id` UUID default `gen_random_uuid()`).
- `db/session.py`: `engine = create_async_engine(settings.database_url)`; `async def get_session() -> AsyncIterator[AsyncSession]` FastAPI dependency.
- Alembic env configured for async; first migration runs `CREATE EXTENSION IF NOT EXISTS timescaledb; postgis; pgcrypto;`.

**Testing:**
- `Unit: TimestampMixin sets created_at/updated_at on insert/update (mocked session)`.
- `Integration: alembic upgrade head on fresh container → extensions present (SELECT * FROM pg_extension)`.
- `Integration: get_session dependency yields a working session, rolls back on exception`.

#### 1.3 — `sites` and `users` tables + RBAC role enum

**What:** Implement the `sites` and `users` models exactly per Data Model Suggestion 1, plus the role enumeration that drives RBAC.

**Design:** SQLAlchemy models mirroring the DDL (`sites`: site_code unique, site_type/commodity/timezone, boundary_geojson JSONB, msha_mine_id; `users`: site_id FK, email unique, role CHECK enum, certifications TEXT[], is_contractor, induction_completed_at). Role enum:
```python
class Role(str, Enum):
    site_manager = "site_manager"; operations_manager = "operations_manager"
    maintenance_planner = "maintenance_planner"; maintenance_supervisor = "maintenance_supervisor"
    technician = "technician"; operator = "operator"; safety_officer = "safety_officer"
    environmental_officer = "environmental_officer"; contractor = "contractor"
    admin = "admin"; viewer = "viewer"
```
Alembic migration creates both tables with the indexes from the model (`idx_users_site`).

**Testing:**
- `Unit: Site/User Pydantic schemas reject invalid site_type/role → ValidationError`.
- `Integration: insert site then user referencing it → success; insert user with bad site_id → FK violation`.
- `Integration: duplicate site_code / duplicate email → IntegrityError`.

---

## Phase 2: AuthN/AuthZ & Audit Backbone

### Purpose
Every API call must be authenticated, authorised by role, and auditable — a non-negotiable for a regulated mining platform (ISO 45001, MSHA). This phase delivers OIDC/JWT login, RBAC dependencies, and the partitioned `audit_log`, so all subsequent CRUD endpoints inherit security and audit automatically.

### Tasks

#### 2.1 — JWT issuance and OIDC login

**What:** Local password login (for dev/small sites) and OIDC code flow (enterprise SSO), both issuing platform JWTs.

**Design:**
- `auth/jwt.py`: `create_access_token(sub: UUID, role: Role, site_id: UUID) -> str` (RFC 7519 claims `sub`, `role`, `site_id`, `exp`, `iat`); `decode_token(token) -> TokenPayload`.
- `auth/oidc.py`: Authlib client; `/auth/oidc/login` → redirect; `/auth/oidc/callback` → map IdP `email` to a `users` row, mint platform JWT.
- `POST /auth/login {email,password}` → `{access_token, token_type:"bearer", expires_in}`.
- Passwords hashed with `argon2`.

**Testing:**
- `Unit: create→decode round-trips claims; expired token → ExpiredSignatureError; tampered signature → JWTError`.
- `Integration (mocked IdP): callback with valid code maps to user, returns JWT; unknown email → 403`.
- `Integration: login with wrong password → 401, no token`.

#### 2.2 — RBAC dependency and object-level authorisation

**What:** FastAPI dependencies enforcing role permissions and site-scoping (OWASP API #1 broken object-level authz, #5 broken function-level authz).

**Design:**
```python
def require_roles(*roles: Role) -> Callable:  # function-level authz
def current_user(token=Depends(bearer)) -> UserCtx
def enforce_site_scope(user: UserCtx, site_id: UUID)  # raises 403 if cross-site & not admin
```
A `PERMISSIONS` matrix maps (resource, action) → allowed roles. All routers depend on `current_user`; mutating endpoints add `require_roles(...)`.

**Testing:**
- `Unit: require_roles(admin) with technician token → 403; with admin → passes`.
- `Unit: enforce_site_scope rejects user accessing another site_id unless admin`.
- `Integration: GET /assets with viewer token → 200; POST /assets with viewer token → 403`.

#### 2.3 — Partitioned audit log + write middleware

**What:** `audit_log` as a TimescaleDB hypertable and an automatic writer for all mutations.

**Design:** Model per Suggestion 1 (`actor_type`, `action`, `entity_type`, `entity_id`, `changes_json`, `ip_address`); converted to a hypertable on `created_at`. `audit/writer.py: record(actor_type, action, entity_type, entity_id, changes, request)`. A service-layer helper diffs before/after JSON. AI, sensor, and AEMP-sync actors are first-class so non-user changes are traceable.

**Testing:**
- `Integration: create asset via API → one audit_log row with action='create', entity_type='asset', changes_json containing new values, user_id set`.
- `Integration: update asset → audit row with changes_json showing only changed fields`.
- `Integration: hypertable chunks created over a 90-day insert range`.

---

## Phase 3: Asset & Fleet Core (Heart of the Product, Part 1)

### Purpose
The asset register with the site → area → equipment → component hierarchy (ISO 55000) is the spine that maintenance, telemetry, inspections, and incidents all attach to. This phase ships full asset CRUD, the self-referencing hierarchy, status lifecycle, and the fleet-status read API — the first user-visible value.

### Tasks

#### 3.1 — `assets` model, hierarchy, and CRUD

**What:** Implement the `assets` table with self-referencing `parent_asset_id`, all enums, and CRUD endpoints.

**Design:** Model per Suggestion 1 (asset_type, category, status, criticality CHECK enums; runtime_hours; next_pm_hours/date; latitude/longitude; `aemp_equipment_id`; `specs_json`; unique `(site_id, asset_tag)`; indexes idx_assets_site/type/parent/pm/aemp). Add denormalised `latest_telemetry JSONB` (borrowed from Suggestion 2) updated by the ingestion pipeline (Phase 6) so fleet dashboards avoid a subquery per asset.
- Endpoints: `POST /sites/{id}/assets`, `GET /assets/{id}`, `PATCH /assets/{id}`, `GET /sites/{id}/assets?category=&status=&type=`, `GET /assets/{id}/children`, `DELETE` → soft (`status='decommissioned'`).
- Status transitions validated by a small state machine: `operational ↔ maintenance ↔ breakdown ↔ standby`, any → `decommissioned` (terminal), `in_transit` allowed from operational/standby.

**Testing:**
- `Unit: status transition validator allows operational→maintenance, rejects decommissioned→operational`.
- `Integration: create parent asset + child component → GET /children returns the component`.
- `Integration: duplicate (site_id, asset_tag) → 409`.
- `Integration: DELETE sets status decommissioned, asset still retrievable`.

#### 3.2 — Fleet-status read API + GeoJSON positions

**What:** The fleet dashboard endpoint and a GeoJSON FeatureCollection of live positions.

**Design:**
- `GET /sites/{id}/fleet` → list of `FleetAssetView{asset_tag, name, asset_type, status, criticality, runtime_hours, position:{lat,lon}, fuel_pct, open_wo_count, next_pm_date}` (open_wo_count joined from Phase 4; defaults to 0 until then).
- `GET /sites/{id}/fleet/geojson` → RFC 7946 FeatureCollection; each Feature `geometry: Point`, `properties` carry status/type for map styling.
- Pagination uses RFC 8288 `Link` headers.

**Testing:**
- `Integration: seed 3 assets with positions → /fleet/geojson returns valid FeatureCollection (schema-validated against RFC 7946)`.
- `Integration: fleet list excludes decommissioned assets`.
- `Unit: GeoJSON serialiser omits Feature geometry when lat/lon null`.

#### 3.3 — Multi-site consolidated KPI endpoint

**What:** Operations-manager view aggregating KPIs across sites the user can access.

**Design:** `GET /fleet/overview` → per-site `{site_code, asset_count, operational_pct, assets_in_breakdown, avg_utilisation}`. Admin sees all sites; others see their own site only (enforced by `enforce_site_scope`). Implemented in `services/kpi.py` with one grouped query.

**Testing:**
- `Integration: admin across 2 sites → 2 rows; site_manager → only their site`.
- `Unit: operational_pct = operational / (total − decommissioned), zero-asset site → 0 not div-by-zero`.

---

## Phase 4: Maintenance — Work Orders & Inventory (Heart, Part 2)

### Purpose
Work orders are the core CMMS workflow and a table-stakes feature across every competitor. This phase ships the full work-order lifecycle with tasks/parts/labour/cost, parts inventory with reorder alerts, and links work orders back to assets — completing the maintenance dashboard.

### Tasks

#### 4.1 — `work_orders` model and lifecycle

**What:** Work-order CRUD with the status state machine and embedded task list.

**Design:** Model per Suggestion 1 (wo_type, priority, status enums; trigger_source; `tasks_json` array of `{seq, description, estimated_hours, actual_hours, technician_id, status, parts:[{part_id, part_number, qty}]}`; `sop_json`; cost columns; unique `(site_id, wo_number)`; indexes idx_wo_asset/site/assigned/planned).
- Lifecycle: `draft → planned → scheduled → in_progress → completed`; `on_hold` reachable from scheduled/in_progress; `cancelled` from any non-completed. `actual_start`/`actual_end` auto-stamped on transitions; `downtime_hours` computed.
- `wo_number` generated `WO-{site_code}-{yyyymm}-{seq}`.
- Endpoints: `POST /sites/{id}/work-orders`, `PATCH /work-orders/{id}`, `POST /work-orders/{id}/transition {to}`, `GET /sites/{id}/work-orders?status=&priority=&asset_id=&assigned_to=`.

**Testing:**
- `Unit: transition completed→in_progress rejected; in_progress→completed stamps actual_end and computes downtime`.
- `Integration: create WO on asset → asset /fleet open_wo_count increments`.
- `Integration: list filtered by priority sorted critical→low`.

#### 4.2 — `inventory_items` + parts reservation & reorder

**What:** Parts catalogue with on-hand/reserved quantities, reorder-point alerts, and reservation when consumed by work orders.

**Design:** Model per Suggestion 1 (unique `(site_id, part_number)`; reorder partial index). Service ops: `reserve(part_id, qty)` (increments `quantity_reserved`, fails if `on_hand − reserved < qty`), `consume(part_id, qty)` (decrements on_hand+reserved, writes cost to WO). `GET /sites/{id}/inventory/reorder-alerts` returns items at/below reorder point. Reorder alerts emitted as events for the webhook/notification layer (Phase 9).

**Testing:**
- `Unit: reserve more than available → InsufficientStock error, no mutation`.
- `Integration: consume parts on WO → on_hand decremented, WO parts_cost_cents updated`.
- `Integration: drop on_hand below reorder_point → appears in reorder-alerts`.

#### 4.3 — Preventive maintenance scheduling

**What:** PM triggers by calendar date, runtime hours, or meter reading that auto-generate work orders.

**Design:** `pm_schedules` (new minor table: `asset_id`, `trigger_type ∈ {calendar,runtime_hours,meter}`, `interval_value`, `last_done_at`, `last_done_hours`, `task_template_json`). A Celery beat job evaluates due PMs nightly and creates `wo_type='preventive'` work orders with `trigger_source='schedule'` (or `'meter_reading'`). Runtime-hours triggers also fire on telemetry update (Phase 6).

**Testing:**
- `Unit: PM due when runtime_hours ≥ last_done_hours + interval`.
- `Integration: beat job creates exactly one WO per due PM, updates next_pm_date on asset`.
- `Integration: PM not yet due → no WO created`.

---

## Phase 5: Safety & Compliance — Incidents, Inspections, Permits, Contractors

### Purpose
Safety is the most regulated domain (ISO 45001, MSHA). This phase delivers incident/near-miss reporting with investigation workflow and MSHA 7000-1 fields, digital pre-shift inspection checklists, permit-to-work with validity enforcement, and contractor induction/competency tracking.

### Tasks

#### 5.1 — `incidents` with investigation workflow + MSHA fields

**What:** Incident CRUD, status workflow, and MSHA 7000-1 form capture.

**Design:** Model per Suggestion 1 (incident_type, severity status enums; persons_involved JSONB; root_cause; corrective_actions; investigation_json; `msha_form_7000_1` JSONB; `msha_reported_at`). Workflow: `reported → under_investigation → corrective_action → closed` (+ `reopened`). On creation of severity ∈ {serious,critical,fatal}, emit a high-priority alert (MSHA's 15-minute clock). `msha_form_7000_1` validated against a JSON Schema mirroring the 7000-1 fields.

**Testing:**
- `Unit: 7000-1 payload missing required field → ValidationError naming field`.
- `Integration: create fatal incident → alert event emitted with msha_due_at = occurred_at + 15min`.
- `Integration: transition to closed sets closed_at`.

#### 5.2 — `inspections` with dynamic checklists

**What:** Pre-shift / pre-operational inspection capture with JSONB checklist and auto-WO on failure.

**Design:** Model per Suggestion 1 (inspection_type enum; `checklist_json` array `{item, result, notes, photo_url}`; status pass/fail/conditional_pass/incomplete; `defects_found`; optional `work_order_id`). On any item `result='fail'`, optionally auto-create a `corrective` WO and link it. Checklist templates per asset_type stored in site `settings_json`.

**Testing:**
- `Unit: defects_found computed = count of result='fail' items`.
- `Integration: failed inspection with auto_wo=true → corrective WO created and linked`.
- `Integration: photo_url accepts presigned S3 URL`.

#### 5.3 — `permits` (permit-to-work) with validity enforcement

**What:** Permit issuance with hazards/controls and expiry enforcement.

**Design:** Model per Suggestion 1 (permit_type, status enums; hazards/controls TEXT[]; valid_from/until; issued_by/holder). `POST /permits/{id}/issue` (draft→issued→active), `/close`. A Celery beat job moves `active`/`issued` permits past `valid_until` to `expired` and alerts. API rejects activating a permit whose window has passed.

**Testing:**
- `Unit: issue rejected if valid_until < now`.
- `Integration: beat job expires overdue active permits`.
- `Integration: cannot transition expired → active`.

#### 5.4 — Contractor induction & competency tracking

**What:** Track contractor users' induction completion, certifications, and site-access eligibility.

**Design:** Uses existing `users` columns (`is_contractor`, `contractor_company`, `induction_completed_at`, `certifications[]`, `msha_training_date`). `GET /sites/{id}/contractors` lists with eligibility flag (`induction_completed_at IS NOT NULL AND msha_training_date within 12 months`). Permit issuance to an ineligible holder is blocked.

**Testing:**
- `Unit: eligibility false when induction null or MSHA training >12 months old`.
- `Integration: issuing permit to ineligible contractor → 422 with reason`.

---

## Phase 6: Telemetry Ingestion & Normalisation (ISO 15143-3, OPC-UA, MQTT)

### Purpose
Real-time sensor and fleet telemetry is the data foundation for predictive maintenance, fleet dashboards, and environmental capture. This phase builds the ingestion pipeline: AEMP polling, OPC-UA and MQTT adapters, a normalisation layer producing canonical `sensor_readings`, and the latest-telemetry cache that powers fleet views. It can be developed in parallel with Phase 5 once Phase 3 is done.

### Tasks

#### 6.1 — `sensor_readings` hypertable + canonical schema

**What:** The time-series readings table as a TimescaleDB hypertable and the canonical reading model.

**Design:** Model per Suggestion 1 (reading_type CHECK enum incl. process metrics; value_numeric/text/json; unit; source enum; is_anomalous; anomaly_score; recorded_at) converted to hypertable on `recorded_at` with a 7-day chunk interval and a 13-month retention policy. Canonical Pydantic `Reading{asset_id, reading_type, value, unit, source, recorded_at}`.

**Testing:**
- `Integration: insert 10k readings across 30 days → chunks created; time-range query uses chunk exclusion`.
- `Integration: retention policy drops chunks older than 13 months (simulated)`.

#### 6.2 — Normalisation layer (OEM → canonical)

**What:** Convert heterogeneous OEM/standard payloads into canonical readings, matched to assets by `aemp_equipment_id`.

**Design:** `normaliser.normalise(source, raw_payload) -> list[Reading]`. ISO 15143-3 mapper extracts `Equipment.EquipmentHeader.EquipmentID`, `CumulativeOperatingHours`, `FuelUsed`, `Location{Latitude,Longitude}`, `FaultCodes[]` → typed readings. Per-OEM quirks (field coverage/units, see standards.md "fragmentation" note) handled in mapper subclasses; unit conversion to canonical SI. Unknown equipment IDs logged and dropped (not inserted).

**Testing:**
- `Fixture: real-shape AEMP JSON → expected canonical readings (runtime_hours, fuel_consumption, gps_position)`.
- `Unit: gallons→litres conversion applied for US OEM payload`.
- `Unit: unmatched aemp_equipment_id → reading dropped, warning logged`.

#### 6.3 — AEMP poller, OPC-UA, and MQTT adapters

**What:** Three transport adapters feeding the normaliser.

**Design:**
- `aemp.py`: Celery beat polls each configured OEM endpoint (OAuth2/API key per standards.md) on an interval, pages results, dedupes by `(equipment_id, recorded_at)`.
- `opcua_adapter.py`: `asyncua` client subscribes to configured node IDs, maps node → reading_type.
- `mqtt_adapter.py`: `aiomqtt` subscriber on configured topics (MQTT v5.0), payloads JSON-Schema-validated before normalisation.
- All write through `pipeline.ingest(readings)` which (a) bulk-inserts, (b) updates `assets.latest_telemetry` and `runtime_hours`, (c) fires runtime-hour PM checks, (d) evaluates threshold/anomaly alerts.

**Testing:**
- `Integration (mocked httpx): AEMP poll → readings inserted, latest_telemetry updated`.
- `Integration (embedded MQTT broker): publish payload → reading persisted; invalid payload → rejected, not persisted`.
- `Integration (mock OPC-UA server): node value change → reading created`.

#### 6.4 — AsyncAPI 3.0 spec + live WebSocket push

**What:** Describe event streams in AsyncAPI 3.0 and push live updates to dashboards.

**Design:** `GET /ws/sites/{id}/fleet` WebSocket emits `asset.telemetry`, `asset.alert`, `wo.created` messages. `asyncapi.yaml` documents channels/payloads (committed, CI-validated). Pub/sub fan-out via Redis so multiple API workers broadcast.

**Testing:**
- `Integration: connect WS, ingest reading → client receives asset.telemetry message`.
- `Validation: asyncapi.yaml passes the AsyncAPI 3.0 validator in CI`.

---

## Phase 7: AI — Predictive Maintenance & SOP Generation

### Purpose
The AI-native advantage. This phase delivers the differentiator: ML failure scoring on mining-equipment-specific sensor profiles (not generic thresholds) with retraining on new failure events, AI-drafted maintenance SOPs, and the `ai_suggestions` store that records every AI output for auditability and an applied/dismissed learning loop. Requires Phase 6 (telemetry) and Phase 4 (work orders/failure events).

### Tasks

#### 7.1 — `ai_suggestions` store + LLM provider abstraction

**What:** Persist all AI outputs and abstract the LLM provider.

**Design:** Model per Suggestion 1 (suggestion_type enum; title/body; suggestion_data JSONB; confidence; is_applied/is_dismissed; llm_model; tokens_used). `ai/provider.py`: `class LLMProvider(Protocol): async def complete(system, user, *, max_tokens) -> Completion` with Anthropic/OpenAI/`none` (stub) implementations selected by config. All calls run in Celery tasks; token usage recorded.

**Testing:**
- `Unit: provider 'none' returns deterministic stub (enables offline tests)`.
- `Integration (mocked provider): SOP request → ai_suggestions row with tokens_used, model`.

#### 7.2 — Predictive failure model (per equipment class)

**What:** Feature extraction + failure-probability scoring per asset_type, retrained on failure events.

**Design:**
- `ml/features.py`: from `sensor_readings`, build rolling windows per asset (vibration mean/std, temperature trend, runtime since last service, fault-code counts).
- `ml/failure_model.py`: per-`asset_type` model (scikit-learn gradient boosting baseline; `river` incremental updater). Output `failure_probability ∈ [0,1]` + contributing features. A `breakdown` work order or `breakdown` status transition is logged as a labelled failure event for retraining.
- Scoring Celery task runs hourly; if `prob ≥ threshold`, writes an `ai_suggestions` row (`predictive_failure`) and optionally creates a `wo_type='predictive', trigger_source='ai_prediction'` work order. Writes `sensor_readings.anomaly_score`/`is_anomalous`.
- `ml/registry.py`: versioned models on disk/S3 keyed by `(asset_type, version)`; retraining triggered when N new failure events accumulate.

**Testing:**
- `Unit: feature extraction over a fixture window → expected aggregates`.
- `Unit: stub model with synthetic failing signal → prob above threshold`.
- `Integration: high-prob score → ai_suggestion + predictive WO created`.
- `Integration: logging a breakdown adds a labelled failure event to the training set`.

#### 7.3 — AI-generated maintenance SOPs

**What:** Draft SOPs from asset history + failure context.

**Design:** `ai/sop.py: generate_sop(work_order_id) -> Suggestion`. Prompt template:
```
System: You are a mining-equipment maintenance engineer. Produce a step-by-step SOP
with safety steps, isolation/LOTO requirements, required parts, and estimated labour hours.
Output strict JSON: {steps:[{seq,instruction,safety_note,parts:[part_number],est_hours}], hazards:[], references:[]}.
User: Asset {make/model/type}. Failure: {failure_code/description}. Recent faults: {...}.
Prior similar work orders: {summaries}.
```
Result validated against a JSON Schema, stored as `ai_suggestions(sop_generation)`; planner can apply it onto the WO's `sop_json` (sets `is_applied`).

**Testing:**
- `Unit: malformed LLM JSON → validation error, suggestion stored with low confidence flag, not auto-applied`.
- `Integration (mocked provider): generate → applying writes sop_json to WO and audit row`.

---

## Phase 8: Production & Environmental/ESG (Ore Processing + GRI 14)

### Purpose
Closes the two underserved domains identified in research: ore-processing production metrics and integrated environmental/ESG capture with automated GRI 14 / ICMM report generation — features largely absent from competitors. Requires Phases 1–3 and benefits from Phase 6 (SCADA-sourced process readings) and Phase 7 (ESG narrative). Parallelisable with Phase 9.

### Tasks

#### 8.1 — `production_records` + ore-processing dashboard

**What:** Per-shift production capture and the processing dashboard.

**Design:** Model per Suggestion 1 (per `(site_id, shift_date, shift)`; ore/waste tonnes, grade, crusher/mill tonnes, recovery_rate, concentrate, metal_produced, cycle times, fuel). Process metrics (`crusher_throughput`, `mill_power_draw`, `flotation_grade`, `recovery_rate`) also ingestible as live `sensor_readings` via SCADA (Phase 6). `GET /sites/{id}/production?from=&to=` returns shift series; `GET /sites/{id}/production/dashboard` returns current-shift grade/tonnage/recovery roll-up.

**Testing:**
- `Unit: duplicate (site, date, shift) → 409`.
- `Integration: shift series returns ordered records; recovery_rate roll-up averages weighted by mill_tonnes`.

#### 8.2 — `environmental_records` with GRI/ICMM mapping + exceedance alerts

**What:** Environmental measurement capture mapped to GRI 14 topics and ICMM principles, with regulatory-limit exceedance detection.

**Design:** Model per Suggestion 1 (record_type incl. tailings/water/dust/emissions; value_numeric; regulatory_limit; `is_exceedance`; `gri_topic`; `icmm_principle`; source). `is_exceedance` computed when `value_numeric > regulatory_limit`. Tailings records include GISTM governance fields in `notes`/structured JSON. Exceedances emit alerts. Sensor-sourced env readings auto-create records via a mapping config.

**Testing:**
- `Unit: value over limit → is_exceedance true and alert emitted`.
- `Integration: GET exceedances endpoint returns only is_exceedance rows in window`.
- `Unit: gri_topic required for record_types in {emissions,water_usage,tailings}`.

#### 8.3 — GRI 14 / ICMM report generation + MSHA forms

**What:** Generate ESG reports (GRI 14) and MSHA 7000-1/7000-2 forms as PDFs, with AI narrative.

**Design:**
- `reporting/gri14.py`: aggregate `environmental_records` by `gri_topic` over a period → structured data; `ai/esg.py` drafts the narrative per topic (`ai_suggestions(esg_narrative)`); `WeasyPrint` renders HTML template → PDF stored in S3; returns presigned URL.
- `reporting/msha.py`: `build_7000_1(incident_id)` and `build_7000_2(site_id, quarter)` from incidents and production/employment data → PDF. (No MSHA submission API exists per standards.md — generate exportable forms.)
- `POST /sites/{id}/reports/gri14 {period}` and `/reports/msha-7000-2 {quarter}` enqueue Celery generation; `GET /reports/{id}` polls status.

**Testing:**
- `Integration: GRI14 report over seeded env data → PDF generated, topics present, narrative from mocked provider`.
- `Unit: 7000-2 quarter aggregation sums employment hours + production tonnes`.
- `Integration: report generation is async; status pending→completed with URL`.

---

## Phase 9: Notifications, Webhooks & Edge Agent

### Purpose
Operationalise alerts (PM due, reorder, exceedance, incident, predictive failure) into delivery channels, expose outbound webhooks for ERP/BI integration, and ship the edge agent that lets remote sites with unreliable connectivity buffer and forward telemetry. Parallelisable with Phase 8.

### Tasks

#### 9.1 — Alert/event bus and notification delivery

**What:** Centralise the alerts emitted across phases into a deliverable notification stream.

**Design:** A lightweight `events` abstraction (`emit(event_type, payload, site_id)`); a `notifications` minor table (`user_id`, `type`, `payload`, `read_at`). Subscriptions route event types to recipients by role (e.g. predictive_failure → maintenance_planner; exceedance → environmental_officer). Delivery channels: in-app (WebSocket), email (SMTP). Push notification stub for PWA.

**Testing:**
- `Unit: exceedance event routes to environmental_officer recipients only`.
- `Integration: emit predictive_failure → planner gets in-app notification + WS push`.

#### 9.2 — Outbound webhooks (signed)

**What:** HMAC-signed webhook delivery with retry for ERP/BI consumers.

**Design:** `webhook_endpoints` (url, secret, event_types[], active). `webhooks/deliver.py` Celery task posts JSON with `X-Signature: sha256=...`, exponential-backoff retries (max 5), dead-letters after exhaustion. Endpoint management API admin-only.

**Testing:**
- `Unit: signature computed correctly and verifiable`.
- `Integration (mock receiver): event → delivery with valid signature; receiver 500 → retried`.

#### 9.3 — Edge ingestion agent (store-and-forward)

**What:** Standalone agent for remote sites buffering telemetry offline and syncing on reconnect.

**Design:** `edge/agent/`: collects OPC-UA/MQTT locally, writes to a local SQLite WAL buffer, and forwards to the central API `POST /ingest/batch` when connectivity is up. Idempotent ingestion keyed by reading `(asset_id, reading_type, recorded_at)`. Configurable buffer cap with oldest-first eviction. Mutual-TLS to the central API (NIST CSF / IEC 62443 boundary hardening).

**Testing:**
- `Integration: agent buffers while API unreachable, flushes on recovery, no duplicate readings`.
- `Unit: buffer eviction drops oldest when cap exceeded`.

---

## Phase 10: Natural-Language Querying, MCP & Mobile PWA

### Purpose
Deliver the remaining AI differentiator (NL querying for non-technical managers) plus the MCP server (AI-assistant integration from the data model's standards alignment) and the offline-capable mobile PWA for field technicians — the last must-have feature. Requires the read APIs from Phases 3–8.

### Tasks

#### 10.1 — Natural-language operational querying

**What:** Ask questions in plain English; get answers grounded in operational data.

**Design:** `ai/nl_query.py: answer(site_id, question) -> Answer`. Uses a constrained tool/function-calling approach: the LLM selects from a whitelist of safe, parameterised query tools (fleet status, open WOs, incident counts, exceedances, production) — never free-form SQL (OWASP injection prevention). Results summarised in natural language; stored as `ai_suggestions(nl_query_response)`. `POST /sites/{id}/ask {question}`.

**Testing:**
- `Unit: question maps to the correct query tool with extracted params (mocked provider)`.
- `Integration: "how many open critical work orders?" → numeric answer matching a direct DB query`.
- `Security: attempt to elicit cross-site data → blocked by site scope`.

#### 10.2 — MCP server

**What:** Expose operational data as MCP tools for external AI assistants.

**Design:** `ai/mcp_server.py` exposes read tools (get_fleet_status, list_open_work_orders, get_incidents, get_environmental_exceedances) and guarded write tools (create_work_order, acknowledge_alert). Auth via platform JWT; every call site-scoped and audited.

**Testing:**
- `Integration: MCP get_fleet_status returns same data as REST /fleet`.
- `Integration: MCP create_work_order respects RBAC + writes audit row`.

#### 10.3 — Mobile PWA with offline support

**What:** Field-technician PWA: work orders, inspections, incident reporting — offline-capable.

**Design:** Next.js PWA; service worker caches app shell + assigned WOs/inspection templates in IndexedDB; mutations queued in an outbox and synced when online (idempotency keys). Barcode/QR asset scan via device camera. Photo capture uploads to S3 (presigned). Optimised for ruggedised Android.

**Testing:**
- `E2E (Playwright, offline mode): complete a WO offline → queued → syncs on reconnect`.
- `E2E: submit inspection with photo offline → photo uploaded after reconnect`.
- `Unit: outbox dedupes by idempotency key`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (skeleton, db, sites/users)        ─── required by everything
    │
Phase 2: Auth & Audit backbone                         ─── requires Phase 1
    │
Phase 3: Asset & Fleet core                            ─── requires Phase 2
    │
    ├── Phase 4: Maintenance (work orders, inventory)  ─── requires Phase 3
    │       │
    │       └── Phase 7: AI predictive maint + SOPs    ─── requires Phase 4 + Phase 6
    │
    ├── Phase 5: Safety & Compliance                   ─── requires Phase 3 (∥ Phase 6)
    │
    └── Phase 6: Telemetry ingestion                   ─── requires Phase 3 (∥ Phase 5)
            │
            ├── Phase 8: Production & Environmental/ESG ─── requires 3 (uses 6,7); ∥ Phase 9
            │
            └── Phase 9: Notifications/Webhooks/Edge    ─── requires 6 (uses alerts from 4,5,8); ∥ Phase 8
                    │
Phase 10: NL Query + MCP + Mobile PWA                  ─── requires read APIs from 3–8
```

**Parallelism opportunities:**
- After Phase 3: **Phases 4, 5, and 6** can be built concurrently.
- After Phase 6: **Phases 8 and 9** can be built concurrently.
- Phase 7 needs both Phase 4 and Phase 6.
- The **web/PWA frontend** (Phase 10.3) can be developed incrementally alongside each backend phase once its read APIs land.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass; coverage for new modules ≥ 80%.
3. `ruff check` and `ruff format --check` pass with no errors.
4. `mypy src/mining_ops` passes with no errors.
5. `bandit` and `pip-audit` report no high-severity findings.
6. `docker compose up` brings the stack to healthy; `alembic upgrade head` applies cleanly on a fresh database.
7. The phase's feature works end-to-end (verified by at least one integration or e2e test exercising the real path).
8. New config options are documented in `.env.example` and the README.
9. New REST endpoints appear in the regenerated `openapi/openapi.json`, and new event channels in `openapi/asyncapi.yaml`; both validate against their specs in CI.
10. New/changed tables have an Alembic migration with a tested downgrade.
11. All mutating endpoints write an `audit_log` row and enforce RBAC + site scope (OWASP API #1/#5).
12. Schemathesis property tests pass against the updated OpenAPI spec.

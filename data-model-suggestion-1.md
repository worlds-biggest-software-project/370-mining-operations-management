# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Mining Operations Management · Created: 2026-05-26

## Philosophy

Every concept in mining operations — sites, assets, users, work orders, inventory, sensor readings, safety incidents, inspections, permits, production records, and environmental data — gets its own table with strict foreign-key relationships. This mirrors the hierarchical structure of mine operations: a site contains areas, areas contain equipment, equipment has components, and each entity generates telemetry, maintenance events, and compliance records.

Mining operations management has three dominant access patterns: (1) "show me the fleet status dashboard with real-time location, utilisation, and health for all equipment across sites" (fleet view), (2) "show the maintenance planner view with open work orders, parts availability, and upcoming PMs" (maintenance view), and (3) "show the safety and environmental compliance dashboard with incidents, inspections, and emissions data" (EHS/ESG view). All three are well-served by indexed relational queries with appropriate joins.

The regulatory environment demands strict auditability: MSHA requires incident reporting within 15 minutes, ISO 45001 mandates OHS records, ISO 14001 requires environmental monitoring, and GRI 14 / ICMM frameworks demand structured ESG data. Separate tables for each compliance domain create clean boundaries for regulatory reporting and audit trails.

**Best for:** Teams building a production mining platform where multi-site fleet management, complex maintenance workflows, regulatory compliance reporting (MSHA, GRI 14), and high-volume sensor data ingestion are priorities.

**Trade-offs:**
- Pro: Clean asset hierarchy — site → area → asset → component — with FK integrity
- Pro: Sensor readings partitioned for time-series performance at industrial scale
- Pro: Regulatory reporting (MSHA, GRI 14) from dedicated tables with standard SQL
- Pro: Work orders with parts and labour traceable through FK chains
- Con: 14 tables to maintain and migrate
- Con: Fleet dashboard requires joining assets + sensor readings + work orders
- Con: Multi-site consolidated view requires cross-site joins
- Con: High sensor data volume requires careful partitioning and retention strategy

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 15143-3 (AEMP 2.0) | Sensor readings schema aligned to AEMP telematics fields |
| ISO 55000 | Asset lifecycle management structure (register, hierarchy, status) |
| ISO 45001 | Safety incident and inspection tables aligned to OHS requirements |
| ISO 14001 | Environmental records table aligned to EMS data capture |
| IEC 62443 | OT/IT security controls for sensor ingestion endpoints |
| OPC-UA (IEC 62541) | Sensor readings ingested via OPC-UA adapters |
| MQTT v5.0 | Real-time telemetry transport from edge devices |
| GRI 14 | Environmental records mapped to GRI mining sector topics |
| ICMM | Safety and environmental data aligned to ICMM Performance Expectations |
| MSHA | Incidents table supports 7000-1 form fields |
| GISTM | Environmental records include tailings governance data |
| GeoJSON (RFC 7946) | Asset location stored as GeoJSON-compatible coordinates |
| OpenAPI 3.1 | REST API definition |
| AsyncAPI 3.0 | Real-time telemetry stream description |
| OAuth 2.0 / OIDC | Authentication |
| MCP | AI assistant integration |

---

## Sites

```sql
CREATE TABLE sites (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                TEXT NOT NULL,
    site_code           TEXT UNIQUE NOT NULL,
    site_type           TEXT NOT NULL CHECK (site_type IN (
                            'open_pit','underground','processing_plant',
                            'mixed','exploration','tailings_facility'
                        )),
    country             TEXT NOT NULL,
    state_province      TEXT,
    timezone            TEXT NOT NULL,
    latitude            NUMERIC(9,6),
    longitude           NUMERIC(9,6),
    boundary_geojson    JSONB,
    commodity           TEXT,
    operator_name       TEXT NOT NULL,
    msha_mine_id        TEXT,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    settings_json       JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Assets

```sql
CREATE TABLE assets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL REFERENCES sites(id),
    parent_asset_id     UUID REFERENCES assets(id),
    asset_tag           TEXT NOT NULL,
    name                TEXT NOT NULL,
    asset_type          TEXT NOT NULL CHECK (asset_type IN (
                            'haul_truck','excavator','loader','dozer',
                            'drill','grader','water_truck','crusher',
                            'conveyor','mill','flotation_cell','pump',
                            'generator','compressor','light_vehicle',
                            'fixed_plant','component','other'
                        )),
    category            TEXT NOT NULL CHECK (category IN (
                            'mobile_equipment','fixed_plant','processing',
                            'support','infrastructure','component'
                        )),
    area                TEXT,
    manufacturer        TEXT,
    model               TEXT,
    serial_number       TEXT,
    year_manufactured   INTEGER,
    purchase_date       DATE,
    purchase_cost_cents BIGINT,
    status              TEXT NOT NULL CHECK (status IN (
                            'operational','maintenance','breakdown',
                            'standby','decommissioned','in_transit'
                        )) DEFAULT 'operational',
    criticality         TEXT NOT NULL CHECK (criticality IN (
                            'critical','high','medium','low'
                        )) DEFAULT 'medium',
    runtime_hours       NUMERIC(10,1) NOT NULL DEFAULT 0,
    next_pm_hours       NUMERIC(10,1),
    next_pm_date        DATE,
    latitude            NUMERIC(9,6),
    longitude           NUMERIC(9,6),
    aemp_equipment_id   TEXT,
    oem_telematics_id   TEXT,
    specs_json          JSONB NOT NULL DEFAULT '{}',
    -- {"payload_tonnes": 220, "engine_power_kw": 2610, "fuel_tank_litres": 3785}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (site_id, asset_tag)
);
CREATE INDEX idx_assets_site ON assets (site_id, category, status);
CREATE INDEX idx_assets_type ON assets (asset_type, status);
CREATE INDEX idx_assets_parent ON assets (parent_asset_id)
    WHERE parent_asset_id IS NOT NULL;
CREATE INDEX idx_assets_pm ON assets (next_pm_date)
    WHERE status = 'operational';
CREATE INDEX idx_assets_aemp ON assets (aemp_equipment_id)
    WHERE aemp_equipment_id IS NOT NULL;
```

---

## Users

```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL REFERENCES sites(id),
    email               TEXT UNIQUE NOT NULL,
    display_name        TEXT NOT NULL,
    employee_id         TEXT,
    role                TEXT NOT NULL CHECK (role IN (
                            'site_manager','operations_manager',
                            'maintenance_planner','maintenance_supervisor',
                            'technician','operator','safety_officer',
                            'environmental_officer','contractor',
                            'admin','viewer'
                        )),
    department          TEXT,
    shift_pattern       TEXT,
    certifications      TEXT[] NOT NULL DEFAULT '{}',
    msha_training_date  DATE,
    is_contractor       BOOLEAN NOT NULL DEFAULT FALSE,
    contractor_company  TEXT,
    induction_completed_at TIMESTAMPTZ,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_site ON users (site_id, role);
```

---

## Work Orders

```sql
CREATE TABLE work_orders (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL REFERENCES sites(id),
    asset_id            UUID NOT NULL REFERENCES assets(id),
    wo_number           TEXT NOT NULL,
    wo_type             TEXT NOT NULL CHECK (wo_type IN (
                            'preventive','corrective','condition_based',
                            'predictive','emergency','inspection','project'
                        )),
    priority            TEXT NOT NULL CHECK (priority IN (
                            'critical','high','medium','low'
                        )) DEFAULT 'medium',
    status              TEXT NOT NULL CHECK (status IN (
                            'draft','planned','scheduled','in_progress',
                            'on_hold','completed','cancelled'
                        )) DEFAULT 'draft',
    title               TEXT NOT NULL,
    description         TEXT,
    failure_code        TEXT,
    failure_description TEXT,
    requested_by_id     UUID REFERENCES users(id),
    assigned_to_id      UUID REFERENCES users(id),
    planned_start       TIMESTAMPTZ,
    planned_end         TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    downtime_hours      NUMERIC(8,2),
    labour_hours        NUMERIC(8,2),
    parts_cost_cents    BIGINT DEFAULT 0,
    labour_cost_cents   BIGINT DEFAULT 0,
    total_cost_cents    BIGINT DEFAULT 0,
    trigger_source      TEXT CHECK (trigger_source IN (
                            'schedule','meter_reading','sensor_alert',
                            'operator_report','inspection','ai_prediction',
                            'manual'
                        )),
    sensor_reading_id   UUID,
    tasks_json          JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "seq": 1, "description": "Replace hydraulic filter",
    --   "estimated_hours": 2.0, "actual_hours": 1.5,
    --   "technician_id": "uuid", "status": "completed",
    --   "parts": [{"part_id": "uuid", "part_number": "HF-4520", "qty": 2}]
    -- }]
    sop_json            JSONB,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (site_id, wo_number)
);
CREATE INDEX idx_wo_asset ON work_orders (asset_id, status);
CREATE INDEX idx_wo_site ON work_orders (site_id, status, priority);
CREATE INDEX idx_wo_assigned ON work_orders (assigned_to_id, status)
    WHERE status IN ('planned', 'scheduled', 'in_progress');
CREATE INDEX idx_wo_planned ON work_orders (planned_start)
    WHERE status IN ('planned', 'scheduled');
```

---

## Inventory Items

```sql
CREATE TABLE inventory_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL REFERENCES sites(id),
    part_number         TEXT NOT NULL,
    description         TEXT NOT NULL,
    category            TEXT,
    manufacturer        TEXT,
    unit_of_measure     TEXT NOT NULL DEFAULT 'each',
    quantity_on_hand    INTEGER NOT NULL DEFAULT 0,
    quantity_reserved   INTEGER NOT NULL DEFAULT 0,
    reorder_point       INTEGER NOT NULL DEFAULT 0,
    reorder_quantity    INTEGER NOT NULL DEFAULT 0,
    unit_cost_cents     BIGINT,
    warehouse_location  TEXT,
    compatible_assets   TEXT[] NOT NULL DEFAULT '{}',
    lead_time_days      INTEGER,
    is_critical_spare   BOOLEAN NOT NULL DEFAULT FALSE,
    last_received_at    TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (site_id, part_number)
);
CREATE INDEX idx_inventory_site ON inventory_items (site_id, category);
CREATE INDEX idx_inventory_reorder ON inventory_items (site_id)
    WHERE quantity_on_hand <= reorder_point;
CREATE INDEX idx_inventory_critical ON inventory_items (site_id)
    WHERE is_critical_spare = TRUE;
```

---

## Sensor Readings

```sql
CREATE TABLE sensor_readings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id            UUID NOT NULL REFERENCES assets(id),
    site_id             UUID NOT NULL,
    reading_type        TEXT NOT NULL CHECK (reading_type IN (
                            'gps_position','runtime_hours','fuel_level',
                            'fuel_consumption','engine_temperature',
                            'hydraulic_pressure','hydraulic_temperature',
                            'vibration','payload','speed',
                            'idle_time','fault_code','battery_voltage',
                            'coolant_temperature','oil_pressure',
                            'ambient_temperature','tyre_pressure',
                            'crusher_throughput','mill_power_draw',
                            'flotation_grade','recovery_rate',
                            'conveyor_speed','pump_flow_rate'
                        )),
    value_numeric       NUMERIC(12,4),
    value_text          TEXT,
    value_json          JSONB,
    unit                TEXT,
    source              TEXT NOT NULL CHECK (source IN (
                            'aemp_api','opc_ua','mqtt','scada',
                            'manual','edge_device','plc'
                        )),
    is_anomalous        BOOLEAN NOT NULL DEFAULT FALSE,
    anomaly_score       NUMERIC(4,3),
    recorded_at         TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_sensor_asset ON sensor_readings (asset_id, reading_type, recorded_at DESC);
CREATE INDEX idx_sensor_site ON sensor_readings (site_id, reading_type, recorded_at DESC);
CREATE INDEX idx_sensor_anomaly ON sensor_readings (asset_id, recorded_at)
    WHERE is_anomalous = TRUE;
```

---

## Incidents

```sql
CREATE TABLE incidents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL REFERENCES sites(id),
    incident_number     TEXT NOT NULL,
    incident_type       TEXT NOT NULL CHECK (incident_type IN (
                            'injury','fatality','near_miss','hazard',
                            'property_damage','environmental_release',
                            'vehicle_collision','fire','electrical',
                            'ground_control','explosive'
                        )),
    severity            TEXT NOT NULL CHECK (severity IN (
                            'minor','moderate','serious','critical','fatal'
                        )),
    occurred_at         TIMESTAMPTZ NOT NULL,
    location_area       TEXT,
    latitude            NUMERIC(9,6),
    longitude           NUMERIC(9,6),
    description         TEXT NOT NULL,
    reported_by_id      UUID NOT NULL REFERENCES users(id),
    reported_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    asset_id            UUID REFERENCES assets(id),
    persons_involved    JSONB NOT NULL DEFAULT '[]',
    -- [{"user_id": "uuid", "name": "John Doe", "injury_type": "laceration", "body_part": "left_hand"}]
    root_cause          TEXT,
    corrective_actions  TEXT,
    investigation_json  JSONB,
    status              TEXT NOT NULL CHECK (status IN (
                            'reported','under_investigation','corrective_action',
                            'closed','reopened'
                        )) DEFAULT 'reported',
    msha_form_7000_1    JSONB,
    msha_reported_at    TIMESTAMPTZ,
    investigator_id     UUID REFERENCES users(id),
    closed_at           TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (site_id, incident_number)
);
CREATE INDEX idx_incidents_site ON incidents (site_id, incident_type, occurred_at DESC);
CREATE INDEX idx_incidents_status ON incidents (status, severity);
CREATE INDEX idx_incidents_asset ON incidents (asset_id) WHERE asset_id IS NOT NULL;
```

---

## Inspections

```sql
CREATE TABLE inspections (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL REFERENCES sites(id),
    asset_id            UUID REFERENCES assets(id),
    inspection_type     TEXT NOT NULL CHECK (inspection_type IN (
                            'pre_shift','pre_operational','safety_walk',
                            'workplace','environmental','equipment',
                            'electrical','fire_safety','road_condition'
                        )),
    inspector_id        UUID NOT NULL REFERENCES users(id),
    shift               TEXT,
    status              TEXT NOT NULL CHECK (status IN (
                            'pass','fail','conditional_pass','incomplete'
                        )),
    checklist_json      JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "item": "Tyre condition", "result": "pass", "notes": null
    -- }, {
    --   "item": "Hydraulic leaks", "result": "fail",
    --   "notes": "Minor leak on left boom cylinder", "photo_url": "..."
    -- }]
    defects_found       INTEGER NOT NULL DEFAULT 0,
    work_order_id       UUID REFERENCES work_orders(id),
    notes               TEXT,
    inspected_at        TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_inspections_site ON inspections (site_id, inspection_type, inspected_at DESC);
CREATE INDEX idx_inspections_asset ON inspections (asset_id, inspected_at DESC)
    WHERE asset_id IS NOT NULL;
CREATE INDEX idx_inspections_fail ON inspections (site_id, inspected_at)
    WHERE status = 'fail';
```

---

## Permits

```sql
CREATE TABLE permits (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL REFERENCES sites(id),
    permit_number       TEXT NOT NULL,
    permit_type         TEXT NOT NULL CHECK (permit_type IN (
                            'hot_work','confined_space','working_at_height',
                            'electrical_isolation','excavation',
                            'blasting','crane_lift','general'
                        )),
    status              TEXT NOT NULL CHECK (status IN (
                            'draft','issued','active','suspended',
                            'closed','expired'
                        )) DEFAULT 'draft',
    location_area       TEXT NOT NULL,
    description         TEXT NOT NULL,
    hazards             TEXT[] NOT NULL DEFAULT '{}',
    controls            TEXT[] NOT NULL DEFAULT '{}',
    issued_by_id        UUID NOT NULL REFERENCES users(id),
    holder_id           UUID NOT NULL REFERENCES users(id),
    valid_from          TIMESTAMPTZ NOT NULL,
    valid_until         TIMESTAMPTZ NOT NULL,
    closed_at           TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (site_id, permit_number)
);
CREATE INDEX idx_permits_site ON permits (site_id, status);
CREATE INDEX idx_permits_active ON permits (valid_until)
    WHERE status = 'active';
```

---

## Production Records

```sql
CREATE TABLE production_records (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL REFERENCES sites(id),
    shift_date          DATE NOT NULL,
    shift               TEXT NOT NULL CHECK (shift IN ('day','night','swing')),
    ore_tonnes_mined    NUMERIC(12,2),
    waste_tonnes_mined  NUMERIC(12,2),
    ore_grade_pct       NUMERIC(6,4),
    crusher_tonnes      NUMERIC(12,2),
    mill_tonnes         NUMERIC(12,2),
    mill_availability_pct NUMERIC(5,2),
    recovery_rate_pct   NUMERIC(5,2),
    concentrate_tonnes  NUMERIC(10,2),
    concentrate_grade_pct NUMERIC(6,4),
    metal_produced_kg   NUMERIC(10,3),
    haul_truck_loads    INTEGER,
    avg_cycle_time_min  NUMERIC(6,2),
    fuel_consumed_litres NUMERIC(10,2),
    notes               TEXT,
    recorded_by_id      UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (site_id, shift_date, shift)
);
CREATE INDEX idx_production_site ON production_records (site_id, shift_date DESC);
```

---

## Environmental Records

```sql
CREATE TABLE environmental_records (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL REFERENCES sites(id),
    record_date         DATE NOT NULL,
    record_type         TEXT NOT NULL CHECK (record_type IN (
                            'tailings','water_usage','water_quality',
                            'dust','noise','vibration','emissions',
                            'waste','biodiversity','rehabilitation'
                        )),
    measurement_point   TEXT,
    value_numeric       NUMERIC(12,4),
    unit                TEXT NOT NULL,
    regulatory_limit    NUMERIC(12,4),
    is_exceedance       BOOLEAN NOT NULL DEFAULT FALSE,
    source              TEXT CHECK (source IN (
                            'sensor','manual','laboratory','scada','estimated'
                        )),
    gri_topic           TEXT,
    icmm_principle      TEXT,
    notes               TEXT,
    recorded_by_id      UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_env_site ON environmental_records (site_id, record_type, record_date DESC);
CREATE INDEX idx_env_exceedance ON environmental_records (site_id, record_date)
    WHERE is_exceedance = TRUE;
CREATE INDEX idx_env_gri ON environmental_records (gri_topic, record_date)
    WHERE gri_topic IS NOT NULL;
```

---

## AI Suggestions

```sql
CREATE TABLE ai_suggestions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL REFERENCES sites(id),
    asset_id            UUID REFERENCES assets(id),
    user_id             UUID REFERENCES users(id),
    suggestion_type     TEXT NOT NULL CHECK (suggestion_type IN (
                            'predictive_failure','sop_generation',
                            'dispatch_optimisation','esg_narrative',
                            'root_cause_analysis','anomaly_alert',
                            'nl_query_response','maintenance_plan',
                            'query_response'
                        )),
    title               TEXT NOT NULL,
    body                TEXT NOT NULL,
    suggestion_data     JSONB,
    confidence          NUMERIC(4,3),
    is_applied          BOOLEAN NOT NULL DEFAULT FALSE,
    is_dismissed        BOOLEAN NOT NULL DEFAULT FALSE,
    llm_model           TEXT,
    tokens_used         INTEGER,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_site ON ai_suggestions (site_id, created_at DESC);
CREATE INDEX idx_ai_asset ON ai_suggestions (asset_id, suggestion_type)
    WHERE asset_id IS NOT NULL;
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID REFERENCES sites(id),
    user_id             UUID REFERENCES users(id),
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user','system','ai','sensor','scada',
                            'aemp_sync','edge_device'
                        )),
    action              TEXT NOT NULL,
    entity_type         TEXT NOT NULL,
    entity_id           UUID NOT NULL,
    changes_json        JSONB,
    ip_address          INET,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_site ON audit_log (site_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log (user_id, created_at);
```

---

## Example Queries

### Fleet status dashboard — equipment with latest sensor data

```sql
SELECT a.asset_tag, a.name, a.asset_type, a.status, a.criticality,
       a.runtime_hours, a.latitude, a.longitude,
       a.next_pm_hours, a.next_pm_date,
       (SELECT value_numeric FROM sensor_readings sr
        WHERE sr.asset_id = a.id AND sr.reading_type = 'fuel_level'
        ORDER BY sr.recorded_at DESC LIMIT 1) AS fuel_pct,
       (SELECT COUNT(*) FROM work_orders wo
        WHERE wo.asset_id = a.id AND wo.status IN ('planned','in_progress')) AS open_wo_count
FROM assets a
WHERE a.site_id = 'site-uuid'
  AND a.category = 'mobile_equipment'
  AND a.status != 'decommissioned'
ORDER BY a.criticality, a.asset_tag;
```

### Maintenance planner view — open work orders by priority

```sql
SELECT wo.wo_number, wo.wo_type, wo.priority, wo.status,
       wo.title, a.asset_tag, a.name AS asset_name,
       u.display_name AS assigned_to,
       wo.planned_start, wo.planned_end,
       wo.parts_cost_cents + wo.labour_cost_cents AS total_cost_cents
FROM work_orders wo
JOIN assets a ON a.id = wo.asset_id
LEFT JOIN users u ON u.id = wo.assigned_to_id
WHERE wo.site_id = 'site-uuid'
  AND wo.status IN ('planned', 'scheduled', 'in_progress')
ORDER BY
    CASE wo.priority WHEN 'critical' THEN 1 WHEN 'high' THEN 2
                     WHEN 'medium' THEN 3 WHEN 'low' THEN 4 END,
    wo.planned_start;
```

### Safety KPIs — incident rates by type

```sql
SELECT incident_type,
       COUNT(*) AS total_incidents,
       COUNT(*) FILTER (WHERE severity IN ('serious','critical','fatal')) AS serious_incidents,
       COUNT(*) FILTER (WHERE status = 'closed') AS closed,
       COUNT(*) FILTER (WHERE status != 'closed') AS open
FROM incidents
WHERE site_id = 'site-uuid'
  AND occurred_at >= now() - INTERVAL '12 months'
GROUP BY incident_type
ORDER BY total_incidents DESC;
```

### Environmental compliance — exceedances

```sql
SELECT record_type, measurement_point, record_date,
       value_numeric, unit, regulatory_limit,
       gri_topic
FROM environmental_records
WHERE site_id = 'site-uuid'
  AND is_exceedance = TRUE
  AND record_date >= now() - INTERVAL '90 days'
ORDER BY record_date DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Sites | 1 | sites |
| Assets | 1 | assets (self-referencing hierarchy: equipment → components) |
| Users | 1 | users (operators, technicians, managers, contractors) |
| Maintenance | 2 | work_orders, inventory_items |
| Telemetry | 1 | sensor_readings (partitioned time-series) |
| Safety | 3 | incidents, inspections, permits |
| Production | 1 | production_records (per-shift ore processing) |
| Environmental | 1 | environmental_records (tailings, water, dust, emissions) |
| AI | 1 | ai_suggestions |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **13** | |

---

## Key Design Decisions

1. **`assets` with self-referencing hierarchy** — `parent_asset_id` enables the site → area → equipment → component hierarchy required by ISO 55000 asset management. A haul truck is an asset; its engine, transmission, and hydraulic system are child assets.

2. **`sensor_readings` partitioned by `recorded_at`** — industrial telemetry generates thousands of readings per minute across a fleet. Partitioning by time enables efficient time-range queries, data retention management, and archival of historical readings.

3. **`assets.aemp_equipment_id`** — ISO 15143-3 (AEMP 2.0) assigns equipment identifiers that OEM telematics APIs use. Storing this ID on the asset row enables direct matching when ingesting mixed-fleet telematics data from Cat, Komatsu, and other OEM APIs.

4. **`work_orders.trigger_source`** — tracking whether a work order was triggered by a schedule, meter reading, sensor alert, AI prediction, or operator report enables maintenance strategy analysis: "what percentage of our work orders are predictive vs. reactive?"

5. **`incidents.msha_form_7000_1`** — MSHA requires specific form fields for accident/injury reporting. Storing the MSHA form data as JSONB on the incident row enables regulatory form generation while keeping the relational columns focused on operational fields.

6. **`environmental_records` with GRI topic and ICMM principle** — mapping each environmental measurement to the relevant GRI 14 topic and ICMM principle enables automated ESG report data extraction without manual classification.

7. **`production_records` per shift** — ore processing metrics (tonnes mined, grade, recovery rate, concentrate produced) are captured per shift, not continuously. This granularity matches mine reporting practices and enables shift-over-shift comparison.

8. **`inspections.checklist_json`** — pre-shift inspection checklists vary by equipment type and site. JSONB accommodates different checklist structures without schema changes. The `defects_found` counter enables rapid inspection-pass-rate analytics.

9. **`permits` with valid time window** — permit-to-work issuance requires tracking validity periods with strict expiry enforcement. The `valid_until` index enables automated expiry alerting and prevents work under expired permits.

10. **13 tables** — mining operations span fleet management, maintenance, safety, production, and environmental compliance. Each regulatory domain gets its own table to ensure clean audit boundaries and regulatory reporting paths.

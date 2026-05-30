# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Mining Operations Management · Created: 2026-05-26

## Philosophy

Mining operations centre on two primary aggregates: the **asset** (equipment with its maintenance history, component hierarchy, and health status) and the **shift** (the operational period during which production, safety, and environmental data are captured). This model makes each the centre of a rich JSONB document: assets embed their component hierarchy, maintenance summary, telemetry overview, and location; shift operations embed incidents, inspections, permits, production metrics, and environmental readings. Sensor readings remain relational and partitioned because industrial telemetry is extremely high-volume.

Mining has two dominant access patterns: (1) "show me this asset's full context — components, maintenance history, current health, and location" and (2) "show me this shift's operational summary — production, safety events, and environmental status." Embedding all context on the asset and operations rows means both views are single-row reads.

For mid-tier mining operations with 50-500 equipment assets and 2-3 shifts per day, the JSONB approach drastically reduces schema complexity. Adding a new sensor type, a new inspection checklist item, or a new environmental metric requires no migration — just a new field in the JSONB structure.

**Best for:** Teams building an MVP for mid-tier mines where fast asset dashboard loading, shift-level operational summaries, minimal schema migrations, and rapid deployment are priorities.

**Trade-offs:**
- Pro: 6 tables — simple schema, fast to deploy
- Pro: Full asset context (components, maintenance, health) in one row read
- Pro: Full shift context (production, safety, environmental) in one row read
- Pro: New equipment types, sensor metrics, and compliance fields require no migration
- Con: Cross-asset analytics require JSONB extraction
- Con: No FK enforcement on component or work order references within JSONB
- Con: Assets with long maintenance histories can produce large JSONB
- Con: Concurrent updates (multiple technicians on same asset) need careful locking

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 15143-3 (AEMP 2.0) | Asset telemetry_json aligned to AEMP fields |
| ISO 55000 | Asset hierarchy and lifecycle in components_json |
| ISO 45001 | Safety incidents and inspections in operations shift data |
| ISO 14001 | Environmental readings in operations shift data |
| OPC-UA / MQTT | Sensor readings ingested via adapters |
| GRI 14 | Environmental metrics mapped to GRI topics |
| MSHA | Incident data supports MSHA reporting fields |
| GeoJSON (RFC 7946) | Asset location as coordinates |
| OpenAPI 3.1 | REST API |
| OAuth 2.0 / OIDC | Authentication |
| MCP | AI assistant integration |

---

## Users

```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
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
    site_json           JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "id": "uuid", "name": "Goldfield Mine",
    --   "site_code": "GFM-01", "site_type": "open_pit",
    --   "country": "AU", "timezone": "Australia/Perth",
    --   "msha_mine_id": null, "commodity": "gold"
    -- }
    certifications      TEXT[] NOT NULL DEFAULT '{}',
    contractor_json     JSONB,
    -- {
    --   "company": "MineTech Services",
    --   "induction_completed_at": "2026-05-01T08:00:00Z",
    --   "competencies": ["hydraulic_systems","electrical_HV"],
    --   "contract_expires": "2026-12-31"
    -- }
    shift_pattern       TEXT,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_role ON users (role);
CREATE INDEX idx_users_site ON users USING GIN (site_json);
```

---

## Assets

```sql
CREATE TABLE assets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL,
    asset_tag           TEXT NOT NULL,
    name                TEXT NOT NULL,
    asset_type          TEXT NOT NULL CHECK (asset_type IN (
                            'haul_truck','excavator','loader','dozer',
                            'drill','grader','water_truck','crusher',
                            'conveyor','mill','flotation_cell','pump',
                            'generator','compressor','light_vehicle',
                            'fixed_plant','other'
                        )),
    category            TEXT NOT NULL CHECK (category IN (
                            'mobile_equipment','fixed_plant','processing',
                            'support','infrastructure'
                        )),
    status              TEXT NOT NULL CHECK (status IN (
                            'operational','maintenance','breakdown',
                            'standby','decommissioned','in_transit'
                        )) DEFAULT 'operational',
    criticality         TEXT NOT NULL CHECK (criticality IN (
                            'critical','high','medium','low'
                        )) DEFAULT 'medium',
    components_json     JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Engine", "component_type": "engine",
    --   "manufacturer": "Cummins", "model": "QSK78",
    --   "serial_number": "CPL-78-0042",
    --   "runtime_hours": 12500, "next_pm_hours": 15000,
    --   "status": "operational",
    --   "sub_components": [
    --     {"id": "uuid", "name": "Turbocharger", "runtime_hours": 12500}
    --   ]
    -- }, {
    --   "id": "uuid", "name": "Transmission", "component_type": "transmission",
    --   "manufacturer": "Allison", "runtime_hours": 12500
    -- }]
    specs_json          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "manufacturer": "Caterpillar", "model": "797F",
    --   "serial_number": "CAT797F-0042",
    --   "year_manufactured": 2022,
    --   "payload_tonnes": 363, "engine_power_kw": 2983,
    --   "fuel_tank_litres": 3785,
    --   "purchase_date": "2022-06-15", "purchase_cost_cents": 500000000
    -- }
    location_json       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "latitude": -30.7505, "longitude": 121.4660,
    --   "area": "Main Pit - Level 4",
    --   "geofence": "active_mining_zone",
    --   "last_updated": "2026-05-26T10:30:00Z"
    -- }
    telemetry_json      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "runtime_hours": 12500.5,
    --   "fuel_level_pct": 68,
    --   "engine_temp_c": 92,
    --   "hydraulic_pressure_bar": 340,
    --   "speed_kmh": 0,
    --   "idle_pct_today": 18.5,
    --   "aemp_equipment_id": "CAT-GFM-042",
    --   "oem_telematics_id": "PL-12345",
    --   "last_aemp_sync": "2026-05-26T10:25:00Z",
    --   "health_score": 0.85,
    --   "fault_codes_active": ["SPN-3364"],
    --   "predicted_failure": {
    --     "component": "hydraulic_pump",
    --     "probability": 0.72,
    --     "estimated_rul_hours": 340,
    --     "model_version": "v2.3"
    --   }
    -- }
    maintenance_json    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "next_pm_hours": 15000, "next_pm_date": "2026-07-15",
    --   "pm_interval_hours": 500,
    --   "total_downtime_hours_ytd": 245,
    --   "availability_pct_mtd": 88.5,
    --   "open_work_orders": [{
    --     "id": "uuid", "wo_number": "WO-2026-0142",
    --     "type": "corrective", "priority": "high",
    --     "title": "Hydraulic pump replacement",
    --     "status": "scheduled", "planned_start": "2026-05-28",
    --     "assigned_to": "Mike Chen"
    --   }],
    --   "recent_completions": [{
    --     "wo_number": "WO-2026-0128", "type": "preventive",
    --     "title": "500hr service", "completed_at": "2026-05-20",
    --     "total_cost_cents": 850000
    --   }],
    --   "lifecycle_cost_cents_ytd": 4500000
    -- }
    inventory_json      JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "part_number": "HF-4520", "description": "Hydraulic filter",
    --   "qty_on_hand": 8, "reorder_point": 4, "unit_cost_cents": 12500,
    --   "is_critical_spare": true, "warehouse": "Main Store Bay 12"
    -- }]
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (site_id, asset_tag)
);
CREATE INDEX idx_assets_site ON assets (site_id, category, status);
CREATE INDEX idx_assets_type ON assets (asset_type, status);
CREATE INDEX idx_assets_telemetry ON assets USING GIN (telemetry_json);
CREATE INDEX idx_assets_maintenance ON assets USING GIN (maintenance_json);
```

---

## Operations

```sql
CREATE TABLE operations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL,
    shift_date          DATE NOT NULL,
    shift               TEXT NOT NULL CHECK (shift IN ('day','night','swing')),
    shift_supervisor_id UUID REFERENCES users(id),
    production_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "ore_tonnes_mined": 45000, "waste_tonnes_mined": 120000,
    --   "strip_ratio": 2.67, "ore_grade_pct": 1.85,
    --   "crusher_tonnes": 42000, "mill_tonnes": 38000,
    --   "mill_availability_pct": 94.5, "recovery_rate_pct": 91.2,
    --   "concentrate_tonnes": 850, "concentrate_grade_pct": 28.5,
    --   "metal_produced_kg": 242.25,
    --   "haul_truck_loads": 185, "avg_cycle_time_min": 28.5,
    --   "fuel_consumed_litres": 85000
    -- }
    safety_json         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "incidents": [{
    --     "id": "uuid", "number": "INC-2026-0042",
    --     "type": "near_miss", "severity": "minor",
    --     "occurred_at": "2026-05-26T14:30:00Z",
    --     "location": "Main Pit Ramp 3",
    --     "description": "Rock fall near haul road edge",
    --     "reported_by": "John Smith",
    --     "status": "under_investigation",
    --     "msha_reportable": false
    --   }],
    --   "inspections": [{
    --     "id": "uuid", "type": "pre_shift",
    --     "asset_tag": "HT-042", "inspector": "Mike Chen",
    --     "result": "pass", "defects": 0, "inspected_at": "2026-05-26T06:00:00Z"
    --   }, {
    --     "id": "uuid", "type": "pre_shift",
    --     "asset_tag": "HT-043", "inspector": "Jane Doe",
    --     "result": "fail", "defects": 1,
    --     "defect_notes": "Hydraulic leak on boom cylinder",
    --     "work_order_created": "WO-2026-0143"
    --   }],
    --   "permits_active": [{
    --     "id": "uuid", "number": "PTW-2026-018",
    --     "type": "hot_work", "location": "Workshop Bay 3",
    --     "holder": "Mike Chen",
    --     "valid_until": "2026-05-26T18:00:00Z"
    --   }],
    --   "shift_safety_score": 92.5,
    --   "total_incidents": 1, "total_inspections": 12,
    --   "inspection_pass_rate": 0.917
    -- }
    environmental_json  JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "readings": [{
    --     "type": "dust", "point": "Crusher PM10",
    --     "value": 45.2, "unit": "µg/m³",
    --     "limit": 50.0, "exceedance": false,
    --     "gri_topic": "GRI 305", "source": "sensor"
    --   }, {
    --     "type": "water_usage", "point": "Process Water",
    --     "value": 2500, "unit": "m³",
    --     "source": "scada"
    --   }, {
    --     "type": "tailings", "point": "TSF-1",
    --     "value": 12.5, "unit": "m (freeboard)",
    --     "limit": 3.0, "exceedance": false,
    --     "gri_topic": "GRI 306"
    --   }],
    --   "exceedance_count": 0,
    --   "water_recycled_pct": 78.5,
    --   "ghg_emissions_tco2e": 185.3
    -- }
    fleet_summary_json  JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total_mobile": 45, "operational": 38, "maintenance": 5, "breakdown": 2,
    --   "fleet_availability_pct": 84.4,
    --   "utilisation_pct": 72.1,
    --   "fuel_efficiency_litres_per_tonne": 1.89,
    --   "top_issues": [
    --     {"asset_tag": "HT-042", "issue": "Hydraulic pump degradation", "severity": "high"}
    --   ]
    -- }
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (site_id, shift_date, shift)
);
CREATE INDEX idx_ops_site ON operations (site_id, shift_date DESC);
CREATE INDEX idx_ops_safety ON operations USING GIN (safety_json);
CREATE INDEX idx_ops_environmental ON operations USING GIN (environmental_json);
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
                            'tyre_pressure','crusher_throughput',
                            'mill_power_draw','flotation_grade',
                            'recovery_rate','conveyor_speed',
                            'pump_flow_rate','dust_level','noise_level'
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
CREATE INDEX idx_sensor_anomaly ON sensor_readings (asset_id, recorded_at)
    WHERE is_anomalous = TRUE;
```

---

## AI Suggestions

```sql
CREATE TABLE ai_suggestions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID NOT NULL,
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
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id             UUID,
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
```

---

## Example Queries

### Asset dashboard — full context in one row read

```sql
SELECT asset_tag, name, asset_type, status, criticality,
       components_json, specs_json, location_json,
       telemetry_json, maintenance_json
FROM assets
WHERE id = 'asset-uuid';
```

### Fleet status from asset telemetry JSONB

```sql
SELECT asset_tag, name, asset_type, status,
       (telemetry_json->>'runtime_hours')::NUMERIC AS runtime_hours,
       (telemetry_json->>'fuel_level_pct')::NUMERIC AS fuel_pct,
       (telemetry_json->>'health_score')::NUMERIC AS health_score,
       telemetry_json->'fault_codes_active' AS active_faults,
       telemetry_json->'predicted_failure' AS failure_prediction,
       location_json->>'area' AS current_area
FROM assets
WHERE site_id = 'site-uuid'
  AND category = 'mobile_equipment'
  AND status != 'decommissioned'
ORDER BY (telemetry_json->>'health_score')::NUMERIC ASC NULLS LAST;
```

### Shift operations summary — single row read

```sql
SELECT shift_date, shift,
       production_json, safety_json, environmental_json,
       fleet_summary_json
FROM operations
WHERE site_id = 'site-uuid'
  AND shift_date = '2026-05-26'
  AND shift = 'day';
```

### Environmental exceedances from operations JSONB

```sql
SELECT o.shift_date, o.shift,
       r->>'type' AS record_type,
       r->>'point' AS measurement_point,
       (r->>'value')::NUMERIC AS value,
       r->>'unit' AS unit,
       (r->>'limit')::NUMERIC AS reg_limit,
       r->>'gri_topic' AS gri_topic
FROM operations o,
     jsonb_array_elements(o.environmental_json->'readings') AS r
WHERE o.site_id = 'site-uuid'
  AND (r->>'exceedance')::BOOLEAN = TRUE
  AND o.shift_date >= now() - INTERVAL '90 days'
ORDER BY o.shift_date DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users | 1 | users (embeds site context, contractor details) |
| Assets | 1 | assets (embeds components, specs, location, telemetry, maintenance, inventory) |
| Operations | 1 | operations (embeds production, safety incidents/inspections/permits, environmental, fleet summary per shift) |
| Telemetry | 1 | sensor_readings (relational, partitioned — high-volume industrial IoT) |
| AI | 1 | ai_suggestions |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **6** | |

---

## Key Design Decisions

1. **Asset as central aggregate** — a technician or planner opens one asset at a time and needs all context: component hierarchy, maintenance history, current telemetry, location, and parts inventory. Embedding everything on the asset row eliminates joins for the primary maintenance workflow.

2. **Operations as shift-level aggregate** — mining data is naturally organised by shift. Production metrics, safety events, environmental readings, and fleet summaries are all captured per shift. Embedding them on a single operations row means the shift handover report is a single-row read.

3. **`components_json` with nested sub-components** — a haul truck typically has 5-15 major components, each with 2-5 sub-components. The nested JSONB structure mirrors the physical asset hierarchy. New component types require no migration.

4. **`telemetry_json` on assets** — the latest telemetry summary (runtime, fuel, temperature, health score, fault codes, predicted failure) is embedded for the fleet dashboard view. This is a snapshot updated from the sensor_readings time-series, not the raw data.

5. **Sensor readings remain relational** — industrial IoT telemetry generates thousands of readings per minute across a fleet. Keeping sensor_readings in its own partitioned table preserves time-series query performance, anomaly detection indexing, and data retention management.

6. **`safety_json` on operations** — incidents, inspections, and permits for a shift (typically 0-2 incidents, 10-30 inspections, 2-5 active permits) are embedded together. This enables the shift safety score and the safety officer's shift review as a single read.

7. **`environmental_json` on operations** — environmental readings per shift (dust, water, tailings, emissions — typically 5-20 measurements) are embedded with GRI topic mapping. New environmental metrics require no migration.

8. **`maintenance_json` on assets** — open work orders, recent completions, lifecycle costs, and PM schedule are embedded for the maintenance planner's asset view. The work order lifecycle (draft → completed) is managed via JSONB updates.

9. **`inventory_json` on assets** — compatible parts with stock levels and reorder points are embedded on the asset row. For mid-tier operations with 500-2,000 part numbers, this keeps the parts lookup within the asset context.

10. **6 tables** — the asset and shift-operations aggregates capture the two primary operational views. Sensor readings stay relational for volume. This keeps the schema simple for mid-tier mine deployments while supporting rapid iteration on equipment types and compliance fields.

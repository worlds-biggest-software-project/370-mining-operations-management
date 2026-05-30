# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Mining Operations Management · Created: 2026-05-26

## Philosophy

Every state change in the mining platform — asset commissioned, sensor reading recorded, work order raised, inspection completed, incident reported, ore processed, emission measured — is recorded as an immutable event in a single append-only store. The event store is the sole source of truth; five materialised read models are projected from the event stream to serve the fleet status, maintenance dashboard, safety board, production metrics, and environmental compliance views.

Mining operations have strict regulatory audit requirements across multiple domains: MSHA mandates accident reporting within 15 minutes with full investigation records, ISO 45001 requires OHS management audit trails, ISO 14001 requires environmental monitoring records, and GRI 14 / ICMM frameworks demand verifiable ESG data lineage. With event sourcing, the immutable event log satisfies all audit requirements by design. Regulators can trace any measurement, any incident, any work order back through the event chain to its source.

The CQRS pattern separates the high-throughput write path (sensor telemetry at thousands of events per minute, SCADA feeds, OPC-UA subscriptions) from the read-optimised fleet and dashboard views. Edge computing nodes at remote mine sites can buffer events locally during connectivity outages and replay them to the central store when connectivity returns — naturally modeled as event append rather than row-level conflict resolution.

**Best for:** Teams building a compliance-first mining platform where MSHA/ISO regulatory audit trails, high-throughput industrial IoT ingestion, edge-to-cloud event synchronisation, and temporal operational replay are priorities.

**Trade-offs:**
- Pro: Regulatory audit compliance by design — MSHA, ISO 45001, ISO 14001 satisfied by immutable events
- Pro: Edge-to-cloud sync maps naturally to event buffering and replay
- Pro: Sensor telemetry and operational events share one storage and query model
- Pro: ESG data lineage verifiable through event chain for GRI 14 reporting
- Con: CQRS adds infrastructure complexity — event store + projections + read models
- Con: Extreme telemetry volume requires aggressive partitioning and snapshot strategy
- Con: Eventual consistency between sensor events and fleet status read model
- Con: Debugging operational issues requires replaying event streams

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Event envelope follows CloudEvents spec (ce_source, ce_type, ce_specversion, ce_time) |
| ISO 15143-3 (AEMP 2.0) | Sensor events carry AEMP-aligned telemetry fields |
| ISO 55000 | Asset lifecycle events (commissioned, maintained, decommissioned) |
| ISO 45001 | Safety events form the OHS audit trail |
| ISO 14001 | Environmental events form the EMS audit trail |
| IEC 62443 | OT/IT security context on event metadata |
| OPC-UA / MQTT | Source protocols recorded on sensor events |
| GRI 14 | Environmental events tagged with GRI topic |
| MSHA | Incident events carry MSHA form fields |
| GISTM | Tailings events aligned to GISTM governance |
| GeoJSON (RFC 7946) | Location data in event payloads |
| AsyncAPI 3.0 | Real-time event stream description |
| OAuth 2.0 / OIDC | Actor identity on every event |
| MCP | AI assistant integration via ai stream events |

---

## Event Store

```sql
CREATE TABLE event_store (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type         TEXT NOT NULL CHECK (stream_type IN (
                            'site','user','asset','maintenance',
                            'safety','production','environmental',
                            'ai','config'
                        )),
    stream_id           UUID NOT NULL,
    sequence_number     BIGINT NOT NULL,
    event_type          TEXT NOT NULL,
    event_data          JSONB NOT NULL,
    metadata            JSONB NOT NULL DEFAULT '{}',
    actor_id            UUID,
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user','system','ai','sensor','scada',
                            'aemp_sync','edge_device','plc'
                        )),
    actor_role          TEXT,
    site_id             UUID,
    asset_id            UUID,
    ce_source           TEXT NOT NULL DEFAULT '/mining-ops',
    ce_specversion      TEXT NOT NULL DEFAULT '1.0',
    ce_type             TEXT NOT NULL,
    ce_time             TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, sequence_number)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_stream ON event_store (stream_type, stream_id, sequence_number);
CREATE INDEX idx_events_type ON event_store (event_type, created_at);
CREATE INDEX idx_events_site ON event_store (site_id, created_at);
CREATE INDEX idx_events_asset ON event_store (asset_id, created_at)
    WHERE asset_id IS NOT NULL;
```

### Key Event Types by Stream

**asset stream:**
- `asset_commissioned` — new equipment added to fleet with specs and components
- `asset_status_changed` — operational → maintenance → breakdown → standby
- `asset_location_updated` — GPS position change (from AEMP or edge device)
- `component_installed` — new component added to asset hierarchy
- `component_replaced` — component swapped (captures old/new serial numbers)
- `runtime_hours_updated` — meter reading incremented
- `fault_code_received` — OEM diagnostic code received
- `health_score_calculated` — AI computed asset health from sensor history
- `failure_predicted` — ML model predicted component failure (component, probability, RUL hours)
- `asset_decommissioned` — equipment retired from service

**maintenance stream:**
- `work_order_created` — new WO with type, priority, trigger source
- `work_order_assigned` — technician assigned
- `work_order_started` — actual work began
- `work_order_task_completed` — individual task finished with parts and labour
- `work_order_completed` — all tasks done, costs finalised
- `work_order_cancelled` — WO cancelled with reason
- `pm_triggered` — preventive maintenance schedule fired (by hours or calendar)
- `parts_issued` — inventory items consumed for a work order
- `parts_received` — stock replenished
- `parts_reorder_triggered` — stock fell below reorder point
- `sop_generated` — AI drafted maintenance procedure

**safety stream:**
- `incident_reported` — injury, near-miss, hazard, or property damage
- `incident_investigation_started` — investigator assigned
- `incident_root_cause_identified` — root cause determined
- `incident_corrective_action_assigned` — corrective action created
- `incident_closed` — investigation completed
- `msha_report_submitted` — MSHA 7000-1 form data submitted
- `inspection_completed` — pre-shift or equipment inspection with pass/fail
- `inspection_defect_found` — defect identified during inspection
- `permit_issued` — permit-to-work activated
- `permit_closed` — permit returned/expired
- `contractor_inducted` — contractor completed site induction

**production stream:**
- `shift_production_recorded` — ore/waste tonnes, grade, recovery for a shift
- `crusher_throughput_recorded` — tonnes through crusher circuit
- `mill_metrics_recorded` — mill availability, throughput, power draw
- `concentrate_produced` — flotation output with grade
- `blast_completed` — blast event with pattern and fragmentation data
- `haul_cycle_completed` — individual truck cycle (load, haul, dump, return)

**environmental stream:**
- `environmental_reading_recorded` — dust, water, noise, emissions measurement
- `exceedance_detected` — measurement exceeded regulatory limit
- `exceedance_resolved` — corrective action brought measurement into compliance
- `tailings_level_recorded` — tailings storage facility freeboard measurement
- `water_usage_recorded` — process water consumption and recycling rate
- `ghg_emissions_calculated` — greenhouse gas emissions for period
- `gri_data_compiled` — GRI 14 data compiled for reporting period

**ai stream:**
- `anomaly_detected` — AI flagged abnormal sensor pattern
- `failure_prediction_generated` — ML model output
- `sop_drafted` — AI generated maintenance procedure
- `esg_narrative_generated` — AI compiled ESG report narrative
- `root_cause_suggested` — AI suggested incident root cause
- `dispatch_optimisation_suggested` — AI recommended fleet routing
- `suggestion_applied` / `suggestion_dismissed`

---

## Stream Snapshot

```sql
CREATE TABLE stream_snapshot (
    stream_type         TEXT NOT NULL,
    stream_id           UUID NOT NULL,
    sequence_number     BIGINT NOT NULL,
    snapshot_data       JSONB NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id)
);
```

---

## Projection Checkpoint

```sql
CREATE TABLE projection_checkpoint (
    projection_name     TEXT PRIMARY KEY,
    last_event_id       UUID NOT NULL,
    last_sequence       BIGINT NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Read Model: Fleet Status

```sql
CREATE TABLE rm_fleet_status (
    asset_id            UUID PRIMARY KEY,
    site_id             UUID NOT NULL,
    asset_tag           TEXT NOT NULL,
    name                TEXT NOT NULL,
    asset_type          TEXT NOT NULL,
    category            TEXT NOT NULL,
    status              TEXT NOT NULL,
    criticality         TEXT NOT NULL,
    location_json       JSONB NOT NULL DEFAULT '{}',
    -- {"latitude": -30.7505, "longitude": 121.4660, "area": "Main Pit - Level 4"}
    telemetry_json      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "runtime_hours": 12500.5, "fuel_level_pct": 68,
    --   "engine_temp_c": 92, "speed_kmh": 0,
    --   "health_score": 0.85,
    --   "fault_codes_active": ["SPN-3364"],
    --   "last_sync": "2026-05-26T10:25:00Z"
    -- }
    components_json     JSONB NOT NULL DEFAULT '[]',
    predicted_failure_json JSONB,
    -- {"component": "hydraulic_pump", "probability": 0.72, "rul_hours": 340}
    maintenance_json    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "next_pm_hours": 15000, "next_pm_date": "2026-07-15",
    --   "open_wo_count": 1, "availability_pct_mtd": 88.5,
    --   "downtime_hours_ytd": 245
    -- }
    specs_json          JSONB NOT NULL DEFAULT '{}',
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fleet_site ON rm_fleet_status (site_id, category, status);
CREATE INDEX idx_fleet_health ON rm_fleet_status ((telemetry_json->>'health_score'));
```

---

## Read Model: Maintenance Dashboard

```sql
CREATE TABLE rm_maintenance_dashboard (
    wo_id               UUID PRIMARY KEY,
    site_id             UUID NOT NULL,
    asset_id            UUID NOT NULL,
    asset_tag           TEXT NOT NULL,
    asset_name          TEXT NOT NULL,
    wo_number           TEXT NOT NULL,
    wo_type             TEXT NOT NULL,
    priority            TEXT NOT NULL,
    status              TEXT NOT NULL,
    title               TEXT NOT NULL,
    assigned_to         TEXT,
    planned_start       TIMESTAMPTZ,
    planned_end         TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    trigger_source      TEXT,
    downtime_hours      NUMERIC(8,2),
    total_cost_cents    BIGINT DEFAULT 0,
    tasks_json          JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "seq": 1, "description": "Replace hydraulic filter",
    --   "status": "completed", "hours": 1.5,
    --   "parts": [{"number": "HF-4520", "qty": 2}]
    -- }]
    parts_json          JSONB NOT NULL DEFAULT '[]',
    timeline_json       JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "event": "work_order_created", "at": "2026-05-25T08:00:00Z", "by": "System"},
    --  {"event": "work_order_assigned", "at": "2026-05-25T09:00:00Z", "to": "Mike Chen"},
    --  {"event": "work_order_started", "at": "2026-05-26T06:30:00Z"}
    -- ]
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_maint_site ON rm_maintenance_dashboard (site_id, status, priority);
CREATE INDEX idx_maint_asset ON rm_maintenance_dashboard (asset_id, status);
CREATE INDEX idx_maint_planned ON rm_maintenance_dashboard (planned_start)
    WHERE status IN ('planned', 'scheduled');
```

---

## Read Model: Safety Board

```sql
CREATE TABLE rm_safety_board (
    site_id             UUID NOT NULL,
    period_type         TEXT NOT NULL CHECK (period_type IN (
                            'daily','weekly','monthly','yearly'
                        )),
    period_start        DATE NOT NULL,
    incident_json       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total": 3, "by_type": {"near_miss": 2, "property_damage": 1},
    --   "by_severity": {"minor": 2, "moderate": 1},
    --   "open_investigations": 1,
    --   "trifr": 2.5, "ltifr": 0.0,
    --   "days_since_lti": 142,
    --   "recent": [{
    --     "id": "uuid", "number": "INC-2026-0042",
    --     "type": "near_miss", "severity": "minor",
    --     "description": "Rock fall near haul road",
    --     "status": "under_investigation",
    --     "occurred_at": "2026-05-26T14:30:00Z"
    --   }]
    -- }
    inspection_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total": 350, "pass": 320, "fail": 18, "conditional": 12,
    --   "pass_rate": 0.914,
    --   "by_type": {"pre_shift": 280, "safety_walk": 45, "equipment": 25},
    --   "top_defects": [
    --     {"item": "Hydraulic leaks", "count": 8},
    --     {"item": "Tyre damage", "count": 5}
    --   ]
    -- }
    permit_json         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "active": 5, "issued_period": 12, "closed_period": 10,
    --   "by_type": {"hot_work": 4, "confined_space": 3, "electrical": 5}
    -- }
    contractor_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total_onsite": 35, "inducted_period": 8,
    --   "certifications_expiring_30d": 3
    -- }
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (site_id, period_type, period_start)
);
CREATE INDEX idx_safety_period ON rm_safety_board (period_type, period_start DESC);
```

---

## Read Model: Production Metrics

```sql
CREATE TABLE rm_production_metrics (
    site_id             UUID NOT NULL,
    period_type         TEXT NOT NULL CHECK (period_type IN (
                            'shift','daily','weekly','monthly'
                        )),
    period_start        TIMESTAMPTZ NOT NULL,
    shift               TEXT,
    mining_json         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "ore_tonnes": 45000, "waste_tonnes": 120000,
    --   "strip_ratio": 2.67, "ore_grade_pct": 1.85,
    --   "haul_loads": 185, "avg_cycle_min": 28.5,
    --   "fuel_litres": 85000
    -- }
    processing_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "crusher_tonnes": 42000, "mill_tonnes": 38000,
    --   "mill_availability_pct": 94.5,
    --   "recovery_pct": 91.2,
    --   "concentrate_tonnes": 850, "concentrate_grade_pct": 28.5,
    --   "metal_kg": 242.25
    -- }
    fleet_json          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "availability_pct": 88.5, "utilisation_pct": 72.1,
    --   "by_type": {
    --     "haul_truck": {"total": 25, "operational": 22, "availability": 88.0},
    --     "excavator": {"total": 5, "operational": 4, "availability": 80.0}
    --   },
    --   "fuel_efficiency": 1.89
    -- }
    cost_json           JSONB,
    -- {
    --   "mining_cost_per_tonne_cents": 420,
    --   "processing_cost_per_tonne_cents": 850,
    --   "maintenance_cost_cents": 2500000,
    --   "fuel_cost_cents": 12750000
    -- }
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (site_id, period_type, period_start)
);
CREATE INDEX idx_prod_period ON rm_production_metrics (period_type, period_start DESC);
```

---

## Read Model: Environmental Compliance

```sql
CREATE TABLE rm_environmental_compliance (
    site_id             UUID NOT NULL,
    period_type         TEXT NOT NULL CHECK (period_type IN (
                            'daily','weekly','monthly','quarterly','yearly'
                        )),
    period_start        DATE NOT NULL,
    readings_json       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "dust": {"avg_ug_m3": 38.5, "max_ug_m3": 48.2, "limit": 50.0, "exceedances": 0},
    --   "water_usage": {"total_m3": 75000, "recycled_pct": 78.5},
    --   "noise": {"avg_dba": 72, "max_dba": 85, "limit": 85},
    --   "tailings": {"freeboard_m": 12.5, "min_required_m": 3.0, "volume_m3": 450000}
    -- }
    emissions_json      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "ghg_tco2e": 5500, "scope1_tco2e": 4800, "scope2_tco2e": 700,
    --   "nox_tonnes": 12.5, "sox_tonnes": 3.2, "pm10_tonnes": 1.8
    -- }
    exceedances_json    JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "type": "dust", "point": "Crusher PM10",
    --   "value": 52.1, "limit": 50.0,
    --   "occurred_at": "2026-05-20T14:30:00Z",
    --   "corrective_action": "Water suppression increased",
    --   "resolved": true
    -- }]
    gri_json            JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "GRI 303": {"water_withdrawal_ml": 75, "water_recycled_pct": 78.5},
    --   "GRI 305": {"scope1_tco2e": 4800, "scope2_tco2e": 700},
    --   "GRI 306": {"tailings_volume_m3": 450000, "waste_tonnes": 120000}
    -- }
    gistm_json          JSONB,
    -- {
    --   "facility": "TSF-1", "consequence_class": "extreme",
    --   "freeboard_m": 12.5, "dam_safety_factor": 1.5,
    --   "last_independent_review": "2026-03-15",
    --   "emergency_plan_current": true
    -- }
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (site_id, period_type, period_start)
);
CREATE INDEX idx_env_period ON rm_environmental_compliance (period_type, period_start DESC);
```

---

## Example Event Replay: Predictive Maintenance Lifecycle

```sql
SELECT event_type, ce_time, actor_type,
       event_data, metadata
FROM event_store
WHERE stream_type IN ('asset', 'maintenance', 'ai')
  AND asset_id = 'asset-uuid'
  AND ce_time >= '2026-05-01'
ORDER BY ce_time;

-- Results show:
-- 1. runtime_hours_updated (12,000 hrs)
-- 2. vibration readings (multiple sensor events, vibration increasing)
-- 3. anomaly_detected (AI: vibration pattern matches hydraulic pump degradation)
-- 4. failure_prediction_generated (hydraulic_pump, p=0.72, RUL=340hrs)
-- 5. work_order_created (predictive WO, trigger: ai_prediction)
-- 6. parts_issued (hydraulic pump, seal kit)
-- 7. work_order_started
-- 8. component_replaced (old pump SN → new pump SN)
-- 9. work_order_completed (downtime: 8hrs, cost: $45,000)
-- 10. health_score_calculated (restored to 0.95)
```

### Edge-to-cloud sync replay

```sql
SELECT event_type, ce_time,
       metadata->>'edge_device_id' AS device,
       metadata->>'buffered_at' AS buffered,
       metadata->>'synced_at' AS synced,
       event_data
FROM event_store
WHERE site_id = 'site-uuid'
  AND actor_type = 'edge_device'
  AND metadata->>'synced_at' IS NOT NULL
ORDER BY (metadata->>'buffered_at')::TIMESTAMPTZ;
```

### MSHA incident audit trail

```sql
SELECT event_type, ce_time, actor_id, actor_role,
       event_data
FROM event_store
WHERE stream_type = 'safety'
  AND stream_id = 'incident-uuid'
ORDER BY sequence_number;

-- Results:
-- 1. incident_reported (within 15 min of occurrence)
-- 2. msha_report_submitted (7000-1 form data)
-- 3. incident_investigation_started
-- 4. incident_root_cause_identified
-- 5. incident_corrective_action_assigned
-- 6. incident_closed
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Infrastructure | 3 | event_store (partitioned), stream_snapshot, projection_checkpoint |
| Read Models | 5 | rm_fleet_status, rm_maintenance_dashboard, rm_safety_board, rm_production_metrics, rm_environmental_compliance |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Sensor telemetry as events** — vibration, temperature, pressure, and fuel readings are events in the asset stream. This unifies operational telemetry with business events (work orders, incidents) in one queryable store, enabling correlation analysis: "what sensor pattern preceded this failure?"

2. **`failure_predicted` event** — ML predictions are first-class events with probability, remaining useful life (RUL), and model version. This provides the audit trail for predictive maintenance: "why was this work order created? Because the model predicted failure with 72% confidence."

3. **Edge device actor type** — events from edge computing nodes at remote mine sites carry `edge_device` actor type with metadata including device ID, buffer timestamp, and sync timestamp. This provides the audit trail for delayed event ingestion during connectivity outages.

4. **Safety events as immutable stream** — MSHA requires incident reporting within 15 minutes and complete investigation records. The safety stream captures every step from initial report through investigation to closure. Point-in-time queries can verify regulatory compliance: "was this incident reported within the 15-minute window?"

5. **`rm_environmental_compliance` with GRI and GISTM mapping** — pre-computed environmental metrics mapped to GRI 14 topics and GISTM governance fields enable automated ESG report generation. The event chain provides data lineage for auditor verification.

6. **`rm_fleet_status` with predicted failure** — pre-computed fleet health with failure predictions enables the operations manager's dashboard without replaying sensor events. The `health_score` field enables fleet-wide prioritisation.

7. **Production events per shift and per cycle** — `shift_production_recorded` captures aggregate shift metrics, while `haul_cycle_completed` captures individual truck cycles. The read model aggregates both into shift/daily/weekly/monthly views.

8. **`component_replaced` event** — captures both old and new component serial numbers. This creates the component lifecycle chain for lifecycle costing analysis and enables traceability for safety-critical components.

9. **CloudEvents envelope** — `ce_source`, `ce_specversion`, `ce_type`, and `ce_time` follow the CloudEvents 1.0 specification. This enables interoperability with industrial event buses and SCADA historians using standard message formats.

10. **8 tables** — three infrastructure tables (event store, snapshots, checkpoints) plus five read models covering the core mining views: fleet status, maintenance planning, safety compliance, production tracking, and environmental monitoring.

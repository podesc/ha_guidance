# `intelbox_analytics` – Database Plan

This document describes the logical layout of the main analytics database on `intelbox`.

- DB name: `intelbox_analytics`
- Main schemas: `base`, `data`, `work`, `model`
- Purpose: store heating/Home-Assistant-related data and model artefacts in a way that is:
  - easy to query and visualise (Grafana, LAN Portal)
  - friendly to simple models and local LLMs
  - predictable and safe (no direct HA runtime dependencies)

No credentials or connection strings are stored here.

---

## 1. Schemas

### 1.1 `base` schema

Static or slow-changing metadata.

Planned tables:

- `base.zone`
  - id, name, description, area, volume

- `base.sensor`
  - id, name, kind (temperature, humidity, power, etc.), unit, zone, notes

- `base.actuator`
  - id, name, kind (setpoint, pump, valve, mode), zone, min/max, cycle limits

- `base.scenario`
  - id, name (e.g. "wet_westerly"), description, criteria (JSON)

- `base.policy`
  - id, name, mode (manual, auto, predictive, etc.), parameters (JSON)

All IDs are local integers or simple text keys. No IPs, hostnames, or secrets.

---

### 1.2 `data` schema

Time-series records and cleaned subsets.

Planned tables:

- `data.sample_raw`
  - timestamp
  - sensor_id
  - value
  - source (e.g. HA, test, manual)
  - quality flag (optional)

- `data.sample_clean`
  - timestamp
  - sensor_id
  - value_clean
  - quality flag
  - notes

These are the main “facts” for downstream features and models.

---

### 1.3 `work` schema

Derived features and intermediate computations.

Planned tables:

- `work.feature_window`
  - window_start, window_end
  - zone_id
  - features JSON (e.g. EWMA temps, slopes, degree-minutes)
  - labels for evaluation (comfort breach yes/no, etc.)

- `work.etl_log`
  - run_id, started_at, finished_at
  - status
  - counts of rows processed
  - notes

This schema is allowed to change more often; it is the scratchpad for ETL and experimentation.

---

### 1.4 `model` schema

Stores fitted models and metrics.

Planned tables:

- `model.thermal_fit`
  - id
  - zone_id
  - version
  - parameters JSON (e.g. RC model coefficients)
  - fitted_at
  - fit_metrics JSON (MAE, RMSE, etc.)

- `model.policy_test`
  - id
  - policy_id (from `base.policy`)
  - test_period
  - metrics JSON (kWh/HDH, breaches, cycles/day)
  - notes

This schema lets simple scripts and local LLMs reason about “what worked better” over time.

---

## 2. Relationship to other systems

- **Home Assistant** remains the live orchestration layer.
  - HA writes measurements to the `data` schema via ETL/bridge processes.
  - HA does not depend directly on this DB to function.

- **Grafana** connects read-only to `intelbox_analytics` to visualise:
  - raw and cleaned samples (`data`)
  - derived features (`work`)
  - model metrics (`model`)

- **LAN Portal** may later expose curated views backed by this DB (for example: zone summaries, sensor registries, and policy comparisons).

- **Local LLMs** can use:
  - `base` for context (zones, sensors, actuators)
  - `data` / `work` for numeric patterns
  - `model` for summaries and evaluation

No connection details for HA, Grafana, LAN Portal, or LLM stacks are stored here.

---

## 3. First implementation steps

1. Create DB `intelbox_analytics` on `intelbox` Postgres.
2. Create schemas `base`, `data`, `work`, `model`.
3. Start with minimal tables:
   - `base.zone`
   - `base.sensor`
   - `data.sample_raw`
4. Add Grafana datasource and a simple “raw samples” panel.
5. Extend from there as ETL and control logic mature.

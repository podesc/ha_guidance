# Home Assistant Clean Build & AI-Box Architecture (v1)

### Footnote  
Where possible, all development should stay within these boundaries. Deviations must follow a process: *evaluation → experiment (sandbox) → commission/bake-in*. Use ADRs (Architecture Decision Records) to record every deviation or major design choice.

0) Objectives
	•	Sanitise HA: consistent entity/device model, predictable areas, minimal noise, disciplined automations.
	•	Stable data layer: external SQL (PostgreSQL/TimescaleDB) for long‑term history + curated domain tables for AI.
	•	Clear split of concerns: HA = orchestration + state; AI boxes = heavy compute, analytics, embeddings, ETL.
	•	Reproducible: Git‑tracked config, deterministic backups, environment notes per host.

⸻

1) Target Topology (by host)

Pi5 (Home Assistant OS or Container)
	•	HA Core
	•	Add‑ons: Mosquitto (MQTT), ESPHome, Samba/SSH, Studio Code Server
	•	Integrations only (no heavy ML, no Grafana/DB here)
	•	Recorder: remote PostgreSQL on AIU22

AIU22 (Intel 12400)
	•	PostgreSQL 16 + TimescaleDB
	•	MQTT bridge (optional) for inter‑LAN routing
	•	Node‑RED (optional) for complex flows you don’t want in HA YAML
	•	Whisper/ASR, RAG services (existing)
	•	Ingestion workers: HA → SQL ETL (listen on MQTT/webhooks) + retention jobs

FUB22 (AMD)
	•	Grafana + Loki/Promtail (logs)
	•	Secondary data services (e.g., DuckDB for ad‑hoc, Parquet lake)
	•	Frigate later (GPU/TPU when ready) with NVR disk

Network
	•	mDNS constrained; prefer static IPs + hostnames
	•	Optional: VLAN: “OT/IoT” for ESPs/Cams; HA/AI boxes on “Core LAN”

⸻

2) Naming & Structure (non‑negotiable)
	•	Areas: Physical rooms/zones only. No virtual/logical areas.
	•	Devices: Building/Zone – Make Model (ShortLocation) e.g., Main House/Kitchen – Shelly 1PM (Under‑sink)
	•	Entities: domain.zone_object → sensor.kitchen_extract_cfm, switch.workshop_ev_relay
	•	Unique IDs: ensure every entity has one for painless renames.
	•	Tags (HA): Use for cross‑cutting sets: safety, critical, noisy, island

⸻

3) HA Configuration Layout (Git‑friendly)

/config
  configuration.yaml
  secrets.yaml
  packages/
    energy.yaml
    safety.yaml
    hvac.yaml
    presence.yaml
  automations/
    _blueprints/
    core/
    alerts/
  includes/
    recorder.yaml
    mqtt.yaml
    template.yaml
    helpers.yaml
  esphome/ (project subdir, not compiled here if you prefer external builder)
  scripts/
  blueprints/ (only the ones you actually use)

	•	Use packages to co‑locate sensors+helpers+automations per domain.
	•	Keep UI‑created automations, then migrate stable ones to YAML under automations/core.

⸻

4) Recorder → Postgres (AIU22)

Postgres: ha DB, owner ha_user, strong password, TLS if cross‑VLAN.

TimescaleDB: enable for states/events hypertables with 30–90d compressed retention; raw in Timescale, curated facts in separate schemas.

configuration.yaml

recorder: !include includes/recorder.yaml

includes/recorder.yaml

db_url: postgresql://ha_user:SECRET@aiu22.lan:5432/ha
auto_purge: false  # Timescale handles retention
exclude:
  domains:
    - media_player
    - updater
    - persistent_notification
  entities:
    - sensor.time
    - sensor.date
commit_interval: 1

Timescale retention jobs (psql):

SELECT add_retention_policy('states', INTERVAL '90 days');
SELECT add_compression_policy('states', INTERVAL '14 days');
ALTER TABLE states SET (timescaledb.compress, timescaledb.compress_segmentby = 'entity_id');


⸻

5) MQTT Discipline
	•	Broker: Mosquitto on Pi5; bridge to AIU22 only if needed.
	•	Topics: site/<building>/<room>/<device>/<metric> e.g., site/workshop/plantroom/extract/co2
	•	QoS: 1 for controls/critical; 0 for chatter.
	•	Discovery: Prefer HA MQTT discovery on but gated: only publish discovery for devices that pass your linting (name, class, unit).

⸻

6) ESPHome Discipline
	•	Use substitutions for name roots; set device_class, state_class, unit_of_measurement.
	•	Calibrate at source; don’t template in HA if device can do it.
	•	OTA passwords + api.encryption keys required.

⸻

7) Automations Strategy
	•	Golden rule: One purpose per automation. If/Then blocks are short.
	•	Alerting: Central helpers + alert component for persisting alarms.
	•	Critical cuts (e.g., island mode): prefer scripts that combine checks and set states, then trigger via physical switch + UI.
	•	Prefer input_number/input_boolean for tunables; expose in a single dashboard.

⸻

8) Dashboards (Minimal)
	•	Ops: Power flows, comms health (last‑updated age, Wi‑Fi RSSI), critical relays.
	•	Energy: PV, grid, EV, ASHP, battery. Time‑shift cards.
	•	Maintenance: Devices by firmware age, entities without area, entities w/o device_class, disabled entities.

⸻

9) Backups & Reproducibility
	•	Nightly HA full backup to AIU22 via Samba/NFS + weekly off‑site (rclone to cloud bucket). Encrypt at rest.
	•	Export HA config to Git (exclude secrets.yaml); keep a secrets.example.yaml.
	•	Document add‑on versions in /config/VERSIONS.md.

⸻

10) Security
	•	HA user auth only; no long‑lived tokens unless scoped + expiry tracked.
	•	Reverse proxy (optional) with mTLS if exposing; otherwise no WAN exposure.
	•	Limit inter‑VLAN with firewall; allow HA→AI boxes on 5432/tcp, 1883/tcp.

⸻

11) AI Integration Pattern
	•	Flow A (Preferred): Device → MQTT → HA state → Postgres (recorder) → Timescale retention
	•	Flow B (Curated Facts): HA automation/webhook pushes summaries to AIU22 aiops schema tables (e.g., daily energy, peak loads, comfort violations).
	•	Flow C (Command Loop): AI job emits MQTT site/…/cmd topics; HA subscribes and enforces policy (allowlist) before action.

Example: Curated facts (SQL schema)

CREATE SCHEMA IF NOT EXISTS aiops;
CREATE TABLE IF NOT EXISTS aiops.daily_energy (
  day date primary key,
  house_kwh numeric,
  workshop_kwh numeric,
  ev_kwh numeric,
  pv_kwh numeric,
  battery_in_kwh numeric,
  battery_out_kwh numeric,
  notes text
);

HA automation pushes daily rollups (pseudocode): schedule 00:10, compute via HA utility_meter sensors, POST to AIU22 webhook.

⸻

12) Grafana (FUB22)
	•	Datasource: Postgres/Timescale.
	•	Dashboards: Energy, HVAC COP, MQTT/ESP uptime, Network RTT.
	•	Provision dashboards in Git; avoid HA Lovelace for long time‑series analysis.

⸻

13) Migration/Sanitisation Plan (5 passes)

Pass 0 – Freeze
	•	Disable discovery sprawl (Zeroconf tweaks) and new integrations auto‑add.
	•	Snapshot.

Pass 1 – Inventory
	•	Export /.storage/core.entity_registry + core.device_registry (read‑only) to CSV for audit.
	•	Report: entities without area, w/o device, disabled/unavailable, duplicate names.

Pass 2 – Prune & Normalize
	•	Delete orphan devices; disable noisy entities (battery voltage spam, etc.).
	•	Apply naming/areas.
	•	Ensure device_class/state_class/unit.

Pass 3 – Recorder & MQTT
	•	Cut over recorder to Postgres; validate write throughput; apply Timescale jobs.
	•	Standardize MQTT topics; enforce discovery only for linted devices.

Pass 4 – Automations
	•	Consolidate UI automations; extract to YAML for stable ones; add alerts + ops dashboard.

Pass 5 – Backups & Git
	•	Wire nightly backups; init Git; add VERSIONS.md and README.md per host.

⸻

14) Health & Linting (suggested sensors)
	•	sensor.ha_db_queue_depth (estimate via events_out - events_in if exposed)
	•	sensor.postgres_replication_lag (0 expected)
	•	sensor.mqtt_messages_rate
	•	binary_sensor.device_offline_xmin (template on last_changed age)

⸻

15) Minimal Config Snippets

MQTT in configuration.yaml

mqtt: !include includes/mqtt.yaml

includes/mqtt.yaml

broker: 127.0.0.1
birth_message:
  topic: ha/status
  payload: online
will_message:
  topic: ha/status
  payload: offline

Template sanity (entity age)

template:
  - sensor:
      - name: "HA Entities Stale (>1h)"
        state: >-
          {{ states | selectattr('last_changed','defined')
                     | selectattr('state','!=','unavailable')
                     | selectattr('last_changed','lt', now() - timedelta(hours=1))
                     | list | count }}


⸻

16) What Not To Do
	•	Don’t store embeddings/AI artefacts in the ha schema. Keep AI tables in aiops.
	•	Don’t let discovery create junk. Every device gets curated names before enabling.
	•	Don’t overfit logic into one mega‑automation. Keep it legible.

⸻

17) Next Actions (ordered)
	1.	Decide HA OS vs Container on Pi5 (recommend HA OS for simplicity + add‑ons).
	2.	Stand up Postgres+Timescale on AIU22; create ha and aiops schemas.
	3.	Wire recorder to Postgres; apply retention/compression.
	4.	Enforce naming/areas; prune entities.
	5.	Standardize MQTT; migrate ESPHome configs to substitutions.
	6.	Add daily rollup automation → aiops.daily_energy webhook.
	7.	Stand up Grafana on FUB22; import starter dashboards.
	8.	Turn on backups + Git; document versions.

⸻

18) Appendices
	•	Webhooks: create HA rest_command for aiu22 endpoint; HA posts JSON payloads.
	•	Policy Gateway: optional Node‑RED on AIU22 to validate cmd topics before HA acts.
	•	Future: Frigate on FUB22; publish motion/objects to MQTT; HA uses as binary_sensors.

⸻

End of v1. Mark up changes and I’ll iterate.


Got it, P. I’ve drafted a tight, step-by-step architecture + sanitisation blueprint in the canvas: “Home Assistant Clean Build & AI-Box Architecture (v1)”. It covers:
	•	Host topology (Pi5 = HA; AIU22 = Postgres/Timescale + ETL; FUB22 = Grafana/logs)
	•	Naming/areas conventions and a Git-friendly config layout
	•	Recorder → Postgres setup with Timescale retention jobs
	•	MQTT/ESPHome discipline, automation strategy, minimal dashboards
	•	Backups/security, AI integration patterns (curated facts schema), and a 5-pass clean-up/migration plan
	•	Snippets you can drop in (recorder/mqtt/templates)

If you want me to turn any section into concrete files (e.g., includes/recorder.yaml, Timescale SQL, or a starter packages/energy.yaml), say which and I’ll produce them ready to paste.
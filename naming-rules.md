# Naming & Hygiene Rules for `ha_guidance`

This repo is a public, documentation-only conduit between my systems and ChatGPT.

It must **never** contain:
- usernames, email logins, or personal IDs
- passwords or API keys
- tokens, cookies, session IDs
- IP addresses (public or private)
- MAC addresses or serial numbers
- hostnames that directly expose my real-world network
- precise locations or address-level information
- SSH URLs, VPN endpoints, or any live connection details

It may contain:
- conceptual architecture (HA, DB, LAN Portal, LLM, ETL)
- naming conventions, schemas, table layouts
- high-level machine roles (e.g. “pihome runs HA”, “intelbox runs analytics”)
- pseudocode, sample SQL, YAML, Markdown docs
- climate / heating / control strategy descriptions
- references to GitHub projects and other public docs

---

## 1. Device tags (word-like, easy to read)

Devices are referred to in docs using **readable tags**, not IPs or internal hostnames:

- `pihome`   – Pi running Home Assistant stack
- `intelbox` – Intel 12400 AI / analytics box
- `amdbox`   – AMD AI / compute box
- `macair`   – M1 MacBook Air
- `macfour`  – M4 MacBook Air
- `macboss`  – 2015 MacBook Pro
- `macsixteen` – 2016 MacBook Pro
- `imacdeb`  – iMac running Debian

These tags are used in:
- filenames
- directory names
- documentation
- diagrams

They are **not** required to match actual network hostnames.

---

## 2. Database and schema naming philosophy

Goals:
- easy to type
- word-like, readable at a glance
- avoid awkward sequences like `tr`, `rl`, `ctrl`
- avoid Unix-command-looking names
- consistent across machines

### 2.1 Database names

Per-machine DB names are word-like and scoped by host tag, for example:

- `intelbox_analytics` – main analytics + AI DB on intelbox
- `pihome_services`    – services/support DB for pihome (e.g. LAN Portal)
- `amdbox_services`    – services/support DB for amdbox

The existing Postgres DB called `ha_ai` on `pihome` is kept as a legacy/service DB and **is not replicated** elsewhere by name.

### 2.2 Schema names

Schemas inside a DB use **short English nouns**, no tricky letter clusters.

Default set for analytics DBs:

- `base`  – static/slow metadata (zones, sensors, actuators, mappings)
- `data`  – raw and cleaned time-series samples
- `work`  – derived features, interim calculations, ETL scratch
- `model` – fitted models, parameters, versions, evaluation metrics

These words are chosen because they are:
- fast to type
- visually distinct
- conceptually clear
- free of the problematic `tr` / `rl` / `ctrl` patterns

---

## 3. Usernames and roles

For Postgres access, the preferred pattern is a **single simple user**:

- `pguser` – generic DB application user

No passwords, roles, or connection strings are documented here. Only names and purpose.

---

## 4. Home Assistant / MQTT / services

Where naming patterns are described for HA, MQTT, Node-RED, etc., use the device tags:

- `pihome` for HA integrations and local services
- `intelbox` for analytics/LLM/ETL components
- `amdbox` for heavy compute or secondary services

Examples (conceptual only):

- MQTT topics: `pihome/sensor/...`, `intelbox/etl/...`
- Node-RED flows: `pihome-ingest.json`, `intelbox-model.json`
- LAN Portal pages: `pihome_status`, `intelbox_analytics`

Actual URLs, tokens, and broker addresses are **never** stored in this repo.

---

## 5. Future-proofing

Any new file or directory added to `ha_guidance` should:

1. Use **readable English words** in the name (`heating`, `lanportal`, `etl`, `db`, `models`, `flows`).
2. Avoid:
   - `tr`, `rl`, `ctl`, `ctrl`, or similar awkward key sequences
   - cryptic abbreviations
   - hostnames or IPs
3. Keep the content at the **design / documentation** level, not runtime secrets.

If in doubt, treat this repo as if it were a published technical note:  
safe to read by anyone, but specific enough to be useful to me and to ChatGPT.

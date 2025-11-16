# System Overview (High-Level, No Secrets)

## Hosts

- **PiHome** – Raspberry Pi 5, runs:
  - Home Assistant (core automations, dashboards)
  - Postgres (`ha_ai` DB with HA recorder + `coredata` schema)
  - LAN Portal (nginx + Filebrowser)
  - Grafana
  - Mosquitto (MQTT broker)

- **AIU22** – Intel Ubuntu server, runs:
  - Postgres (`opsdb`, future analytics)
  - [add: docker stacks / services, but no IPs, no credentials]

- **FUB22** – AMD Ubuntu server, runs:
  - Syncthing
  - [add: anything else notable]

- **Macs (M1, M4, others)** – client / admin machines:
  - Primary SSH + browser UI
  - GitHub Desktop for `ha_guidance`

## Core Services

- **Home Assistant**  
  - DB: `ha_ai` (Postgres on PiHome)  
  - Tables we treat as HA-owned: `states`, `events`, `statistics`, etc.  
  - Our schema: `coredata` (analytics-friendly tables & views).

- **LAN Portal**  
  - Nginx on PiHome (`:8090`)
  - Editor (Filebrowser) on PiHome (`:8092`)
  - Docs mount at `/srv/lanportal/docs` → includes `ha_guidance` checkout.

- **Grafana**  
  - PiHome (`:3030`)
  - Read-only dashboards over Postgres / MQTT.

- **Syncthing**  
  - Nodes: PiHome, FUB22, AIU22, Macs (details kept high-level).

## Security & Data Rules

- No passwords, tokens, or private IP details go into `ha_guidance`.
- No container `docker-compose.yml` with secrets in this repo.
- DB access here is **described**, not credentialled.

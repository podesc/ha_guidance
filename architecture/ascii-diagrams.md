# ASCII Architecture Diagrams

Text-only versions of the key diagrams.  
These mirror the Mermaid diagrams but are robust in any environment.

---

## 1. Layer Stack (Master Architecture)

WORLD LAYER
  - external info, weather, tariffs, zones, inventory

  ↓

SENSORS OUTSIDE
  - outdoor climate, wind, rain, boundary sensors

  ↓

SENSORS INSIDE
  - room temps, humidity, flow and return, power, occupancy

  ↓

PROCESSING LAYER
  - ETL, cleaning, feature building
  - NO policy or authority here

  ↓

COGNITION / REVIEW LAYER
  - predetermined rules and modes
  - human input (nudges, temporary modes, structural proposals)
  - AI input (predictions, suggestions, experiment proposals)
  - review-required cases (uncertainty, experiments, exceptions)

  ↓

ACTUATION LAYER
  - Home Assistant, pumps, valves, setpoints, EV charging, devices

Side channel:
  PROCESSING  →  DATA PLANE (all DBs, tables)
  ACTUATION   →  DATA PLANE
  DATA PLANE  →  COGNITION / REVIEW

Shared services (used by many layers):
  - TIME SERVICE
  - SAFETY INTERLOCKS
  - INVENTORY / ID / LOCATION

History stream:
  - fed by Processing, Cognition, Actuation
  - append-only logs, configs, experiments

---

## 2. Cognition / Review Internals

COGNITION / REVIEW LAYER

  Predetermined:
    - rules, modes, defaults

  Human Input:
    - nudges
    - temporary modes
    - structural proposals

  AI Input:
    - predictions
    - suggestions
    - experiment proposals

  Review Required:
    - uncertainty
    - exceptions
    - experimental approval

All four feed into:

  VALIDATED DECISION
    ↓
  ACTUATION LAYER

---

## 3. History Stream

HISTORY STREAM

  Event Log:
    - actions
    - readings
    - overrides
    - decisions

  Config Log:
    - before → after
    - proposer
    - approver
    - when applied

  Experiment Log:
    - experiment definition
    - window (start, end)
    - rollback target
    - metrics observed
    - outcome (kept or discarded)

All feed into:

  IMMUTABLE APPEND-ONLY RECORD
    - no edits
    - corrections are appended, not rewritten

---

## 4. Policy Model

POLICY (Human-selected)

  Modes:
    - comfort_first
    - balanced
    - cheap_first

  Each mode defines how strongly the system values:
    - comfort
    - cost
    - stability

  Flow:

    POLICY
      ↓
    AI PROPOSALS (only within allowed policy bounds)
      ↓
    REVIEW DECISIONS (accept / reject / treat as experiment)
      ↓
    ACTUATION LAYER

Humans can always apply short-lived overrides (feelings).  
System returns to the selected policy once overrides expire.

---

## 5. Overview Summary

WORLD
  ↓
SENSORS OUTSIDE
  ↓
SENSORS INSIDE
  ↓
PROCESSING (ETL, features)
  ↓
COGNITION / REVIEW
  ↓
ACTUATION (HA, devices)

Side flows:
  PROCESSING  →  DATA PLANE (all DBs, tables)
  ACTUATION   →  DATA PLANE
  DATA PLANE  →  COGNITION / REVIEW

Shared services:
  - TIME SERVICE  (now, tariffs, calendars, dependencies)
  - SAFETY        (hard limits, safe states)
  - INVENTORY     (identity, location, asset mapping)

HISTORY STREAM:
  - receives events and configs from Processing, Cognition, Actuation
  - remains append-only and is the basis for learning and forensics

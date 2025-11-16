System Architecture — Layers, Services, Policy, and History

This document defines the conceptual architecture for the Home Automation & AI-Control system.
It is implementation-neutral, device-neutral, and designed to scale from basic rule-based HA to AI-assisted predictive control.

⸻

1. Layers (Vertical Pipeline)

The system is organised into Layers.
Each layer has a clear role and cannot “reach around” the one above or below.

1. World Layer
	•	External weather (temperature, wind, rain, cloud)
	•	Tariffs and schedules
	•	Zone definitions
	•	Inventory, locations, asset mapping (PDA domain)
	•	High-level human constraints (“don’t run dryer at night”)

⸻

2. Sensor Layers

2.1 Sensors Outside
	•	External temperature
	•	Wind, rain, illumination
	•	Cameras, boundary sensors
	•	External power feeds / sub-panels where applicable

2.2 Sensors Inside
	•	Room temperatures & humidity
	•	Flow/return temperatures
	•	Electricity / circuit monitoring
	•	Occupancy indicators
	•	Local asset sensors (fridge power, boiler data, etc.)

⸻

3. Processing Layer (ETL + Features)

This layer prepares data and does not make decisions.
	•	Timestamp alignment
	•	Cleaning and validation
	•	EWMA smoothing
	•	Slopes (dT/dt and power gradients)
	•	Degree-minutes / thermal load features
	•	Occupancy likelihood
	•	Feature salience
	•	Prepared inputs for models

All decision-making happens above.

⸻

4. Cognition / Review Layer

This is the root of authority and the meeting point of:
	1.	Predetermined logic
	•	Simple rules
	•	Modes
	•	Structural defaults
	•	Safety policies (min/max, bans, limits)
	2.	Human input
	•	Nudges (short-term, auto-expiring)
	•	Temporary modes
	•	Structural proposals (require review)
	3.	AI input
	•	Predictions (thermal, cost, comfort)
	•	Suggested actions
	•	Contextual alerts
	•	Experimental proposals
	4.	Review required
	•	When uncertainty is high
	•	When conditions are unusual
	•	When structural change is requested
	•	When experiments need approval

Role of this Layer
	•	Approves / rejects AI proposals
	•	Approves structural changes
	•	Approves experiments (time-boxed bundles with rollback)
	•	Ensures that all decisions pass through authority
	•	Sends validated decisions to the Actuation Layer

This layer enforces governance, not safety.
Safety is below, always-on, and bypasses authority.

⸻

5. Data Layer (Data Plane)

The Data Layer is not one database.
It is all databases and all tables across the system, including:
	•	telemetry history
	•	features
	•	model outputs
	•	experiments (start, end, rollback)
	•	evaluation metrics
	•	structural configuration versions
	•	dense/long “truth” logs

Conceptually: T1…Tx = the entire data plane.

This layer is read/write for the system, but append-only for history.

⸻

6. Actuation Layer

Where decisions become actions.
	•	Heat pump modes
	•	Pump/valve control
	•	Setpoint scheduling
	•	EV charging control
	•	Lighting states
	•	HA automation triggers
	•	“Do X at Time Y” commands
	•	Subsystem resets and fallbacks

This layer must obey:
	•	decisions from Cognition
	•	hard limits from Safety
	•	schedules from Time Service

Actuation events are written back to History.

⸻

2. Shared Services (Cross-Cutting)

These are not layers; all layers may call them.

⸻

2.1 Time Service

A universal mechanism providing:
	•	“now”
	•	tariff profile
	•	calendar schedule
	•	cheap/expensive windows
	•	cooldown periods
	•	dependency timing (PM-style rules)

Time is not encoded as logic.
It is simply a shared reference for everything else.

⸻

2.2 Safety Interlock Service

An absolute enforcement layer:
	•	Maximum/minimum temperatures
	•	Maximum cycle rates
	•	Forbidden modes
	•	Physical safe states
	•	Sanity checks

Neither Human nor AI can override Safety.

⸻

2.3 PDA: Identity, Inventory, Location Service

Provides:
	•	rooms, zones, structure
	•	assets and their identity (persistent across replacement/moves)
	•	sensor → asset binding
	•	location context for all incoming data

This prevents downstream logic from needing to reconstruct “where” and “what” every time.

⸻

3. History Stream (Truth Layer)

A dedicated, append-only stream capturing:

3.1 Event Log
	•	sensor readings
	•	actuator events
	•	decisions applied
	•	overrides
	•	AI proposal timestamps
	•	review outcomes

3.2 Configuration Log

Every structural change, including:
	•	before → after diff
	•	who proposed (Human/AI)
	•	who approved (Review)
	•	when applied
	•	what triggered the change

3.3 Experiment Log
	•	experiment definition
	•	window (start/end)
	•	rollback target
	•	metrics observed
	•	evaluation outcome
	•	whether the new config was retained or discarded

3.4 Immutability Rules
	•	history is never edited
	•	corrections are appended, not rewritten
	•	forensic trace of all behaviour is preserved

History is the audit trail, the truth record, and the place where seasonal learning is performed.

⸻

4. Policy Model (Comfort / Cost / Stability)

Policy is intentionally simple and human-chosen.

The system tries to optimise:
	1.	Comfort – avoid cold/hot discomfort
	2.	Cost – avoid unnecessary energy spend
	3.	Stability – avoid thrash, cycling, noise

Policies define the weighting between them:
	•	comfort_first
	•	balanced
	•	cheap_first

AI does not micro-encode tariff logic.
Instead:
	•	AI proposes actions consistent with the chosen policy
	•	Review validates or rejects proposals
	•	Humans may override temporarily (feelings)
	•	System snaps back to policy once overrides expire

Tariff is just an input, not a philosophy.

⸻

5. System Philosophy

The entire architecture is built on four principles:
	1.	Authority lives in the Review layer.
	2.	Safety is absolute and below all authority.
	3.	History is immutable truth.
	4.	AI proposes; humans decide.

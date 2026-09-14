# ADR-0002: Vehicle and Powertrain Model

* **Status:** Accepted
* **Date:** 2026-09-14
* **Stage:** Stage 0 — Domain & Engineering Foundation

## Context

OmniFleet must support multiple vehicle types with different energy sources and different valid activity types.

The initial product direction includes:

* Internal Combustion Engine vehicles
* Battery Electric Vehicles
* Plug-in Hybrid Electric Vehicles

A naive model might describe the vehicle using a single field such as:

```text
fuel_type = petrol | diesel | electric | phev
```

This is insufficient.

`petrol`, `diesel` and `LPG` describe an energy source.

`ICE`, `EV` and `PHEV` describe vehicle powertrain behavior.

Those are different concepts.

A Plug-in Hybrid, for example, may use:

```text
petrol
+
electricity
```

and therefore generate both:

```text
RefuelEvent
ChargeEvent
```

A multi-fuel combustion vehicle may also use more than one fuel source.

The vehicle model must therefore describe capabilities without forcing unrelated concepts into one enum.

---

## Decision

OmniFleet will model a Vehicle using three distinct concepts:

```text
Vehicle
├── Powertrain
├── Fuel Configuration
└── Electric Configuration
```

The initial supported powertrain types are:

```text
ICE
EV
PHEV
```

The powertrain determines which event types are valid.

Energy configuration describes what the vehicle actually consumes.

---

# Powertrain

`Powertrain` represents the high-level propulsion architecture of the vehicle.

Initial values:

```text
ICE
EV
PHEV
```

---

## ICE

An Internal Combustion Engine vehicle.

Supports:

```text
RefuelEvent
```

Does not support:

```text
ChargeEvent
```

unless the vehicle is later reclassified into a different supported powertrain model.

Typical energy sources include:

```text
Petrol
Diesel
LPG
CNG
```

---

## EV

A Battery Electric Vehicle.

Supports:

```text
ChargeEvent
```

Does not support:

```text
RefuelEvent
```

An EV has an electric-energy configuration and no combustion-fuel configuration.

---

## PHEV

A Plug-in Hybrid Electric Vehicle.

Supports both:

```text
RefuelEvent
ChargeEvent
```

A PHEV therefore contains:

```text
Fuel Configuration
+
Electric Configuration
```

A typical event stream may look like:

```text
Refuel
Charge
Charge
Refuel
Charge
Refuel
```

These event types remain independent observations.

OmniFleet must not attempt to convert liters and kilowatt-hours into a single arbitrary "energy consumption" unit for normal user-facing analytics.

Common monetary metrics may be calculated later where appropriate.

---

# Why Powertrain and Energy Source Are Separate

The following model is rejected:

```text
vehicle.fuel_type =
petrol
diesel
electric
phev
```

because the values do not belong to the same conceptual category.

For example:

```text
PHEV
```

does not tell the application whether the combustion side uses:

```text
Petrol
Diesel
```

or another fuel.

Instead:

```text
Vehicle
powertrain = PHEV

Fuel Configuration
fuel_type = Petrol

Electric Configuration
battery_capacity = ...
```

represents the real-world vehicle more accurately.

---

# Fuel Configuration

A `FuelConfiguration` describes the fuel-based energy capabilities of a compatible vehicle.

It may exist for:

```text
ICE
PHEV
```

and must not exist for a pure EV.

Conceptually:

```text
FuelConfiguration
- supported fuel type(s)
- primary fuel type
- optional tank capacity
```

---

## Initial Fuel Types

The initial known fuel types are:

```text
Petrol
Diesel
LPG
CNG
```

This ADR defines the domain categories, not their final database enum representation.

Additional fuel types may be introduced later when there is a demonstrated use case.

Examples might include:

```text
E85
Hydrogen
```

but they are not required for the first product versions.

---

# Multi-Fuel Vehicles

Some combustion vehicles can use more than one fuel.

Example:

```text
Petrol + LPG
```

The domain model must not permanently prevent this.

Therefore a vehicle's fuel configuration should conceptually allow one or more fuel types.

Example:

```text
Vehicle
powertrain = ICE

Fuel Configuration
supported_fuels:
- Petrol
- LPG
```

A RefuelEvent records the actual fuel type used for that specific transaction.

Example:

```text
RefuelEvent A
fuel_type = Petrol

RefuelEvent B
fuel_type = LPG
```

This allows fuel-specific analytics later.

---

## Primary Fuel

A multi-fuel vehicle may optionally have a primary or default fuel type.

Example:

```text
supported_fuels:
- Petrol
- LPG

primary_fuel:
LPG
```

This may be useful for:

* form defaults
* filtering
* dashboard presentation

It does not restrict which supported fuel may be recorded.

---

# Electric Configuration

An `ElectricConfiguration` describes the electrically rechargeable part of a vehicle.

It may exist for:

```text
EV
PHEV
```

and must not exist for a normal ICE-only vehicle.

Conceptually:

```text
ElectricConfiguration
- optional battery capacity
- optional usable battery capacity
```

Other electrical metadata may be added later if real product requirements justify it.

---

# Battery Capacity

Battery capacity is useful metadata but is not required to record a ChargeEvent.

A user should still be able to use OmniFleet if they do not know the exact battery capacity.

Therefore:

```text
battery_capacity_kwh
```

is optional.

If introduced, the model should eventually distinguish between concepts such as:

```text
gross battery capacity
usable battery capacity
```

rather than assuming every manufacturer-reported capacity means the same thing.

This distinction is intentionally deferred until necessary.

---

# Fuel Tank Capacity

Fuel tank capacity is also optional metadata.

A RefuelEvent does not require the application to know the physical tank capacity.

Therefore:

```text
tank_capacity_l
```

must not be required for basic refuel tracking.

It may later be useful for:

* validation warnings
* analytics
* plausibility checks

but should not block data entry.

---

# Event Capabilities

The initial mapping is:

| Powertrain | RefuelEvent | ChargeEvent |
| ---------- | ----------: | ----------: |
| ICE        |         Yes |          No |
| EV         |          No |         Yes |
| PHEV       |         Yes |         Yes |

The application must validate that recorded event types are compatible with the Vehicle's powertrain.

Example:

```text
Vehicle:
powertrain = EV

Create RefuelEvent
→ REJECT
```

and:

```text
Vehicle:
powertrain = ICE

Create ChargeEvent
→ REJECT
```

---

# HEV Vehicles

Conventional non-plug-in hybrid vehicles are intentionally **not modeled as a separate initial powertrain type**.

From OmniFleet's current energy-tracking perspective, a conventional HEV:

* receives fuel through RefuelEvents
* does not receive externally recorded electrical charging sessions

Therefore it can initially be represented as:

```text
powertrain = ICE
```

with optional future metadata describing hybrid technology.

This avoids introducing a new event capability that behaves identically to ICE from OmniFleet's current perspective.

If HEV-specific analytics later become meaningful, this decision may be revisited.

---

# Mild Hybrids

Mild hybrid vehicles are treated the same way as conventional HEVs for the initial product.

They use:

```text
powertrain = ICE
```

because their externally recorded energy source remains combustion fuel.

The presence of an internal electrical assistance system does not currently change the events OmniFleet records.

---

# Other Vehicle Technologies

The following are explicitly outside the committed initial model:

```text
Hydrogen Fuel Cell
Hydrogen ICE
Range Extender EV
Series Hybrid
Alternative synthetic fuels
```

This does not mean OmniFleet must never support them.

The model should avoid assumptions that make future extension unnecessarily difficult, but Stage 0 should not model unsupported technologies without a concrete requirement.

---

# Vehicle Identity

A Vehicle represents one physical vehicle tracked by a Household.

Conceptual identity metadata may include:

```text
make
model
year
registration / plate
nickname
```

The exact field set belongs to persistence and product UX decisions.

Vehicle identity must remain separate from its powertrain configuration.

Example:

```text
Vehicle
- Hyundai
- i30
- 2011

Powertrain
- ICE

Fuel Configuration
- Diesel
```

---

# Registration / Plate

Registration plate is useful identifying metadata but must not be treated as the permanent technical identity of a Vehicle.

Registration may change during the lifetime of a physical vehicle.

Therefore the internal OmniFleet Vehicle ID remains the stable identifier.

A plate may later be:

```text
optional
editable
historically tracked
```

depending on product requirements.

Historical plate tracking is deferred.

---

# Vehicle Year

Model year or registration year may be stored as metadata.

The exact semantic meaning must be made clear in the UI if introduced.

OmniFleet should avoid silently treating:

```text
manufacturing year
model year
first registration year
```

as identical concepts.

For the initial product, a simple optional year field is sufficient.

---

# Vehicle Configuration Changes

Some vehicle properties may change over time.

Examples:

* LPG conversion
* battery replacement
* tank replacement
* powertrain modification

Stage 0 does not introduce full configuration history.

The initial model assumes that vehicle configuration represents the current known state.

However, event history must preserve the energy type recorded on each event.

Example:

```text
RefuelEvent
fuel_type = Petrol
```

must remain historically correct even if the vehicle configuration changes later.

---

# Refuel Event Fuel Type

Every RefuelEvent must specify the actual fuel type used.

The event's fuel type must be one of the fuel types supported by the Vehicle at the time of recording.

Conceptually:

```text
Vehicle supported fuels:
Petrol
LPG

Refuel:
Diesel
→ REJECT
```

This prevents invalid data while still supporting multi-fuel vehicles.

---

# Charge Events

ChargeEvents belong to electrically rechargeable vehicles.

Compatible powertrains:

```text
EV
PHEV
```

The vehicle's electrical configuration provides metadata about the battery.

The ChargeEvent itself contains the actual observation.

Examples:

```text
energy added
start SoC
end SoC
charging cost
charging time
```

The vehicle configuration must not be used as a replacement for event-level measurements.

---

# Vehicle-Level Defaults

Some vehicle configuration values may act as UI defaults.

Examples:

```text
primary fuel
preferred currency
usual charging configuration
```

Defaults improve data-entry speed but do not override event-level truth.

Example:

```text
Vehicle primary fuel = Petrol

RefuelEvent fuel_type = LPG
```

is valid if LPG is a supported fuel.

---

# Alternatives Considered

## Alternative A — Single fuel_type Enum

Example:

```text
vehicle.fuel_type =
petrol
diesel
lpg
electric
phev
```

### Advantages

* very simple schema
* easy initial forms

### Disadvantages

* mixes powertrain and energy source
* cannot cleanly represent PHEV
* weak support for multi-fuel vehicles
* difficult to extend
* encourages event logic based on overloaded values

### Decision

Rejected.

---

## Alternative B — Separate Vehicle Types

Example:

```text
ICEVehicle
EVVehicle
PHEVVehicle
```

### Advantages

* strong type separation
* each vehicle model contains only relevant fields

### Disadvantages

* duplicates shared vehicle concepts
* complicates vehicle lists and household ownership
* makes common reporting more difficult
* risks inheritance-like domain complexity
* PHEV still overlaps both energy domains

### Decision

Rejected.

OmniFleet benefits more from a shared Vehicle model with explicit capabilities.

---

## Alternative C — Fully Capability-Based Model

Example:

```text
VehicleCapabilities
- can_refuel
- can_charge
```

without a powertrain type.

### Advantages

* highly flexible
* future vehicle technologies easy to express
* avoids fixed enum categories

### Disadvantages

* can produce nonsensical configurations
* loses useful semantic information
* harder to communicate in the UI
* requires more validation logic
* unnecessary flexibility for current requirements

### Decision

Rejected for the initial model.

Capabilities are derived from a meaningful powertrain type.

---

## Alternative D — Powertrain + Energy Configuration

Example:

```text
Vehicle
├── Powertrain
├── Fuel Configuration
└── Electric Configuration
```

### Advantages

* separates propulsion architecture from energy source
* represents PHEV naturally
* supports multi-fuel ICE vehicles
* allows optional capacity metadata
* keeps Vehicle shared across all types
* makes event compatibility explicit

### Disadvantages

* more complex than one enum
* requires consistency validation between powertrain and configuration

### Decision

Accepted.

---

# Invariants

The following rules are part of the domain model.

## ICE

An ICE vehicle must have a Fuel Configuration.

An ICE vehicle must not require an Electric Configuration.

It may record RefuelEvents.

It may not record ChargeEvents.

---

## EV

An EV must have an Electric Configuration.

An EV must not have a combustion Fuel Configuration.

It may record ChargeEvents.

It may not record RefuelEvents.

---

## PHEV

A PHEV must have:

```text
Fuel Configuration
and
Electric Configuration
```

It may record both:

```text
RefuelEvent
ChargeEvent
```

---

## Fuel Compatibility

A RefuelEvent fuel type must be supported by the Vehicle's Fuel Configuration.

---

## Capacities

Tank and battery capacities are metadata.

They are not required to record valid refuel or charging events unless a future calculation explicitly depends on them.

---

## Historical Events

Changing current Vehicle configuration must not silently rewrite historical event data.

---

# Initial MVP Behavior

The first ICE-focused vertical slice only needs to expose:

```text
Vehicle
powertrain = ICE

Fuel Configuration
supported fuel(s)
```

EV and PHEV support may be introduced later without redesigning the Vehicle identity model.

For the earliest product flow, a typical vehicle may therefore look like:

```text
Vehicle:
Hyundai i30

Powertrain:
ICE

Fuel Configuration:
Diesel
```

The broader model is established now so that the later EV/PHEV implementation does not require restructuring the core Vehicle domain.

---

# Persistence Guidance

The persistence layer should preserve the distinction between:

```text
powertrain
fuel configuration
electric configuration
```

The exact relational model is intentionally deferred.

Possible implementations include:

```text
vehicles
vehicle_fuel_configurations
vehicle_electric_configurations
```

or a simpler representation if it maintains the same domain semantics.

The database schema should not be allowed to collapse the model back into a single overloaded `fuel_type` field.

---

# Validation Scenarios

These scenarios should later become domain or integration tests.

## Scenario 1 — ICE Diesel

```text
Given:
Vehicle powertrain = ICE
Supported fuel = Diesel

When:
A Diesel RefuelEvent is recorded

Then:
The event is accepted
```

---

## Scenario 2 — Invalid ICE charging

```text
Given:
Vehicle powertrain = ICE

When:
A ChargeEvent is recorded

Then:
The event is rejected
```

---

## Scenario 3 — EV charging

```text
Given:
Vehicle powertrain = EV

When:
A ChargeEvent is recorded

Then:
The event is accepted
```

---

## Scenario 4 — Invalid EV refuel

```text
Given:
Vehicle powertrain = EV

When:
A RefuelEvent is recorded

Then:
The event is rejected
```

---

## Scenario 5 — PHEV

```text
Given:
Vehicle powertrain = PHEV
Fuel type = Petrol
Electric configuration exists

When:
A Petrol RefuelEvent is recorded
And:
A ChargeEvent is recorded

Then:
Both events are accepted
```

---

## Scenario 6 — Multi-Fuel ICE

```text
Given:
Vehicle powertrain = ICE
Supported fuels:
- Petrol
- LPG

When:
A Petrol RefuelEvent is recorded
And:
An LPG RefuelEvent is recorded

Then:
Both events are accepted
```

---

## Scenario 7 — Unsupported Fuel

```text
Given:
Vehicle supported fuel = Diesel

When:
A Petrol RefuelEvent is recorded

Then:
The event is rejected
```

---

## Scenario 8 — Unknown Tank Capacity

```text
Given:
Vehicle powertrain = ICE
Tank capacity is unknown

When:
A valid RefuelEvent is recorded

Then:
The event is accepted
```

---

## Scenario 9 — Unknown Battery Capacity

```text
Given:
Vehicle powertrain = EV
Battery capacity is unknown

When:
A valid ChargeEvent is recorded

Then:
The event is accepted
```

---

# Deferred Decisions

This ADR intentionally does not define:

* exact SQL tables
* enum storage strategy
* exact battery metadata
* gross vs usable battery capacity storage
* vehicle configuration history
* hydrogen support
* manufacturer-specific powertrain variants
* VIN support
* registration history
* energy-equivalent cross-powertrain metrics
* manufacturer API integration
* automatic vehicle specification lookup

These decisions are not required to establish the initial Vehicle domain.

---

# Consequences

## Positive

The Vehicle model remains consistent across ICE, EV and PHEV vehicles.

Powertrain and energy source are no longer mixed into one field.

PHEVs become a natural combination of two supported event domains rather than a special exception.

Multi-fuel combustion vehicles can also be represented without redesigning the model later.

---

## Negative

The model is more complex than a single `fuel_type` property.

The application must validate consistency between:

```text
Powertrain
Fuel Configuration
Electric Configuration
```

Some configuration concepts will require separate persistence structures.

This complexity is accepted because it represents real domain differences rather than infrastructure overhead.

---

# Result

OmniFleet will use a shared `Vehicle` model.

A Vehicle has a defined `Powertrain`:

```text
ICE
EV
PHEV
```

Energy configuration is modeled separately.

ICE vehicles use Fuel Configuration.

EV vehicles use Electric Configuration.

PHEVs use both.

Fuel types describe actual fuel sources rather than vehicle categories.

Multi-fuel combustion vehicles are supported conceptually from the beginning.

Tank and battery capacities remain optional metadata.

Event compatibility is determined by powertrain and energy configuration.

This decision is accepted as part of the Stage 0 foundation.

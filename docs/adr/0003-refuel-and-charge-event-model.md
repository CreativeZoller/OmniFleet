# ADR-0003: Refuel and Charge Event Model

* **Status:** Accepted
* **Date:** 2026-09-14
* **Stage:** Stage 0 — Domain & Engineering Foundation

## Context

OmniFleet tracks real-world vehicle energy events.

For combustion-powered vehicles this means refueling.

For electrically rechargeable vehicles this means charging.

These events form the primary historical dataset from which later analytics are derived.

Examples include:

```text
consumption
cost per kilometer
unit price trends
monthly spending
driver efficiency
```

Because these calculations may evolve over time, OmniFleet must preserve the original observations independently from derived values.

The event model therefore needs to answer:

* what was actually observed?
* what was entered by the user?
* what can be calculated?
* what information is mandatory?
* what information is optional?
* what relationships exist between consecutive events?

---

# Decision

OmniFleet will model fuel and electrical charging as two separate event types:

```text
RefuelEvent
ChargeEvent
```

They share several concepts:

```text
vehicle
recorded_by
odometer
time
money
currency
notes
```

but remain separate domain models because their measurements and semantics differ.

OmniFleet will not introduce a generic:

```text
EnergyEvent
```

as the primary domain entity at this stage.

---

# Why Separate Event Types

A RefuelEvent measures concepts such as:

```text
liters
fuel type
full tank
range before
range after
```

A ChargeEvent measures concepts such as:

```text
kWh
State of Charge
charging duration
charger type
power
```

Trying to force these into one universal event would create many fields that are meaningless for one side of the model.

Example:

```text
energy_event
- liters?
- kwh?
- full_tank?
- soc?
- charger_type?
```

This would weaken validation and domain clarity.

Therefore:

```text
RefuelEvent
```

and:

```text
ChargeEvent
```

remain distinct.

Unified reporting may later combine them at a reporting or query layer.

---

# RefuelEvent

A `RefuelEvent` represents one real-world fuel purchase or refueling action.

Conceptually:

```text
RefuelEvent
- id
- vehicle
- recorded_by
- occurred_at
- recorded_at
- odometer_km
- fuel_type
- liters
- unit_price
- total_amount
- currency
- is_full_tank
- range_before_km
- range_after_km
- onboard_avg_consumption_l100
- notes
```

This is a domain model, not the final SQL schema.

---

# Required Refuel Observations

For the initial product, a valid RefuelEvent requires:

```text
vehicle
occurred_at
odometer_km
fuel_type
liters
currency
is_full_tank
```

and sufficient monetary information to represent the transaction.

At least one valid cost representation must be available.

The exact monetary rules are defined below.

---

# Vehicle

Every RefuelEvent belongs to exactly one Vehicle.

The Vehicle must support RefuelEvents according to ADR-0002.

Valid:

```text
ICE
PHEV
```

Invalid:

```text
EV
```

---

# Recorded By

Every newly created event should preserve which authenticated User recorded it.

Conceptually:

```text
recorded_by
```

This is activity attribution.

It does not imply ownership of the Vehicle or event.

Historical attribution should survive later household membership changes where possible.

---

# Event Time vs Record Time

OmniFleet distinguishes between:

```text
occurred_at
```

and:

```text
recorded_at
```

Example:

```text
Refuel happened:
2026-09-14 17:35

Entered into OmniFleet:
2026-09-15 08:10
```

Both are valid facts.

`occurred_at` represents the real-world event.

`recorded_at` represents when OmniFleet received the record.

Analytics should normally use:

```text
occurred_at
```

rather than creation time.

---

# Odometer

A RefuelEvent records the vehicle's odometer at the event:

```text
odometer_km
```

The odometer is a raw observation.

It is used later to calculate distance between vehicle events.

For normal data entry:

```text
current_odometer >= previous_valid_odometer
```

is required.

However, historical corrections and instrument replacement may eventually require special workflows.

Therefore decreasing odometer readings are considered invalid for normal event creation but not an impossible real-world situation.

Correction behavior is deferred.

---

# Fuel Type

Every RefuelEvent records the actual fuel type used.

Example:

```text
Diesel
Petrol
LPG
CNG
```

The fuel type must be supported by the Vehicle's Fuel Configuration.

Example:

```text
Vehicle supported fuels:
Petrol
LPG

Refuel:
Diesel

→ REJECT
```

Fuel type belongs to the event because a multi-fuel vehicle may use different fuel types across different events.

---

# Fuel Quantity

The actual amount of fuel added is recorded as:

```text
liters
```

Rules:

```text
liters > 0
```

The quantity added is an observation.

It must not automatically be interpreted as:

```text
fuel consumed since previous event
```

because this is only reliably true under specific measurement conditions.

---

# Full vs Partial Refuel

Every RefuelEvent records whether the vehicle was filled to a known full reference point.

Initial representation:

```text
is_full_tank = true | false
```

This value is critical to later consumption calculations.

Example:

```text
30 liters added
is_full_tank = false
```

does not tell OmniFleet how much fuel was consumed since the previous event.

The value:

```text
liters
```

remains purchase quantity.

Consumption is derived separately.

---

# Monetary Observations

A fuel transaction may expose:

```text
liters
unit_price
total_amount
```

Mathematically:

```text
liters × unit_price ≈ total_amount
```

However, receipts may contain rounding differences.

Therefore OmniFleet must not assume perfect equality.

---

# Monetary Source of Truth

The transaction's `total_amount` is considered the authoritative monetary amount when explicitly provided by the user or receipt.

`unit_price` is also preserved as an observed value when available.

Example:

```text
liters = 42.17
unit_price = 1.579
total_amount = 66.58
```

Small differences caused by currency rounding are valid.

OmniFleet should not silently rewrite a user-provided `total_amount` to force exact multiplication.

---

# Missing Monetary Values

The UI may allow one monetary value to be derived from the others.

For example:

```text
liters
+
unit_price
```

can produce:

```text
total_amount
```

or:

```text
total_amount
/
liters
```

can produce an effective unit price.

The domain should preserve whether a value was:

```text
observed
or
derived
```

where this distinction becomes important.

The exact persistence representation of provenance is deferred.

---

# Currency

Every monetary RefuelEvent must include:

```text
currency
```

Examples:

```text
EUR
HUF
```

The currency applies to:

```text
unit_price
total_amount
```

within that transaction.

Cross-currency normalization is not part of the event itself.

---

# Range Before Refuel

Optional:

```text
range_before_km
```

This records the vehicle's displayed estimated remaining range immediately before refueling.

It is an onboard estimate, not a physical measurement of remaining fuel.

---

# Range After Refuel

Optional:

```text
range_after_km
```

This records the vehicle's displayed estimated remaining range after refueling and after the vehicle has updated its range estimate.

It may not be available immediately on every vehicle.

Therefore it remains optional.

---

# Onboard Average Consumption

Optional:

```text
onboard_avg_consumption_l100
```

This records the consumption value displayed by the vehicle.

Example:

```text
6.3 L/100km
```

It represents the car's own estimate.

It must remain separate from OmniFleet-calculated consumption.

---

# Notes

A RefuelEvent may include optional free-text notes.

Examples:

```text
mostly motorway
winter tires
roof box installed
very cold weather
```

Notes are not intended to become structured analytics inputs during the initial product stages.

---

# ChargeEvent

A `ChargeEvent` represents one external charging session.

Conceptually:

```text
ChargeEvent
- id
- vehicle
- recorded_by
- started_at
- ended_at
- recorded_at
- odometer_km
- start_soc_pct
- end_soc_pct
- energy_kwh
- total_amount
- currency
- charger_type
- power_kw
- notes
```

---

# Vehicle Compatibility

ChargeEvents are supported by:

```text
EV
PHEV
```

and rejected for:

```text
ICE
```

according to ADR-0002.

---

# Charging Time

A charging session may have:

```text
started_at
ended_at
```

For initial manual entry, at least one event time must be known.

The final UX may allow:

```text
ended_at only
```

for simple historical entry if the exact start time is unknown.

When both timestamps are available:

```text
ended_at >= started_at
```

must hold.

Charging duration is derived from these timestamps.

It should not become an independent source-of-truth value when both timestamps exist.

---

# Charge Odometer

A ChargeEvent may record:

```text
odometer_km
```

For the initial analytics model, odometer recording is expected to be required where consumption between charging observations is calculated.

The product UX may later distinguish:

```text
required for analytics
```

from:

```text
required to store the event
```

but the initial OmniFleet workflow should strongly require odometer readings for consistency with the vehicle history.

---

# State of Charge

A ChargeEvent may record:

```text
start_soc_pct
end_soc_pct
```

Valid range:

```text
0 <= SoC <= 100
```

Normally:

```text
end_soc_pct >= start_soc_pct
```

for a charging session.

Exceptional bidirectional charging or unusual workflows are not part of the initial scope.

---

# Energy Added

Electrical energy delivered during the session is represented as:

```text
energy_kwh
```

If present:

```text
energy_kwh > 0
```

This value represents observed delivered energy.

It should not automatically be inferred from:

```text
battery capacity × SoC difference
```

because:

* charging losses exist
* usable battery capacity may differ from nominal capacity
* battery-management systems contain hidden buffers
* SoC itself is an estimate

Therefore:

```text
energy_kwh
```

and:

```text
start/end SoC
```

remain independent observations.

---

# Charging Losses

The initial model does not attempt to calculate exact charging losses.

Future versions may distinguish:

```text
energy drawn from charger
energy stored in battery
```

if reliable data becomes available.

For Stage 0, `energy_kwh` should represent whatever delivered-energy value the user or charging provider reports.

The meaning should be documented by the UI where necessary.

---

# Charging Cost

A ChargeEvent may contain:

```text
total_amount
currency
```

and potentially an effective:

```text
unit_price_per_kwh
```

A charging session may use pricing models more complicated than:

```text
price × kWh
```

Examples include:

```text
session fee
time-based fee
parking fee
subscription discount
```

Therefore `total_amount` remains the authoritative actual transaction cost when available.

An effective price per kWh may be derived as:

```text
total_amount / energy_kwh
```

for reporting.

---

# Charger Type

Optional:

```text
charger_type
```

Initial categories may include:

```text
AC
DC
```

This is descriptive metadata.

A more detailed connector or charging-standard model is explicitly deferred.

---

# Charging Power

Optional:

```text
power_kw
```

This records a relevant charging-power observation where available.

It must not automatically be interpreted as:

```text
average session power
```

unless explicitly defined that way.

A charger may advertise:

```text
150 kW
```

while the vehicle actually charges at significantly less.

Therefore the semantic meaning of `power_kw` must be clarified before implementation.

For the initial model it remains optional metadata.

---

# Home Charging

A home charging session is still a normal ChargeEvent.

The fact that charging occurred at home belongs to the future Place domain.

Example:

```text
ChargeEvent
→ Place = Home
```

No separate `HomeChargeEvent` is needed.

---

# PHEV Events

A PHEV may create both event types.

Example:

```text
RefuelEvent
ChargeEvent
ChargeEvent
RefuelEvent
```

Each event retains its own native measurements.

OmniFleet does not attempt to merge the raw event streams.

Unified PHEV cost analytics may later consume both streams.

---

# Event Ordering

Vehicle events are primarily ordered by real-world event time, not database insertion order.

For a RefuelEvent this is:

```text
occurred_at
```

A ChargeEvent currently records:

```text
started_at
ended_at
```

The canonical ChargeEvent timestamp used for history ordering and calendar reports (`started_at` vs `ended_at`, or a derived `occurred_at`) is deferred until the EV vertical slice.

Vehicle history may additionally use odometer readings for validation.

An event entered later may have an earlier real-world timestamp.

Example:

```text
Event A recorded today
occurred 3 days ago

Event B recorded yesterday
occurred yesterday
```

Therefore database insertion order must not be treated as vehicle-history order.

---

# Editing Events

Events may need correction after creation.

Examples:

```text
wrong odometer
wrong liters
wrong amount
wrong timestamp
```

The initial product may support editing.

However, changing historical observations can affect every derived calculation after that event.

Therefore:

```text
edit raw event
→ recalculate affected derived metrics
```

must be the conceptual behavior.

Exact audit/versioning strategy is deferred.

---

# Deleting Events

Deleting historical events can change consumption segments and reports.

Therefore event deletion is not considered a simple isolated CRUD operation.

The eventual implementation must account for affected calculations.

Possible approaches include:

```text
hard delete
soft delete
correction event
audit history
```

The final strategy is deferred to persistence and audit design.

---

# Raw vs Derived Data

The following are raw event observations:

```text
odometer
liters
fuel type
full tank
range before
range after
onboard consumption
SoC
energy kWh
timestamps
transaction amount
currency
```

Examples of derived values:

```text
distance traveled
L/100km
kWh/100km
cost/km
effective unit price
charging duration
Driver vs Car ratio
```

Derived values must not replace the underlying observations.

---

# Alternatives Considered

## Alternative A — Generic EnergyEvent

```text
EnergyEvent
- liters?
- kwh?
- soc?
- full_tank?
```

### Advantages

* one table or API model
* unified event list

### Disadvantages

* many nullable fields
* weak semantics
* complex validation
* ICE and EV concepts become mixed
* PHEV support appears simpler while actually becoming less explicit

### Decision

Rejected.

Unified reporting can combine separate domain events later.

---

## Alternative B — Store Only Minimum Financial Data

Example:

```text
date
amount
liters
```

### Advantages

* very fast implementation
* simple UI

### Disadvantages

* prevents advanced consumption analytics
* loses odometer-based history
* cannot validate real-world efficiency
* cannot support the planned experimental metrics

### Decision

Rejected.

OmniFleet intentionally captures richer vehicle observations.

---

## Alternative C — Store Calculated Consumption Directly

Example:

```text
refuel.consumption = 6.2
```

as the authoritative value.

### Advantages

* fast reporting
* simple dashboard queries

### Disadvantages

* algorithms cannot evolve safely
* historical values become difficult to reproduce
* corrections may leave stale values
* calculation provenance becomes unclear

### Decision

Rejected as the primary source of truth.

Raw observations remain authoritative.

Derived data may later be cached for performance if it can be reproduced.

---

# Invariants

## Refuel

A RefuelEvent:

* belongs to exactly one compatible Vehicle
* has a positive fuel quantity
* identifies the fuel type
* identifies the event time
* records odometer
* identifies whether the fill was full or partial
* identifies currency when monetary data exists

---

## Fuel Compatibility

The recorded fuel type must be supported by the Vehicle.

---

## Charge

A ChargeEvent:

* belongs to exactly one compatible Vehicle
* identifies a charging time
* records odometer for the initial analytics workflow
* contains valid SoC values when supplied
* contains positive delivered energy when supplied
* identifies currency when monetary data exists

---

## Time

When both charge timestamps exist:

```text
ended_at >= started_at
```

---

## SoC

When supplied:

```text
0 <= start_soc_pct <= 100
0 <= end_soc_pct <= 100
```

For normal charging:

```text
end_soc_pct >= start_soc_pct
```

---

## Raw Observation Preservation

A derived value must never become the only retained representation of the original event data.

---

# Initial MVP Scope

The first usable OmniFleet vertical slice focuses on ICE refueling.

Therefore the first implemented event model only needs:

```text
RefuelEvent
```

with approximately:

```text
vehicle
recorded_by
occurred_at
odometer
fuel_type
liters
unit_price / total_amount
currency
is_full_tank
range_before
range_after
onboard_avg_consumption
notes
```

ChargeEvent is designed during Stage 0 but can be implemented later.

This allows the first product slice to stay small without making later EV/PHEV support require a domain redesign.

---

# Validation Scenarios

## Scenario 1 — Valid Full Refuel

```text
Given:
ICE Diesel vehicle

When:
odometer = 105000 km
fuel = Diesel
liters = 42.5
is_full_tank = true
currency = EUR
valid cost data exists

Then:
RefuelEvent is accepted
```

---

## Scenario 2 — Partial Refuel

```text
Given:
ICE vehicle

When:
20 liters are added
is_full_tank = false

Then:
RefuelEvent is valid
But:
The 20 liters must not automatically be treated
as fuel consumed since the previous event
```

---

## Scenario 3 — Invalid Quantity

```text
liters = 0

→ REJECT
```

---

## Scenario 4 — Unsupported Fuel

```text
Vehicle supports Diesel

Refuel fuel_type = Petrol

→ REJECT
```

---

## Scenario 5 — Range Data Missing

```text
range_before = unknown
range_after = unknown

→ RefuelEvent remains valid
```

Range-based experimental analytics are simply unavailable.

---

## Scenario 6 — Cost Rounding

```text
liters = 42.17
unit_price = 1.579
total_amount = 66.58

Small arithmetic rounding difference exists.

→ Event remains valid
```

The provided total is preserved.

---

## Scenario 7 — EV Charging

```text
Given:
EV

When:
energy_kwh = 38.2
start_soc = 20
end_soc = 82
currency = EUR
valid transaction cost exists

Then:
ChargeEvent is accepted
```

---

## Scenario 8 — Invalid EV Refuel

```text
Vehicle = EV

Create RefuelEvent

→ REJECT
```

---

## Scenario 9 — Historical Entry

```text
Today = 2026-09-14

User enters:
occurred_at = 2026-09-10

→ ACCEPT
```

`recorded_at` and `occurred_at` remain different.

---

## Scenario 10 — Event Correction

```text
Existing event:
odometer = 105000

Correct value:
odometer = 105100

When corrected:
dependent calculations must be considered stale
and recalculated
```

---

# Deferred Decisions

This ADR deliberately leaves the following unresolved:

* exact SQL schema
* decimal precision
* money/value types
* exact source tracking for derived monetary values
* correction history
* soft-delete strategy
* event versioning
* odometer-reset workflow
* charging-loss modeling
* ChargeEvent canonical calendar time (`started_at` vs `ended_at`)
* detailed charger standards
* pricing breakdowns for charging
* location representation
* attachment linkage
* OCR data provenance
* import workflows

These can be decided without changing the fundamental event boundaries.

---

# Consequences

## Positive

OmniFleet keeps the real-world observations that later analytics depend on.

ICE and electric charging remain cleanly separated.

PHEVs can naturally generate both event types.

Calculation algorithms can evolve without losing the original measurements.

Partial refuels remain valid records without pretending they provide verified consumption values.

---

## Negative

The event models contain more fields than a minimal fuel-log application.

Some optional observations require additional user input.

Historical event editing will require careful recalculation behavior.

The model also requires explicit validation around event compatibility and measurement quality.

These costs are accepted because analytics are a core product goal.

---

# Result

OmniFleet will use separate:

```text
RefuelEvent
ChargeEvent
```

domain entities.

Both preserve real-world observations as the source of truth.

RefuelEvents capture fuel quantity, odometer, fuel type, transaction information, full/partial state and optional onboard range/consumption observations.

ChargeEvents capture electrical energy, odometer, SoC, charging time and transaction information.

Calculated metrics are derived from these events rather than treated as authoritative input.

This decision is accepted as part of the Stage 0 foundation.

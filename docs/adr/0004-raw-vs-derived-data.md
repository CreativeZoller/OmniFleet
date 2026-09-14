# ADR-0004: Raw vs. Derived Data

* **Status:** Accepted
* **Date:** 2026-09-14
* **Stage:** Stage 0 — Domain & Engineering Foundation

## Context

OmniFleet is not only a vehicle log.

A significant part of its value comes from calculating information from recorded vehicle events.

Examples include:

```text
distance traveled
fuel consumption
electric consumption
cost per kilometer
weighted unit price
monthly spending
Driver vs. Car efficiency
```

Many of these values depend on:

* multiple events
* historical ordering
* calculation assumptions
* measurement quality
* algorithms that may evolve over time

If calculated results are stored as if they were original facts, several problems appear.

For example:

```text
Refuel A
odometer = 100000 km

Refuel B
odometer = 100500 km
liters = 32.4
```

may produce:

```text
consumption = 6.48 L/100km
```

But if Refuel A is later corrected to:

```text
99950 km
```

then the previous consumption result is no longer valid.

Likewise, an experimental algorithm may change after real-world validation.

OmniFleet therefore needs a clear distinction between:

1. what actually happened or was observed
2. what OmniFleet calculated from those observations

---

# Decision

OmniFleet will treat **raw observations as the primary source of truth**.

Calculated or aggregated values are considered **derived data**.

Conceptually:

```text
Raw Observations
      ↓
Calculation Logic
      ↓
Derived Data
```

Derived values must remain reproducible from the underlying raw observations whenever practical.

A derived value must not replace or destroy the source observations used to calculate it.

---

# Raw Data

Raw data represents facts, measurements, or observations recorded about a real-world event.

Examples include:

```text
odometer
liters
fuel type
total amount
currency
full / partial refuel
range before
range after
vehicle-reported consumption
energy kWh
start SoC
end SoC
timestamps
```

These values describe what was:

* observed
* entered
* imported
* confirmed

at the time of an event.

---

# Raw Does Not Mean Perfect

A raw observation is not automatically guaranteed to be physically exact.

For example:

```text
range_after = 720 km
```

means:

> the vehicle displayed approximately 720 km of remaining range

It does not mean:

> the vehicle physically contained exactly enough energy to travel 720 km

Likewise:

```text
start_soc = 35%
```

is a vehicle-reported State of Charge observation.

It remains raw input even though the underlying measurement itself may be approximate.

The important distinction is:

```text
Raw
=
the original observed value
```

not:

```text
Raw
=
scientifically exact truth
```

---

# Derived Data

Derived data is produced from one or more raw observations.

Examples:

```text
distance traveled
L/100km
kWh/100km
cost/km
weighted average price
monthly cost
charging duration
efficiency ratio
```

Derived values may depend on:

```text
one event
multiple events
time ranges
vehicle configuration
calculation version
measurement quality rules
```

---

# Example: Distance

Consider:

```text
Refuel A
odometer = 100000

Refuel B
odometer = 100500
```

The following is raw:

```text
100000
100500
```

The following is derived:

```text
distance = 500 km
```

The distance does not need to become the primary stored truth because it can be reproduced from the odometer readings.

---

# Example: Verified Consumption

Given:

```text
Previous full refuel:
100000 km

Current full refuel:
100500 km

Current fuel quantity:
32.4 L
```

Derived:

```text
distance = 500 km

consumption =
32.4 / 500 × 100

= 6.48 L/100km
```

The authoritative inputs remain:

```text
100000 km
100500 km
32.4 L
full-tank state
```

not:

```text
6.48 L/100km
```

---

# Example: Weighted Unit Price

Given:

```text
Refuel A:
20 L
30 EUR

Refuel B:
50 L
80 EUR
```

A simple arithmetic average of observed unit prices:

```text
(1.50 + 1.60) / 2
= 1.55 EUR/L
```

is not the correct fleet-level unit price.

The derived weighted value uses confirmed transaction totals:

```text
sum(total_amount) / sum(quantity)
```

```text
110 EUR / 70 L
≈ 1.5714 EUR/L
```

Using `quantity × unit_price` is only equivalent when those products match the confirmed totals.

This calculation is derived from the original transactions.

If one event changes, the aggregate should change automatically.

---

# Source of Truth Principle

OmniFleet follows this hierarchy:

```text
Original observation
        ↓
Validated event
        ↓
Derived calculation
        ↓
Aggregate / report
```

The further down the chain a value is, the less suitable it is as an independent source of truth.

For example:

```text
monthly average consumption
```

should never become an authoritative replacement for:

```text
individual refuel events
```

---

# User-Entered vs System-Derived

Some fields may be either entered by the user or calculated by OmniFleet.

Example:

```text
liters
unit_price
total_amount
```

A user may enter:

```text
liters = 40
unit_price = 1.60
```

and OmniFleet may calculate:

```text
total_amount = 64.00
```

Alternatively, a user may enter:

```text
liters = 40
total_amount = 64.00
```

and OmniFleet may calculate:

```text
effective unit price = 1.60
```

This creates a provenance question.

The domain must conceptually distinguish:

```text
observed / entered value
```

from:

```text
derived value
```

when the distinction affects behavior or trust.

The exact persistence representation of provenance is deferred.

---

# Confirmed Data

Future features such as OCR may introduce another category:

```text
suggested
```

For example:

```text
OCR reads receipt

liters = 41.23
total = 67.10
```

These values must not automatically become authoritative raw observations.

The intended lifecycle is:

```text
Extracted
   ↓
Suggested
   ↓
User confirms
   ↓
Recorded observation
```

Only after confirmation should the value become part of the trusted event dataset.

---

# Imported Data

Future integrations may import data from:

* vehicle APIs
* charging networks
* fuel providers
* CSV files
* external applications

Imported values are still raw observations if they represent original external measurements.

However, their origin should remain identifiable.

Conceptually:

```text
source = manual
source = OCR-confirmed
source = import
source = provider API
```

Detailed source/provenance modeling is deferred.

---

# Derived Data Should Be Recomputable

Where practical, derived values should be reproducible from:

```text
raw observations
+
domain rules
+
calculation version
```

Conceptually:

```text
Result =
calculate(
    observations,
    rules,
    version
)
```

This provides several benefits:

* algorithms can improve
* historical corrections can propagate
* bugs can be fixed
* tests can reproduce output
* analytics remain explainable

---

# Derived Data May Be Cached

This ADR does not prohibit storing calculated values.

For performance or reporting purposes, OmniFleet may later persist:

```text
cached calculations
materialized views
summary tables
precomputed analytics
```

However, persisted derived data must be treated as:

```text
rebuildable
```

rather than authoritative.

Conceptually:

```text
raw data
= source of truth

cached calculation
= optimization
```

If the cache disagrees with reproducible calculations, the cache is wrong.

---

# Calculation Dependencies

Derived metrics may depend on more than the event they are displayed beside.

Example:

```text
Refuel A
Refuel B
```

The consumption displayed for segment:

```text
A → B
```

depends on both events.

Therefore changing:

```text
Refuel A
```

may invalidate calculations associated with:

```text
Refuel B
```

and possibly later events.

OmniFleet must therefore avoid assuming:

```text
one event edit
=
one isolated calculation change
```

Historical edits can have downstream effects.

---

# Invalidating Derived Data

Conceptually, whenever a source observation changes:

```text
Raw Event Updated
       ↓
Dependent Derived Data Invalid
       ↓
Recalculate
```

Examples of source changes include:

```text
odometer corrected
liters corrected
full-tank flag corrected
timestamp corrected
fuel type corrected
```

The exact technical invalidation mechanism is deferred.

Possible implementations may include:

```text
recalculate on read
background recalculation
dirty markers
versioned calculations
database views
```

This ADR defines the behavior, not the implementation.

---

# Calculation Versioning

Some calculation methods may evolve.

This is especially relevant for:

```text
estimated consumption
experimental Driver vs. Car metrics
EV efficiency calculations
PHEV combined analytics
```

Therefore OmniFleet should be able to explain:

```text
which calculation logic produced a value
```

if calculated values are persisted or externally exposed.

Conceptually:

```text
calculation_method
calculation_version
```

may become necessary.

Exact versioning mechanics are deferred to the calculation architecture.

---

# Verified, Estimated and Experimental

Measurement quality applies to derived metrics.

A calculated value should not only communicate:

```text
value = 6.4
```

but where relevant also:

```text
quality = verified
```

or:

```text
quality = estimated
```

or:

```text
quality = experimental
```

This metadata is part of the semantic meaning of the result.

Example:

```text
6.4 L/100km
verified
```

and:

```text
6.4 L/100km
estimated
```

are not equivalent claims.

---

# Presentation Must Not Hide Quality

The frontend must not present derived values in a way that removes their uncertainty.

For example, if a dashboard contains:

```text
Consumption
6.3 L/100km
```

but that number is experimental, the user should have some way to understand that.

Possible UI representations may later include:

```text
Verified
Estimated
Experimental

icons
badges
tooltips
different chart styles
```

The exact UX is deferred.

The semantic distinction is not.

---

# Raw Corrections

Raw observations may be corrected.

Example:

```text
original odometer:
105000

corrected odometer:
105100
```

The corrected value becomes the current valid observation.

However, corrections may need historical traceability.

Future audit functionality may preserve:

```text
previous value
new value
who changed it
when
reason
```

This ADR does not mandate full event sourcing or immutable records.

It only requires that corrections trigger correct downstream behavior.

---

# Event Sourcing

Full event sourcing was considered but is not required.

OmniFleet does not need to model every application state change as an immutable domain event merely to preserve calculation correctness.

The initial architecture should remain simpler.

A conventional relational model with:

```text
current records
+
audit metadata where needed
+
recalculable analytics
```

is sufficient.

---

# Database Normalization

Derived values should not be duplicated in core event tables without a clear reason.

For example, this model is discouraged:

```text
refuels
- odometer
- liters
- consumption_l100
- distance_since_previous
- cost_per_km
```

when:

```text
consumption_l100
distance_since_previous
cost_per_km
```

can be reliably derived.

Persisting such fields without invalidation logic creates stale-data risks.

---

# Reporting Layer

Reporting may combine multiple raw event sources.

For example:

```text
RefuelEvent
      \
       → Cost Report
      /
ChargeEvent
```

The report itself is derived data.

It may be generated dynamically or from cached/reporting structures.

It must never redefine or mutate the original vehicle events.

---

# Aggregation

Aggregates include:

```text
monthly spending
yearly spending
average consumption
average unit price
distance by month
cost per vehicle
```

These values depend on a selected scope.

For example:

```text
Vehicle
Household
Currency
Date Range
Energy Type
```

The scope must be part of the meaning of an aggregate.

A number such as:

```text
Average Price = 1.61
```

is incomplete without knowing:

```text
which currency?
which fuel?
which vehicle?
which period?
```

---

# Multi-Currency Derived Data

Raw monetary values remain in their original currencies.

Example:

```text
50 EUR
40000 HUF
```

Without an explicit exchange-rate model, OmniFleet must not derive:

```text
combined total = 150 EUR
```

or any equivalent normalized amount.

Initial aggregates therefore remain grouped by currency.

Future normalized reports may derive converted values from:

```text
original amount
+
exchange-rate observation
```

The original transaction remains unchanged.

---

# Deleting Raw Events

Deleting a raw event may invalidate derived data.

Example:

```text
Full Refuel A
Partial Refuel B
Full Refuel C
```

If:

```text
Refuel B
```

is deleted, calculations across the segment may change.

Therefore event deletion must trigger the same dependency considerations as editing.

This is another reason calculated values must not be treated as isolated stored facts.

---

# Alternatives Considered

## Alternative A — Store All Calculated Values

Example:

```text
refuel
- liters
- odometer
- consumption
- distance
- cost_per_km
```

### Advantages

* easy reporting
* fast reads
* simple frontend

### Disadvantages

* stale data after edits
* duplicated truth
* difficult algorithm changes
* hard-to-explain discrepancies
* complex update chains

### Decision

Rejected as the primary model.

Calculated values may only be persisted as reproducible caches or explicitly versioned outputs.

---

## Alternative B — Never Store Derived Data

### Advantages

* no stale calculations
* single source of truth
* simple semantics

### Disadvantages

* expensive analytics may be repeatedly recalculated
* complex dashboards may become inefficient
* large future datasets may benefit from precomputation

### Decision

Rejected as an absolute rule.

OmniFleet may cache derived values where useful, but caches remain rebuildable.

---

## Alternative C — Raw Observations + Reproducible Derived Data

### Advantages

* preserves original facts
* allows algorithm evolution
* supports correction workflows
* enables reproducible analytics
* supports caching without changing authority
* keeps uncertainty explicit

### Disadvantages

* calculation dependencies need management
* some reads become more complex
* editing historical events can require recalculation

### Decision

Accepted.

---

# Invariants

## Raw Preservation

Original confirmed observations must not be discarded merely because a derived result exists.

---

## Derived Reproducibility

Where practical, derived values must be reproducible from stored observations and documented calculation rules.

---

## Cache Authority

Cached derived values are not authoritative.

---

## Correction Propagation

Changing a source observation must invalidate or update dependent calculations.

---

## Measurement Quality

Derived metrics that differ in reliability must preserve their quality classification.

---

## Currency Integrity

Cross-currency derived totals require explicit exchange-rate information.

---

## Provenance

Automatically suggested data must not silently become confirmed raw input.

---

# Initial MVP Behavior

The first ICE-focused version may calculate values at request time.

For example:

```text
GET refuel history
        ↓
Load events
        ↓
Calculate segments
        ↓
Return analytics
```

This is acceptable because the initial dataset will be small.

No dedicated analytics cache is required for the first MVP.

If later performance requires it, derived data may be cached without changing the source-of-truth model.

---

# Validation Scenarios

## Scenario 1 — Recalculate After Odometer Correction

```text
Refuel A:
100000 km

Refuel B:
100500 km
32 L

Derived:
distance = 500 km
consumption = 6.4 L/100km
```

User corrects:

```text
Refuel A:
99900 km
```

Then:

```text
distance = 600 km
consumption ≈ 5.33 L/100km
```

The old result must not remain authoritative.

---

## Scenario 2 — Experimental Formula Changes

```text
Driver vs. Car Algorithm v1
→ ratio = X
```

A later validated formula changes the calculation.

Then historical raw:

```text
range before
range after
odometer
```

remains unchanged.

The result may be recalculated with the new methodology.

---

## Scenario 3 — OCR Suggestion

```text
OCR suggests:
liters = 42.5
```

Before confirmation:

```text
not authoritative
```

After user confirms:

```text
recorded observation
```

---

## Scenario 4 — Cached Result Mismatch

```text
Raw events calculate:
6.4 L/100km

Cached value:
6.7 L/100km
```

Then:

```text
cache is stale
```

The raw-derived calculation wins.

---

## Scenario 5 — Currency Separation

```text
Event A = 60 EUR
Event B = 20000 HUF
```

Without exchange-rate data:

```text
combined normalized total
→ unavailable
```

Both original amounts remain reportable separately.

---

## Scenario 6 — Historical Event Deletion

```text
A → B → C
```

Delete:

```text
B
```

Then:

```text
dependent segment analytics must be recalculated
```

Existing stale results must not remain authoritative.

---

# Deferred Decisions

This ADR does not define:

* exact calculation engine architecture
* cache implementation
* dirty-state tracking
* calculation persistence
* algorithm-version storage
* audit-log format
* OCR provenance schema
* import provenance schema
* event correction history
* materialized views
* background processing

These are implementation decisions that can be made later while preserving this source-of-truth model.

---

# Consequences

## Positive

The historical dataset remains usable even when calculation logic evolves.

Incorrect observations can be corrected without permanently corrupting analytics.

Experimental calculations can be changed without rewriting the original vehicle history.

The system can explain how values were produced.

---

## Negative

Derived values may require recalculation.

Historical edits can affect more than one displayed result.

Some reporting queries may become more complex.

If future performance optimization introduces caches, cache invalidation must be handled correctly.

These costs are accepted because trustworthy and evolvable analytics are a core OmniFleet requirement.

---

# Result

OmniFleet treats confirmed real-world observations as its primary source of truth.

Calculated values are derived from those observations.

Derived values may be recalculated, versioned, cached or replaced as algorithms evolve.

They must not destroy or silently redefine their source observations.

Measurement quality and calculation provenance remain part of the meaning of analytical results.

This decision is accepted as part of the Stage 0 foundation.

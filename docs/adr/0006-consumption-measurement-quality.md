# ADR-0006: Consumption Measurement Quality

* **Status:** Accepted
* **Date:** 2026-09-14
* **Stage:** Stage 0 — Domain & Engineering Foundation

## Context

OmniFleet calculates multiple forms of vehicle consumption and efficiency from recorded RefuelEvents and ChargeEvents.

However, not every calculated value is supported by the same quality of evidence.

For example:

```text
Full tank
→ drive
→ full tank
```

can provide a reliable ICE consumption measurement.

But:

```text
Partial refuel
→ drive
→ partial refuel
```

does not reveal exactly how much fuel was consumed between the two observations.

Similarly, vehicle-provided range estimates can be useful for comparing driving behavior, but they are influenced by internal manufacturer algorithms and changing driving conditions.

If OmniFleet displays all of these values simply as:

```text
6.3 L/100km
```

the user cannot tell how that number was obtained or how much confidence should be placed in it.

The system therefore needs an explicit quality model for calculated metrics.

---

# Decision

OmniFleet will classify consumption-related calculated results using three initial quality categories:

```text
Verified
Estimated
Experimental
```

The classification describes the **method used to produce the result**, not whether the number happens to be numerically close to reality.

Conceptually:

```text
Calculated Metric
├── value
├── unit
├── quality
└── calculation method
```

---

# Verified

A `Verified` metric is based on a measurement method with sufficient direct observations to calculate the result reliably under the assumptions of that method.

For ICE vehicles, the primary example is full-to-full consumption.

Example:

```text
Refuel A
odometer = 100000
full tank = true

Refuel B
odometer = 100500
full tank = true
fuel added = 32.4 L
```

Then:

```text
distance = 500 km

consumption =
32.4 / 500 × 100

= 6.48 L/100km
```

The result may be classified as:

```text
6.48 L/100km
quality = Verified
```

because the amount added at the second full refill approximately represents the fuel consumed since the previous full reference point.

---

# Verified Does Not Mean Laboratory Exact

`Verified` does not mean scientifically perfect or error-free.

Real-world measurements may still contain:

* pump measurement tolerances
* odometer inaccuracies
* differences in filling level
* temperature-related fuel-volume variation
* user input mistakes

The meaning is:

> OmniFleet has enough direct observations to apply a recognized measurement method without relying on additional estimation.

---

# Full-to-Full Segments

A verified ICE consumption segment requires compatible full-refuel boundaries.

Conceptually:

```text
Full Refuel A
        ↓
driving / optional partial refuels
        ↓
Full Refuel B
```

The segment may contain partial refuels between the two full measurements.

Example:

```text
Full A
Partial B
Partial C
Full D
```

The verified consumption for:

```text
A → D
```

must account for all fuel added within the segment:

```text
fuel consumed approximately =
fuel added at B
+
fuel added at C
+
fuel added at D
```

provided A and D represent compatible full reference points.

This is important because verified consumption must not require every individual refuel to be full.

---

# Example: Full → Partial → Full

Given:

```text
Full A
odometer = 100000

Partial B
odometer = 100250
fuel = 15 L

Full C
odometer = 100600
fuel = 23 L
```

Then:

```text
distance = 600 km

fuel used ≈
15 + 23
= 38 L
```

Derived:

```text
38 / 600 × 100
≈ 6.33 L/100km
```

This segment may still be classified as:

```text
Verified
```

because it is bounded by full reference points.

The partial event itself does not independently produce a verified segment.

---

# Estimated

An `Estimated` metric is derived from incomplete or indirect information but is still considered useful.

Examples may include:

* consumption between non-full refuels
* consumption inferred from partial information
* calculations using known approximations
* EV efficiency when observations do not cleanly represent the full energy flow

Estimated values must be clearly distinguishable from verified measurements.

Example:

```text
6.2 L/100km
quality = Estimated
```

must not be presented as equivalent to:

```text
6.2 L/100km
quality = Verified
```

---

# Partial Refuels

A partial refuel is a valid transaction but does not independently reveal consumption.

Example:

```text
Previous event:
unknown remaining fuel

Current event:
30 L added

Current tank:
not full
```

The system does not know:

```text
fuel before event
fuel after event
```

therefore:

```text
30 L
```

cannot automatically be interpreted as:

```text
30 L consumed since previous event
```

Any consumption result produced from such a segment requires additional assumptions and must not be classified as Verified.

---

# Estimated Consumption Methodology

ADR-0006 defines the quality classification but does not mandate one universal estimation algorithm.

Estimated calculations may evolve as the application is validated with real-world data.

Any implemented estimation method must document:

* required inputs
* assumptions
* limitations
* calculation version
* situations where the result is unavailable

The calculation methodology itself belongs in `docs/calculations.md`.

---

# Experimental

An `Experimental` metric is based on a method OmniFleet is actively evaluating.

The initial example is the range-based:

```text
Driver vs. Car
```

efficiency metric.

Vehicle range predictions are not direct measurements of physical fuel or battery energy.

They can change based on:

* recent consumption
* driving style
* temperature
* terrain
* traffic
* HVAC use
* manufacturer-specific algorithms

Therefore metrics derived primarily from changing range estimates remain experimental until sufficient real-world validation exists.

---

# Range-Based Efficiency

Conceptually, OmniFleet may compare:

```text
Range After Previous Refuel
-
Range Before Current Refuel
```

with:

```text
Actual Distance Traveled
```

to derive a ratio:

```text
Efficiency Ratio =
Predicted Range Consumed
/
Actual Distance Traveled
```

Interpretation:

```text
≈ 1.00
vehicle prediction roughly matched reality

< 1.00
vehicle underestimated how far the available fuel/energy would go

> 1.00
vehicle overestimated how far the available fuel/energy would go
```

This result is classified as:

```text
Experimental
```

rather than Verified.

---

# Experimental Does Not Mean Useless

Experimental metrics may still be valuable.

They may reveal:

* trends
* changes in driving behavior
* seasonal differences
* differences between predicted and real-world vehicle behavior

The classification simply communicates that the calculation is not treated as an established direct consumption measurement.

---

# Quality Belongs to the Result

Quality classification belongs to a calculated metric, not globally to an event.

For example, the same RefuelEvent may participate in:

```text
Verified full-to-full consumption

Experimental range-based efficiency

Derived cost/km
```

Each output may have different semantics.

Therefore OmniFleet must not assign a single quality flag such as:

```text
refuel.quality
```

and assume it applies to every calculation.

---

# Method and Quality

A calculation result should conceptually contain enough information to explain itself.

Example:

```text
Metric:
ICE Consumption

Value:
6.48

Unit:
L/100km

Quality:
Verified

Method:
full-to-full
```

Another result may be:

```text
Metric:
Driver vs. Car Efficiency

Value:
0.87

Unit:
ratio

Quality:
Experimental

Method:
range-delta-v1
```

This makes analytical output explainable.

---

# Calculation Availability

OmniFleet should prefer:

```text
metric unavailable
```

over generating a misleading number.

Example:

```text
Partial Refuel A
Partial Refuel B
No usable range observations
```

Then:

```text
Verified Consumption:
Unavailable
```

rather than inventing a result from insufficient information.

Missing analytics is preferable to false precision.

---

# Unknown Is Different From Zero

A missing metric must never be represented as zero.

Example:

```text
consumption unavailable
```

must not become:

```text
0.0 L/100km
```

because zero has a real mathematical meaning.

Conceptually:

```text
None / unavailable
```

and:

```text
0
```

are different states.

---

# EV Consumption

EV consumption requires similar quality awareness.

A common calculation is:

```text
kWh/100km =
energy / distance × 100
```

However, the meaning depends on the energy observation.

For example:

```text
charger delivered energy
```

may include charging losses that are not stored in the battery.

Meanwhile:

```text
SoC change × nominal battery capacity
```

is itself an estimate.

Therefore EV metrics must document which energy source was used.

---

# EV Verified Classification

Stage 0 deliberately avoids declaring every charger-based:

```text
kWh / distance
```

calculation automatically Verified.

The exact quality rules for EV consumption require the later calculation specification to distinguish:

* charger-delivered energy
* battery energy
* charging loss
* charging-session boundaries
* driving distance boundaries

Until that methodology is explicitly defined, EV calculation quality should be conservative.

---

# PHEV Measurement Quality

PHEVs introduce two independent energy streams:

```text
fuel
electricity
```

Therefore:

```text
L/100km
```

alone may not describe total energy usage.

Likewise:

```text
kWh/100km
```

alone may represent only part of the vehicle's propulsion energy.

OmniFleet should therefore avoid presenting either value as a universal PHEV efficiency metric without context.

Possible future PHEV metrics include:

```text
fuel consumption
electric consumption
fuel cost/km
electric cost/km
combined energy cost/km
electric driving share
```

Each metric will require its own quality and calculation rules.

---

# Cost Metrics

Measurement quality is mainly intended for physical consumption and efficiency calculations.

Financial calculations based directly on confirmed monetary observations may not require the same:

```text
Verified / Estimated / Experimental
```

classification.

For example:

```text
total spent this month
```

derived from confirmed transactions is straightforward.

However, a cost metric relying on estimated distance or estimated consumption may still require quality metadata.

Quality therefore attaches to the calculation where uncertainty exists rather than being forced onto every metric.

---

# Quality Must Survive Aggregation

If a dashboard aggregates measurements of different quality levels, it must not silently hide that distinction.

Example:

```text
3 verified measurements
2 estimated measurements
```

should not automatically become one unlabeled:

```text
average consumption
```

without defined aggregation rules.

Possible future strategies include:

* verified-only averages
* separate series
* explicit mixed-quality averages
* confidence indicators

The exact UI and aggregation policy is deferred.

---

# Default Reporting Preference

Where both verified and lower-quality measurements exist, OmniFleet should prefer verified values for primary consumption reporting.

Conceptually:

```text
Primary consumption trend
→ Verified measurements when available
```

Estimated and experimental data may be displayed as additional information.

This avoids allowing lower-confidence calculations to visually dominate stronger measurements.

---

# Historical Reclassification

Quality classification may change if the calculation methodology changes.

Example:

An experimental method may later become sufficiently validated to be treated as an estimated production metric.

This does not require rewriting raw observations.

Instead:

```text
raw data
+
calculation rules v2
```

may produce a differently classified derived result.

This follows ADR-0004.

---

# Quality Is Not User-Editable

Users should not manually choose:

```text
Verified
Estimated
Experimental
```

for calculated results.

The quality classification is determined by:

* available observations
* calculation method
* domain rules

For example:

```text
full-to-full inputs available
→ Verified
```

rather than:

```text
user selects "Verified"
```

This prevents semantic inconsistency.

---

# Invalid Measurements

A segment may fail validation entirely.

Examples:

```text
distance <= 0
fuel quantity impossible for selected calculation
missing required boundary
event ordering invalid
```

Such cases should produce:

```text
Invalid / unavailable result
```

rather than forcing them into one of the three quality categories.

Therefore conceptually:

```text
Calculation Result
or
No Valid Result
```

comes before quality classification.

---

# UI Representation

The frontend should eventually communicate quality without overwhelming users.

Possible representations include:

```text
Verified
Estimated
Experimental
```

badges or tooltip explanations.

Charts may also distinguish series by quality.

The exact visual design is deferred.

The important rule is that uncertainty must not be hidden.

---

# API Representation

If analytical metrics are exposed through an API, quality should be machine-readable.

Conceptually:

```json
{
  "value": "6.48",
  "unit": "L/100km",
  "quality": "verified",
  "method": "full-to-full"
}
```

This is illustrative rather than a finalized API contract.

---

# Alternatives Considered

## Alternative A — No Quality Classification

All calculations are returned simply as values.

### Advantages

* simplest implementation
* simplest UI

### Disadvantages

* misleading comparisons
* hides assumptions
* experimental metrics appear authoritative
* impossible for consumers to distinguish calculation quality

### Decision

Rejected.

---

## Alternative B — Accurate / Inaccurate Boolean

Example:

```text
is_accurate = true / false
```

### Advantages

* simple

### Disadvantages

* overly binary
* "inaccurate" is ambiguous
* estimated and experimental methods are meaningfully different
* real-world verified measurements still contain tolerances

### Decision

Rejected.

---

## Alternative C — Numeric Confidence Score

Example:

```text
confidence = 0.82
```

### Advantages

* granular
* potentially useful for advanced analytics

### Disadvantages

* implies mathematical precision OmniFleet cannot currently justify
* difficult to derive objectively
* difficult to explain to users

### Decision

Rejected for the initial product.

---

## Alternative D — Semantic Quality Levels

```text
Verified
Estimated
Experimental
```

### Advantages

* understandable
* domain-oriented
* communicates method quality without fake precision
* extensible
* works naturally with calculation metadata

### Disadvantages

* each calculation method needs an explicit classification
* aggregation requires additional rules

### Decision

Accepted.

---

# Invariants

## Quality Is Derived

Measurement quality is determined by calculation rules, not manually entered by the user.

---

## Verified Requires Sufficient Evidence

A result cannot be classified as Verified unless its documented method requirements are satisfied.

---

## Experimental Is Explicit

Experimental calculations must never be presented without their experimental classification.

---

## Missing Is Not Zero

Unavailable calculations are represented as unavailable rather than numeric zero.

---

## Method Is Explainable

A quality-classified result should be traceable to a documented calculation method.

---

## Raw Data Is Unchanged

Changing quality rules or calculation methods never alters the original observations.

---

# Initial ICE Rules

For the first ICE-focused MVP:

## Full-to-Full Consumption

```text
Quality:
Verified
```

when valid full-refuel boundaries exist and required observations are available.

---

## Partial-Only Consumption

```text
Quality:
Estimated
```

only if an explicitly documented estimation method can produce a valid result.

Otherwise:

```text
Unavailable
```

---

## Driver vs. Car

```text
Quality:
Experimental
```

for the initial implementation.

---

## Vehicle-Reported Consumption

The onboard vehicle value is not an OmniFleet consumption calculation.

It should be labeled as something such as:

```text
Vehicle-reported
```

rather than:

```text
Verified
```

because it represents an external observation.

---

# Validation Scenarios

## Scenario 1 — Full to Full

```text
Full Refuel A
100000 km

Full Refuel B
100500 km
32.4 L
```

Result:

```text
6.48 L/100km
Verified
```

---

## Scenario 2 — Full, Partial, Full

```text
Full A
100000 km

Partial B
100250 km
15 L

Full C
100600 km
23 L
```

Result for A → C:

```text
38 L / 600 km × 100
≈ 6.33 L/100km

Verified
```

---

## Scenario 3 — Partial to Partial

```text
Partial A
Partial B
```

with no sufficient additional measurement.

Result:

```text
Verified consumption unavailable
```

No fake full-to-full result is produced.

---

## Scenario 4 — Range-Based Driver Comparison

Valid range and odometer observations exist.

Result:

```text
Efficiency Ratio = 0.87
Experimental
```

---

## Scenario 5 — Missing Range

Range-based inputs are incomplete.

Result:

```text
Driver vs. Car
Unavailable
```

not:

```text
0
```

---

## Scenario 6 — Onboard Consumption

Vehicle display reports:

```text
6.1 L/100km
```

OmniFleet records:

```text
Vehicle-reported consumption:
6.1 L/100km
```

It does not label the value Verified.

---

## Scenario 7 — Corrected Historical Event

A source odometer value is corrected.

Then:

```text
existing calculated result
→ invalidated
→ recalculated
→ quality re-evaluated
```

---

# Deferred Decisions

This ADR deliberately does not define:

* exact estimated-consumption algorithm
* final Driver vs. Car formula
* EV verified-consumption rules
* PHEV combined efficiency methodology
* dashboard quality visualization
* mixed-quality aggregation rules
* quality enum storage
* calculation engine architecture
* confidence scores
* long-term validation thresholds for experimental metrics

These belong to later calculation and implementation decisions.

---

# Consequences

## Positive

OmniFleet avoids false precision.

Users can distinguish strong measurements from estimates and experiments.

New algorithms can be introduced without pretending they have the same validity as established measurement methods.

Dashboard and API consumers receive richer semantic information.

---

## Negative

Analytics become somewhat more complex.

The frontend must explain measurement quality.

Aggregations across mixed-quality values require explicit rules.

Some situations will correctly produce no consumption result at all.

These costs are accepted because trustworthy analytics are more important than always displaying a number.

---

# Result

OmniFleet classifies calculated consumption and efficiency metrics as:

```text
Verified
Estimated
Experimental
```

when applicable.

Verified results require sufficient observations for a documented direct measurement method.

Estimated values use incomplete or indirect information and must remain visibly distinct.

Experimental values represent methods still being validated against real-world data.

Unavailable data is preferable to misleading precision.

The initial ICE full-to-full method is classified as Verified, while the range-based Driver vs. Car metric remains Experimental.

This decision is accepted as part of the Stage 0 foundation.

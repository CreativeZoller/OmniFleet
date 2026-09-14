# OmniFleet Calculation Methodology

* **Status:** Stage 0 — Accepted
* **Date:** 2026-09-14

## Purpose

This document defines the initial calculation methodology used by OmniFleet.

It translates the domain and architectural decisions from the Stage 0 ADRs into explicit mathematical rules.

The goals are:

* calculations are reproducible
* raw observations remain authoritative
* calculation quality is explicit
* missing information does not produce fake precision
* formulas can later become executable tests
* experimental methods remain distinguishable from established measurements

This document defines **calculation behavior**, not database implementation or UI presentation.

---

# 1. General Principles

## 1.1 Raw observations are authoritative

Calculations operate on confirmed source observations such as:

```text
odometer
liters
energy_kwh
total_amount
unit_price
currency
is_full_tank
range_before
range_after
State of Charge
timestamps
```

Calculated results never replace these source observations.

---

## 1.2 Missing is not zero

If insufficient information exists to calculate a metric, the result is:

```text
Unavailable
```

not:

```text
0
```

Zero is a valid numeric result and therefore cannot represent missing information.

---

## 1.3 Invalid is different from unavailable

Two different failure states exist conceptually.

### Unavailable

The data is valid, but there is not enough information to calculate the metric.

Example:

```text
Partial refuel
No compatible full-tank boundary
```

### Invalid

The observations violate the requirements of the calculation.

Example:

```text
distance <= 0
```

A calculation should not silently convert invalid data into a numeric result.

---

# 2. Precision

All financial calculations use decimal arithmetic as defined in ADR-0005.

Measurements entered as decimal values should also avoid unnecessary binary floating-point conversion where practical.

Calculations should retain sufficient internal precision.

Rounding should normally happen only when presenting the final result.

Preferred:

```text
raw observations
        ↓
full precision calculation
        ↓
display rounding
```

Avoid:

```text
round input
→ calculate
→ round intermediate result
→ aggregate
→ round again
```

---

# 3. Units

Calculation names and results must include their physical units.

Examples:

```text
L/100km
kWh/100km
EUR/L
HUF/L
EUR/kWh
EUR/km
km
```

Values with incompatible dimensions must not be combined directly.

For example:

```text
L/100km
+
kWh/100km
```

has no useful meaning.

---

# 4. ICE Refuel Notation

For a RefuelEvent `Rᵢ`, define:

```text
Oᵢ = odometer in km
Lᵢ = liters added
Fᵢ = whether the tank was filled to the full reference level
Aᵢ = total monetary amount
Pᵢ = observed unit price
RAᵢ = range displayed after refueling
RBᵢ = range displayed before refueling
Cᵢ = vehicle-reported average consumption, if available
```

Events are evaluated within a single Vehicle.

---

# 5. Distance Between Observations

For two valid observations `A` and `B`:

```text
Distance = O_B - O_A
```

Requirements:

```text
O_B > O_A
```

If:

```text
O_B = O_A
```

the segment does not have a usable driving distance.

If:

```text
O_B < O_A
```

the normal calculation is invalid and the data should enter a correction workflow.

---

# 6. Verified ICE Consumption

The primary verified ICE consumption method is the **full-to-full method**.

A verified segment starts at a full-tank reference point and ends at the next compatible full-tank reference point.

Conceptually:

```text
Full A
  ↓
driving
  ↓
optional partial refuels
  ↓
Full B
```

---

## 6.1 Direct Full-to-Full

Example:

```text
Full A
O_A = 100000 km

Full B
O_B = 100500 km
L_B = 32.4 L
```

Distance:

```text
D = 100500 - 100000
  = 500 km
```

Fuel consumed:

```text
V = 32.4 L
```

Consumption:

```text
C = V / D × 100
```

Therefore:

```text
C = 32.4 / 500 × 100
  = 6.48 L/100km
```

Quality:

```text
Verified
```

Method identifier conceptually:

```text
ice.full_to_full.v1
```

---

# 7. Full → Partial → Full

Partial refuels between compatible full reference points do **not** destroy the verified segment.

Example:

```text
Full A
O = 100000

Partial B
O = 100250
L = 15 L

Full C
O = 100600
L = 23 L
```

Distance:

```text
D = 100600 - 100000
  = 600 km
```

Fuel added after the starting full reference:

```text
V = 15 + 23
  = 38 L
```

Consumption:

```text
C = 38 / 600 × 100
  ≈ 6.3333 L/100km
```

Quality:

```text
Verified
```

---

# 8. General Verified ICE Segment

Let:

```text
Rₐ
```

be a full-tank anchor and:

```text
Rᵦ
```

the next full-tank anchor.

All events between them belong to the segment.

Distance:

```text
D = Oᵦ - Oₐ
```

Fuel:

```text
V = Σ Lᵢ
```

for all refuels:

```text
a < i <= b
```

Then:

```text
Consumption =
V / D × 100
```

Requirements:

```text
Fₐ = true
Fᵦ = true
D > 0
V > 0
```

and all relevant refuel events in the interval must be available.

---

# 9. Why the Starting Refuel Quantity Is Not Included

Consider:

```text
Full A
...
Full B
```

At `A`, the tank establishes the starting reference level.

At `B`, the tank is restored to the same reference level.

Therefore the fuel added **after A and through B** replaces the fuel consumed during the segment.

The liters added at `A` belong to the previous fuel balance and are not included in the `A → B` calculation.

---

# 10. Consecutive Verified Segments

The normal calculation engine should construct verified segments using consecutive compatible full anchors.

Example:

```text
Full A
Partial B
Full C
Partial D
Full E
```

creates:

```text
A → C
C → E
```

This prevents overlapping primary segment calculations.

Longer-period verified averages can later aggregate these individual segments.

---

# 11. Partial-Only ICE Segments

Example:

```text
Partial A
Partial B
```

with:

```text
L_B = 30 L
```

does **not** imply:

```text
fuel consumed = 30 L
```

because the system does not know the exact fuel level at either boundary.

Therefore:

```text
Verified consumption = Unavailable
```

---

# 12. Estimated ICE Consumption

Stage 0 deliberately does **not** define a universal production-grade estimated consumption formula for arbitrary partial-refuel segments.

This is intentional.

A useful estimate must document:

* the observations it uses
* assumptions
* known limitations
* calculation version
* quality classification

Until such a method is validated:

```text
partial-only absolute ICE consumption
```

may remain:

```text
Unavailable
```

rather than producing a misleading L/100km value.

Any future partial-refuel estimation method will be classified as:

```text
Estimated
```

unless it satisfies requirements for a verified measurement.

---

# 13. Vehicle-Reported Consumption

If the vehicle displays its own average consumption:

```text
C_car = 6.2 L/100km
```

OmniFleet stores this as:

```text
Vehicle-reported consumption
```

It is not automatically:

```text
Verified
Estimated
```

OmniFleet does not know by default:

* the vehicle manufacturer's algorithm
* the averaging window
* whether the trip computer was reset
* whether the displayed value covers the same segment

Therefore it remains an external observation.

---

# 14. Verified vs Vehicle-Reported Consumption

A direct comparison becomes meaningful only when both values refer to sufficiently compatible periods.

If:

```text
C_verified
```

is a verified OmniFleet measurement and:

```text
C_car
```

is known to represent the same driving interval, then a relative difference may be calculated:

```text
Difference =
(C_verified / C_car - 1) × 100
```

Example:

```text
C_verified = 6.0
C_car = 6.3
```

Then:

```text
Difference =
(6.0 / 6.3 - 1) × 100
≈ -4.76%
```

Meaning:

```text
verified consumption was approximately 4.76%
lower than the vehicle-reported value
```

However, unless the observation periods are known to align, OmniFleet should not present this comparison as authoritative.

---

# 15. Experimental Driver vs. Car Metric

OmniFleet may record:

```text
range_after
```

after one refuel and:

```text
range_before
```

before the next refuel.

For two **consecutive ICE RefuelEvents** `A` and `B`:

```text
D =
O_B - O_A
```

Displayed range decrease:

```text
ΔR =
RA_A - RB_B
```

Experimental ratio:

```text
DriverVsCarRatio =
ΔR / D
```

or:

```text
DriverVsCarRatio =
(Range_After_A - Range_Before_B)
/
(Odometer_B - Odometer_A)
```

Method identifier conceptually:

```text
ice.range_delta_ratio.v1
```

Quality:

```text
Experimental
```

---

# 16. Experimental Ratio Interpretation

Under a sufficiently stable vehicle range-estimation model:

```text
≈ 1.0
```

means displayed remaining range decreased at approximately the same rate as actual traveled distance.

```text
< 1.0
```

means displayed range decreased more slowly than actual distance traveled.

```text
> 1.0
```

means displayed range decreased faster than actual distance traveled.

For example:

```text
0.83
```

means:

```text
displayed range decreased by approximately
83 km for every 100 km actually traveled
```

This may indicate more efficient driving than the initial range prediction suggested.

However, the interpretation is experimental.

---

# 17. Important Limitation of the Range Metric

Vehicle range estimators often change while driving.

They may respond to:

* recent fuel consumption
* speed
* traffic
* terrain
* ambient temperature
* HVAC usage
* manufacturer-specific algorithms

Therefore:

```text
DriverVsCarRatio
```

must **not** be described as an exact measurement of:

```text
actual consumption / predicted consumption
```

without additional assumptions.

It is better understood as:

> a comparison between displayed range depletion and actual distance traveled.

This distinction is fundamental.

---

# 18. Range Metric Preconditions

The metric requires:

```text
O_B > O_A
RA_A available
RB_B available
RA_A >= 0
RB_B >= 0
```

If:

```text
ΔR <= 0
```

normal efficiency interpretation becomes unreliable.

This may indicate:

* estimator recalibration
* incorrect observations
* unusual driving conditions

The result should normally be marked:

```text
Unavailable / not interpretable
```

rather than presented as a normal efficiency score.

---

# 19. Range Metric and Partial Refuels

The Driver vs. Car metric does not require both refuels to be full.

Example:

```text
Partial A
        ↓
driving
        ↓
Partial B
```

can still produce a range-depletion ratio if:

```text
range_after_A
range_before_B
odometer_A
odometer_B
```

are available.

That does **not** convert the segment into a verified fuel-consumption measurement.

The output remains:

```text
Experimental
```

---

# 20. Range Metric and PHEVs

The initial Driver vs. Car formula is **not applied to PHEVs**.

A PHEV may travel part of the odometer distance using electricity.

Therefore:

```text
fuel range decrease
/
total odometer distance
```

would mix different propulsion sources.

Until a PHEV-specific methodology exists:

```text
Driver vs. Car range metric for PHEV
=
Unavailable
```

---

# 21. Fuel Cost per Verified Segment

For a verified full-to-full segment:

```text
Full A
...
Full B
```

define transaction spending:

```text
S = Σ Aᵢ
```

for refuels:

```text
a < i <= b
```

in the same currency.

A useful derived metric is:

```text
Refueling Spend per km =
S / D
```

Example:

```text
S = 84 EUR
D = 600 km
```

Then:

```text
0.14 EUR/km
```

---

# 22. Meaning of Segment Spend per km

This metric should be understood as:

> refueling expenditure required to restore the fuel balance across the verified segment.

It is not intended as formal fuel-inventory accounting.

OmniFleet does not initially track which exact liters purchased at which earlier price were physically burned during each kilometer.

For practical vehicle-cost tracking, replacement/refueling spend per distance is sufficient.

---

# 23. Monthly Spending

Monthly spending is based directly on confirmed transaction totals.

For one currency:

```text
Monthly Spend =
Σ total_amount
```

for events whose:

```text
occurred_at
```

falls within the selected month.

For ChargeEvents, the canonical calendar timestamp (`started_at` vs `ended_at`) is deferred until the EV vertical slice. Until then, monthly charging spend must use one explicitly documented timestamp rather than mixing both.

Refuel and Charge spending may be reported:

```text
separately
```

or combined when:

```text
currency is identical
```

and the report clearly describes the scope.

---

# 24. Multi-Currency Spending

Without exchange-rate data:

```text
60 EUR
+
20,000 HUF
```

does not produce one combined total.

Correct reporting:

```text
EUR: 60
HUF: 20,000
```

Cross-currency totals require explicit exchange-rate observations.

---

# 25. Unit Price

If quantity and total amount are known, an effective unit price can be derived.

Fuel:

```text
Effective Fuel Price =
total_amount / liters
```

Unit:

```text
currency/L
```

Charging:

```text
Effective Charging Price =
total_amount / energy_kwh
```

Unit:

```text
currency/kWh
```

---

# 26. Observed vs Effective Unit Price

A receipt may contain an observed:

```text
unit_price
```

while OmniFleet can also derive:

```text
effective_unit_price
```

from:

```text
total_amount / quantity
```

These values may differ because of:

* rounding
* discounts
* session fees
* parking fees
* time fees
* provider billing rules

Both values may therefore have legitimate meaning.

---

# 27. Weighted Average Fuel Price

For compatible fuel transactions in the same currency:

```text
Weighted Average Fuel Price =
Σ total_amount
/
Σ liters
```

Example:

```text
20 L → 30 EUR
50 L → 80 EUR
```

Then:

```text
110 EUR / 70 L
≈ 1.571428 EUR/L
```

This is preferred over:

```text
average(transaction unit prices)
```

because transactions may contain different quantities.

---

# 28. Weighted Average Charging Price

For charging sessions in the same currency:

```text
Weighted Effective Charging Price =
Σ total_amount
/
Σ energy_kwh
```

This is an **effective** price.

If total amounts contain:

* session fees
* parking fees
* time fees

then it does not represent the pure provider energy tariff.

---

# 29. Aggregation Compatibility

Weighted averages require compatible dimensions.

Valid:

```text
EUR/L + EUR/L
```

Invalid:

```text
EUR/L + HUF/L
```

without FX conversion.

Invalid:

```text
EUR/L + EUR/kWh
```

because physical units differ.

---

# 30. Charging Session Duration

If both timestamps exist:

```text
started_at
ended_at
```

then:

```text
Duration =
ended_at - started_at
```

Requirement:

```text
ended_at >= started_at
```

Duration is derived.

The timestamps remain source observations.

---

# 31. Charging Effective Price

If:

```text
energy_kwh > 0
```

and total cost is known:

```text
EffectivePricePerKWh =
total_amount / energy_kwh
```

Example:

```text
30 kWh
14 EUR
```

Then:

```text
≈ 0.4667 EUR/kWh
```

This remains valid even if the provider tariff consisted of:

```text
0.40 EUR/kWh
+
2 EUR session fee
```

because it represents the effective transaction cost.

---

# 32. EV Consumption Terminology

OmniFleet must distinguish between different meanings of electrical consumption.

Possible concepts include:

```text
charger-delivered energy
battery-stored energy
vehicle traction energy
```

These are not necessarily identical because charging losses exist.

Therefore:

```text
kWh/100km
```

must always identify what energy basis it uses.

---

# 33. EV Grid-Energy Intensity

A useful future EV metric is:

```text
Grid Energy per 100 km
```

based on charger-reported energy delivered.

A strong measurement requires compatible battery-state boundaries.

Conceptually:

```text
Charge Anchor A
end SoC = X

↓ driving and possible intermediate charges

Charge Anchor B
end SoC = X
```

If all charging energy between those anchors is captured:

```text
E =
Σ energy_kwh
```

and:

```text
D =
O_B - O_A
```

then:

```text
GridEnergyIntensity =
E / D × 100
```

Unit:

```text
kWh/100km
```

This measures grid energy delivered per distance, including charging losses represented by the reported charging energy.

---

# 34. EV Quality Classification

The exact EV quality policy remains intentionally conservative.

A charger-energy calculation should not automatically be called:

```text
Verified vehicle consumption
```

because:

* charger measurements represent grid-delivered energy
* charging losses exist
* SoC boundaries may not align
* charging events may be missing
* battery capacity changes with conditions and age

The later EV implementation must define whether the metric is described as:

```text
Verified grid-energy intensity
```

or another appropriately precise term.

Until then, the general formula is documented but final product classification is deferred.

---

# 35. SoC-Based Energy Estimation

If usable battery capacity `B` is known:

```text
B = usable battery capacity in kWh
```

a rough battery-energy difference may be estimated as:

```text
Estimated Battery Energy =
B × (SoC difference / 100)
```

Example:

```text
B = 60 kWh
SoC change = 50%
```

Then:

```text
≈ 30 kWh
```

This is not treated as directly measured energy.

Reasons include:

* battery buffers
* capacity degradation
* temperature
* BMS estimation
* nonlinear behavior

Quality:

```text
Estimated
```

at best.

---

# 36. PHEV Analytics

PHEVs contain two independent energy streams:

```text
Fuel
Electricity
```

The following remain separate:

```text
L/100km
kWh/100km
```

OmniFleet must not add them together.

---

# 37. PHEV Combined Spending

Monetary values can be combined if they use the same currency.

For a selected interval:

```text
Combined Energy Spend =
Fuel Spend
+
Charging Spend
```

Then:

```text
Combined Energy Spend per km =
Combined Energy Spend
/
Distance
```

Unit:

```text
currency/km
```

This can provide a useful cross-energy PHEV metric without pretending liters and kWh are physically equivalent.

Exact segment-boundary rules for PHEV reporting will be defined during PHEV implementation.

---

# 38. Currency Conversion

If an explicit exchange rate exists:

```text
rate =
target currency units
/
source currency unit
```

then:

```text
Converted Amount =
Original Amount × Rate
```

Example conceptually:

```text
40,000 HUF
×
EUR/HUF rate
=
derived EUR value
```

The converted result is always:

```text
Derived
```

The original amount remains unchanged.

---

# 39. Historical FX

Historical reports must use an explicitly defined historical rate policy.

OmniFleet must not silently apply today's exchange rate to old transactions.

The final policy may use:

* transaction-day rate
* nearest available rate
* manually selected rate

but must be documented before normalized reports are implemented.

---

# 40. Calculation Result Model

A calculated metric should conceptually provide:

```text
MetricResult
- value
- unit
- quality, when relevant
- method
- method_version
```

Example:

```text
value = 6.48
unit = L/100km
quality = Verified
method = full-to-full
version = 1
```

Another:

```text
value = 0.83
unit = ratio
quality = Experimental
method = range-delta
version = 1
```

---

# 41. Suggested Method Identifiers

Initial conceptual identifiers:

```text
ice.full_to_full.v1
ice.range_delta_ratio.v1

money.effective_unit_price.v1
money.weighted_unit_price.v1
money.spend_per_distance.v1

charge.duration.v1
charge.effective_price_per_kwh.v1
```

These identifiers do not need to become database values immediately.

They provide stable terminology for:

* documentation
* tests
* future API metadata
* future calculation versioning

---

# 42. Recalculation

If any source observation used by a calculation changes:

```text
source observation changed
        ↓
previous derived result becomes stale
        ↓
recalculate
```

Example:

```text
odometer correction
100500 → 100600
```

changes:

```text
distance
consumption
cost/km
range ratio
```

where those calculations depend on the observation.

---

# 43. Event Deletion

Removing an event may change neighboring segments.

Example:

```text
Full A
Partial B
Full C
```

If `B` is deleted:

```text
fuel quantity for A → C changes
```

Therefore derived calculations must be reconstructed from the current valid event stream.

---

# 44. Calculation Ordering

Vehicle-event calculations use real-world event order rather than database creation order.

Primary inputs include:

```text
occurred_at
odometer
```

If these conflict in a way that makes the sequence ambiguous or impossible:

```text
calculation should fail or require correction
```

rather than silently choosing an arbitrary ordering.

---

# 45. Dashboard Consumption Aggregation

Primary consumption reporting should prefer verified segments.

For example:

```text
Verified Consumption Trend
```

uses verified full-to-full segments.

Estimated and experimental metrics should normally appear separately.

OmniFleet should not silently combine:

```text
Verified
Estimated
Experimental
```

into one unlabeled average.

---

# 46. Average Verified Consumption

For multiple verified ICE segments, the preferred overall average is distance-weighted.

If each segment has:

```text
Vᵢ = fuel consumed
Dᵢ = distance
```

then:

```text
Overall Consumption =
Σ Vᵢ / Σ Dᵢ × 100
```

This is preferred over:

```text
average(segment L/100km values)
```

because segments may cover different distances.

---

# 47. Example of Correct Consumption Aggregation

Segment A:

```text
300 km
18 L

= 6.0 L/100km
```

Segment B:

```text
700 km
49 L

= 7.0 L/100km
```

Simple average:

```text
6.5 L/100km
```

is not the correct overall value.

Correct:

```text
(18 + 49)
/
(300 + 700)
× 100

= 6.7 L/100km
```

---

# 48. Moving Average

Dashboard smoothing may use a moving window across verified segments.

Example:

```text
last 3 verified segments
```

The preferred moving consumption value should again use:

```text
Σ fuel
/
Σ distance
× 100
```

rather than simply averaging the three displayed L/100km values.

---

# 49. Date Buckets

Consumption is naturally event/segment-based.

Therefore the default consumption chart should use:

```text
one point per measurement segment
```

rather than forcing calculations into daily or weekly buckets.

Time buckets are more appropriate for:

```text
spending
distance
transaction count
```

when enough data exists.

---

# 50. Calculation Safety Principle

When OmniFleet has to choose between:

```text
showing a plausible-looking number
```

and:

```text
showing no number because the evidence is insufficient
```

it should choose:

```text
Unavailable
```

A missing metric is preferable to false precision.

---

# 51. Stage 0 Calculation Classification

The initial calculation set is:

| Metric                                                   | Method                                       | Initial Quality                |
| -------------------------------------------------------- | -------------------------------------------- | ------------------------------ |
| ICE consumption                                          | Full-to-full                                 | Verified                       |
| ICE consumption with partial events between full anchors | Full-to-full aggregate                       | Verified                       |
| Arbitrary partial-only ICE consumption                   | Not yet defined                              | Unavailable                    |
| Driver vs. Car range ratio                               | Range delta / distance                       | Experimental                   |
| Vehicle-reported consumption                             | External observation                         | Not an OmniFleet quality class |
| Effective fuel price                                     | Amount / liters                              | Derived                        |
| Weighted fuel price                                      | Σ amount / Σ liters                          | Derived                        |
| Spend per distance                                       | Segment spend / distance                     | Derived                        |
| Charging duration                                        | End - start                                  | Derived                        |
| Effective charging price                                 | Amount / kWh                                 | Derived                        |
| SoC-based battery energy                                 | Capacity × SoC delta                         | Estimated                      |
| EV grid-energy intensity                                 | Defined conceptually; final quality deferred | TBD                            |
| PHEV combined energy spend/km                            | Planned                                      | TBD                            |

---

# 52. Stage 0 Non-Goals

This document does not yet define:

* arbitrary partial-refuel absolute consumption estimation
* exact EV verified-consumption rules
* charging-loss correction
* PHEV equivalent-energy conversion
* ChargeEvent canonical calendar time for ordering and monthly spending
* CO₂ calculations
* WLTP comparisons
* fuel-density conversions
* weather normalization
* route normalization
* statistical confidence intervals
* predictive consumption models

These should only be introduced when real product requirements justify them.

---

# 53. Calculation Definition of Done

A new OmniFleet calculation method should not be considered complete until it defines:

1. **Name**
2. **Purpose**
3. **Required observations**
4. **Formula**
5. **Units**
6. **Validation conditions**
7. **Unavailable conditions**
8. **Quality classification where applicable**
9. **Known assumptions**
10. **Known limitations**
11. **Version identifier**
12. **Representative test scenarios**

This rule applies to both manually designed and AI-assisted future calculation work.

---

# Result

OmniFleet calculations are derived from preserved real-world observations.

The initial authoritative ICE consumption method is full-to-full, including partial refuels between compatible full-tank anchors.

Arbitrary partial refuels do not produce fake verified consumption values.

The range-based Driver vs. Car metric remains explicitly experimental and measures displayed-range depletion relative to actual distance rather than claiming to be an exact fuel-consumption measurement.

Financial aggregates use quantity-weighted calculations and preserve currency boundaries.

EV and PHEV calculations remain deliberately conservative until their measurement boundaries are defined precisely.

All important calculation methods are expected to become reproducible automated tests.

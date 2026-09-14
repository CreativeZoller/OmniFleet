# OmniFleet Domain & Calculation Scenarios

* **Status:** Stage 0 — Accepted
* **Date:** 2026-09-14

## Purpose

This document defines representative real-world scenarios for OmniFleet.

The scenarios serve as executable specifications for future implementation.

They are intended to become:

* unit tests
* domain tests
* integration tests
* regression tests

where appropriate.

The goal is to define expected behavior before implementation details influence the business rules.

---

# 1. Scenario Conventions

The examples use the following terminology:

```text
Verified
Estimated
Experimental
Unavailable
Invalid
```

These meanings follow ADR-0006.

Monetary values use explicit currencies.

Event ordering is based on real-world event time and odometer observations rather than database insertion order.

---

# 2. ICE — Direct Full-to-Full Consumption

## Given

```text
Vehicle:
Powertrain = ICE
Fuel = Diesel
```

First refuel:

```text
Odometer = 100000 km
Full tank = true
```

Second refuel:

```text
Odometer = 100500 km
Fuel added = 32.40 L
Full tank = true
```

## Expected

Distance:

```text
100500 - 100000
= 500 km
```

Consumption:

```text
32.40 / 500 × 100
= 6.48 L/100km
```

Result:

```text
Consumption = 6.48 L/100km
Quality = Verified
Method = ice.full_to_full.v1
```

---

# 3. ICE — Full → Partial → Full

## Given

First refuel:

```text
Odometer = 100000 km
Full tank = true
```

Second refuel:

```text
Odometer = 100250 km
Fuel added = 15.00 L
Full tank = false
```

Third refuel:

```text
Odometer = 100600 km
Fuel added = 23.00 L
Full tank = true
```

## Expected

Distance:

```text
100600 - 100000
= 600 km
```

Fuel added inside the segment:

```text
15 + 23
= 38 L
```

Consumption:

```text
38 / 600 × 100
≈ 6.333333 L/100km
```

Result:

```text
Consumption ≈ 6.33 L/100km
Quality = Verified
```

The partial refuel does not break the verified segment.

---

# 4. ICE — Multiple Partial Refuels Between Full Anchors

## Given

```text
Full A:
100000 km

Partial B:
100200 km
10 L

Partial C:
100400 km
12 L

Partial D:
100550 km
8 L

Full E:
100800 km
20 L
```

## Expected

Distance:

```text
800 km
```

Fuel:

```text
10 + 12 + 8 + 20
= 50 L
```

Consumption:

```text
50 / 800 × 100
= 6.25 L/100km
```

Result:

```text
6.25 L/100km
Verified
```

---

# 5. ICE — Partial → Partial

## Given

```text
Partial A:
100000 km
20 L

Partial B:
100300 km
25 L
```

No compatible full-tank anchors exist.

## Expected

```text
Verified Consumption = Unavailable
```

OmniFleet must not calculate:

```text
25 / 300 × 100
```

and present it as actual consumption.

---

# 6. ICE — Full → Partial Without Closing Anchor

## Given

```text
Full A:
100000 km

Partial B:
100300 km
20 L
```

## Expected

The verified segment is still open.

```text
Verified Consumption = Unavailable
```

The system waits for a later compatible full-tank event.

---

# 7. ICE — Starting Full Refuel Quantity Is Excluded

## Given

```text
Full A:
100000 km
Fuel added = 40 L

Full B:
100500 km
Fuel added = 30 L
```

## Expected

For the segment:

```text
A → B
```

fuel used is:

```text
30 L
```

not:

```text
40 + 30 = 70 L
```

Consumption:

```text
30 / 500 × 100
= 6.0 L/100km
```

---

# 8. ICE — Consecutive Verified Segments

## Given

```text
Full A:
100000 km

Partial B:
100250 km
15 L

Full C:
100600 km
23 L

Partial D:
100900 km
18 L

Full E:
101200 km
20 L
```

## Expected

Two primary verified segments exist:

```text
A → C
C → E
```

Not:

```text
A → E
```

as an additional overlapping primary segment.

Longer-period aggregation may later combine the two verified segments.

---

# 9. ICE — Zero Distance

## Given

```text
Full A:
100000 km

Full B:
100000 km
30 L
```

## Expected

```text
Distance = 0
Calculation = Invalid
```

OmniFleet must not divide by zero.

---

# 10. ICE — Decreasing Odometer

## Given

```text
Previous event:
100500 km

New event:
100400 km
```

## Expected

Normal event creation:

```text
REJECT
```

or route into a future correction workflow.

No normal consumption segment is calculated.

---

# 11. ICE — Positive Fuel Quantity Required

## Given

```text
Fuel added = 0 L
```

## Expected

```text
REJECT
```

For a RefuelEvent:

```text
liters > 0
```

is required.

---

# 12. ICE — Negative Fuel Quantity

## Given

```text
Fuel added = -15 L
```

## Expected

```text
REJECT
```

---

# 13. ICE — Wrong Fuel Type

## Given

Vehicle:

```text
Powertrain = ICE
Supported fuel = Diesel
```

Refuel:

```text
Fuel type = Petrol
```

## Expected

```text
REJECT
```

The event fuel must be supported by the Vehicle.

---

# 14. ICE — Multi-Fuel Vehicle

## Given

Vehicle:

```text
Powertrain = ICE
Supported fuels:
- Petrol
- LPG
```

Events:

```text
Refuel A:
Petrol

Refuel B:
LPG
```

## Expected

Both events are valid.

Fuel type is stored per event.

---

# 15. EV Cannot Refuel

## Given

```text
Vehicle powertrain = EV
```

## When

Create:

```text
RefuelEvent
```

## Expected

```text
REJECT
```

---

# 16. ICE Cannot Charge

## Given

```text
Vehicle powertrain = ICE
```

## When

Create:

```text
ChargeEvent
```

## Expected

```text
REJECT
```

---

# 17. PHEV Supports Both Event Types

## Given

Vehicle:

```text
Powertrain = PHEV
Fuel = Petrol
Electric configuration = available
```

## When

Create:

```text
Petrol RefuelEvent
```

and later:

```text
ChargeEvent
```

## Expected

Both events are valid.

---

# 18. Fuel Cost — Exact Total

## Given

```text
Liters = 40
Unit price = 1.60 EUR/L
Total amount = 64.00 EUR
```

## Expected

Stored observations:

```text
Liters = 40
Observed Unit Price = 1.60 EUR/L
Total Amount = 64.00 EUR
```

Transaction is valid.

---

# 19. Fuel Cost — Receipt Rounding

## Given

```text
Liters = 42.17
Unit price = 1.579 EUR/L
Total amount = 66.58 EUR
```

## Expected

The transaction is valid.

OmniFleet preserves:

```text
66.58 EUR
```

as the explicitly confirmed total.

It must not overwrite the total merely because multiplication produces a slightly different decimal result.

---

# 20. Fuel Cost — Total Derived

## Given

```text
Liters = 40
Unit price = 1.60 EUR/L
Total amount = missing
Currency = EUR
```

## Expected

Derived:

```text
Total amount = 64.00 EUR
```

The value is system-derived.

---

# 21. Fuel Cost — Effective Unit Price Derived

## Given

```text
Liters = 40
Total amount = 64 EUR
Unit price = missing
```

## Expected

```text
Effective unit price =
64 / 40
= 1.60 EUR/L
```

The value is derived.

---

# 22. Weighted Fuel Price

## Given

Transaction A:

```text
20 L
30 EUR
```

Transaction B:

```text
50 L
80 EUR
```

## Expected

Total fuel:

```text
70 L
```

Total spending:

```text
110 EUR
```

Weighted average:

```text
110 / 70
≈ 1.571428 EUR/L
```

Result:

```text
≈ 1.5714 EUR/L
```

OmniFleet must not use the simple arithmetic average of transaction unit prices.

---

# 23. Weighted Fuel Price Across Different Quantities

## Given

```text
5 L @ 2.00 EUR/L
50 L @ 1.50 EUR/L
```

## Expected

Costs:

```text
10 EUR
75 EUR
```

Total:

```text
85 EUR
55 L
```

Weighted average:

```text
85 / 55
≈ 1.54545 EUR/L
```

Not:

```text
1.75 EUR/L
```

---

# 24. Mixed Currency Spending

## Given

```text
Refuel A:
60 EUR

Refuel B:
20000 HUF
```

No exchange-rate observations exist.

## Expected

Report:

```text
EUR: 60
HUF: 20000
```

Unavailable:

```text
Combined monetary total
```

---

# 25. Currency Conversion Does Not Rewrite Original

## Given

Original transaction:

```text
40000 HUF
```

Later an exchange rate becomes available.

Derived report:

```text
≈ X EUR
```

## Expected

Original event remains:

```text
40000 HUF
```

The converted value exists only as derived reporting data.

---

# 26. Free Charging

## Given

```text
Energy = 15 kWh
Total amount = 0 EUR
```

## Expected

```text
ACCEPT
```

Zero-cost energy events are valid.

---

# 27. Negative Transaction Cost

## Given

```text
Total amount = -10 EUR
```

for a normal RefuelEvent or ChargeEvent.

## Expected

```text
REJECT
```

Refunds require a separate future model.

---

# 28. Charge Event — Valid Session

## Given

Vehicle:

```text
Powertrain = EV
```

Charge:

```text
Odometer = 50000 km
Start SoC = 20%
End SoC = 80%
Energy = 42 kWh
Total = 18 EUR
Started = 18:00
Ended = 18:45
```

## Expected

Event is valid.

Derived duration:

```text
45 minutes
```

Derived effective price:

```text
18 / 42
≈ 0.428571 EUR/kWh
```

---

# 29. Charge Event — Invalid SoC

## Given

```text
Start SoC = -5%
```

## Expected

```text
REJECT
```

---

# 30. Charge Event — SoC Above 100

## Given

```text
End SoC = 105%
```

## Expected

```text
REJECT
```

---

# 31. Charge Event — End Before Start

## Given

```text
Started at 19:00
Ended at 18:30
```

## Expected

```text
REJECT
```

---

# 32. Charge Event — Zero Energy

## Given

```text
Energy = 0 kWh
```

where energy is explicitly supplied as a normal charging observation.

## Expected

Normal energy-added interpretation:

```text
REJECT
```

If future workflows need zero-energy sessions, they require explicit semantics rather than reusing a normal ChargeEvent ambiguously.

---

# 33. Charge Event — Unknown Battery Capacity

## Given

```text
Vehicle = EV
Battery capacity = unknown
```

Valid ChargeEvent:

```text
Energy = 30 kWh
```

## Expected

```text
ACCEPT
```

Battery capacity is optional metadata.

---

# 34. Refuel — Unknown Tank Capacity

## Given

```text
Vehicle = ICE
Tank capacity = unknown
```

Valid RefuelEvent:

```text
Liters = 35
```

## Expected

```text
ACCEPT
```

Tank capacity is optional metadata.

---

# 35. Effective Charging Price With Session Fee

## Given

Provider pricing:

```text
30 kWh
0.40 EUR/kWh
2 EUR session fee
```

Actual total:

```text
14 EUR
```

## Expected

Observed provider energy tariff may be:

```text
0.40 EUR/kWh
```

Effective transaction price:

```text
14 / 30
≈ 0.466667 EUR/kWh
```

Both have different meanings.

---

# 36. Driver vs. Car — Neutral Case

## Given

Refuel A:

```text
Odometer = 100000 km
Range after = 700 km
```

Next Refuel B:

```text
Odometer = 100500 km
Range before = 200 km
```

## Expected

Actual distance:

```text
500 km
```

Displayed range decrease:

```text
700 - 200
= 500 km
```

Ratio:

```text
500 / 500
= 1.00
```

Result:

```text
Driver vs. Car Ratio = 1.00
Quality = Experimental
```

---

# 37. Driver vs. Car — Better Than Initial Prediction

## Given

Refuel A:

```text
Odometer = 100000
Range after = 700
```

Refuel B:

```text
Odometer = 100500
Range before = 285
```

## Expected

Distance:

```text
500 km
```

Range depletion:

```text
700 - 285
= 415 km
```

Ratio:

```text
415 / 500
= 0.83
```

Result:

```text
0.83
Experimental
```

Interpretation:

Displayed range decreased by approximately 83 km for every 100 km actually traveled.

This may indicate better real-world efficiency than the initial range prediction suggested.

---

# 38. Driver vs. Car — Worse Than Initial Prediction

## Given

Refuel A:

```text
Odometer = 100000
Range after = 700
```

Refuel B:

```text
Odometer = 100500
Range before = 115
```

## Expected

Range depletion:

```text
700 - 115
= 585 km
```

Ratio:

```text
585 / 500
= 1.17
```

Result:

```text
1.17
Experimental
```

---

# 39. Driver vs. Car — Missing Range

## Given

```text
Range after previous refuel = unknown
```

## Expected

```text
Driver vs. Car Ratio = Unavailable
```

Not:

```text
0
```

---

# 40. Driver vs. Car — Range Increased

## Given

Refuel A:

```text
Range after = 600 km
```

Next Refuel B:

```text
Range before = 650 km
```

Distance traveled:

```text
100 km
```

## Expected

```text
Range delta = -50 km
```

Normal ratio interpretation is not meaningful.

Result:

```text
Unavailable / Not Interpretable
```

The system should not display a normal negative efficiency ratio without explicit future methodology.

---

# 41. Driver vs. Car — Partial Refuels

## Given

Refuel A:

```text
Full tank = false
Range after = 600
Odometer = 100000
```

Refuel B:

```text
Full tank = false
Range before = 180
Odometer = 100500
```

## Expected

The experimental range metric may still be calculated:

```text
(600 - 180) / 500
= 0.84
```

Result:

```text
0.84
Experimental
```

But:

```text
Verified fuel consumption = Unavailable
```

The two calculations remain independent.

---

# 42. Driver vs. Car — PHEV

## Given

```text
Vehicle = PHEV
```

Valid range and odometer observations exist.

## Expected

Initial ICE range-delta method:

```text
Unavailable
```

The method is not applied to PHEVs because electric propulsion may contribute to traveled distance.

---

# 43. Vehicle-Reported Consumption

## Given

Vehicle display:

```text
6.2 L/100km
```

## Expected

Stored as:

```text
Vehicle-reported consumption = 6.2 L/100km
```

It is not automatically classified:

```text
Verified
```

or:

```text
Estimated
```

It remains an external observation.

---

# 44. Vehicle-Reported vs Verified Consumption

## Given

Compatible interval:

```text
Vehicle-reported = 6.3 L/100km
Verified OmniFleet = 6.0 L/100km
```

## Expected

Relative difference:

```text
(6.0 / 6.3 - 1) × 100
≈ -4.7619%
```

Interpretation:

```text
Verified consumption was approximately
4.76% lower than vehicle-reported consumption
```

This comparison should only be shown if both observations reasonably refer to the same interval.

---

# 45. Average Verified Consumption — Correct Weighting

## Given

Segment A:

```text
300 km
18 L
6.0 L/100km
```

Segment B:

```text
700 km
49 L
7.0 L/100km
```

## Expected

Total:

```text
1000 km
67 L
```

Average:

```text
67 / 1000 × 100
= 6.7 L/100km
```

Not:

```text
(6.0 + 7.0) / 2
= 6.5 L/100km
```

---

# 46. Moving Average Across Three Verified Segments

## Given

```text
Segment A:
200 km
12 L

Segment B:
500 km
32 L

Segment C:
300 km
21 L
```

## Expected

Total distance:

```text
1000 km
```

Total fuel:

```text
65 L
```

Moving consumption:

```text
65 / 1000 × 100
= 6.5 L/100km
```

The moving average is distance-weighted.

---

# 47. Monthly Spending Uses Event Time

## Given

Refuel occurred:

```text
2026-08-31 20:00
```

Recorded into OmniFleet:

```text
2026-09-02 08:00
```

## Expected

The transaction belongs to:

```text
August 2026
```

for normal spending reports.

`recorded_at` does not determine the spending month.

---

# 48. Historical Event Entered Later

## Given

Today:

```text
2026-09-14
```

User creates:

```text
Refuel occurred_at = 2026-09-01
```

with otherwise valid data.

## Expected

```text
ACCEPT
```

Historical entry is valid.

---

# 49. Event Creation Order Differs From Event Time

## Given

Event B is inserted first:

```text
Occurred = 2026-09-10
Odometer = 100500
```

Later Event A is inserted:

```text
Occurred = 2026-09-01
Odometer = 100000
```

## Expected

Vehicle history ordering becomes:

```text
A
B
```

not database insertion order.

Derived calculations use real-world ordering.

---

# 50. Contradictory Time and Odometer

## Given

Event A:

```text
2026-09-10
100500 km
```

Event B:

```text
2026-09-11
100400 km
```

## Expected

The normal event sequence is inconsistent.

Result:

```text
Invalid / requires correction
```

OmniFleet must not silently calculate distance as:

```text
-100 km
```

---

# 51. Historical Odometer Correction

## Given

Initial events:

```text
Full A:
100000 km

Full B:
100500 km
32 L
```

Initial result:

```text
6.4 L/100km
```

User corrects A:

```text
99900 km
```

## Expected

New distance:

```text
600 km
```

New consumption:

```text
32 / 600 × 100
≈ 5.3333 L/100km
```

Old derived result becomes stale.

Correct result:

```text
≈ 5.33 L/100km
Verified
```

---

# 52. Historical Fuel Quantity Correction

## Given

Verified segment originally contains:

```text
Fuel = 32 L
Distance = 500 km
```

Result:

```text
6.4 L/100km
```

Fuel corrected to:

```text
30 L
```

## Expected

Recalculated:

```text
30 / 500 × 100
= 6.0 L/100km
```

---

# 53. Historical Full-Tank Flag Correction

## Given

```text
Full A
Partial B
Full C
```

produces one verified:

```text
A → C
```

segment.

Later B is corrected to:

```text
Full B
```

## Expected

Primary verified segments become:

```text
A → B
B → C
```

Derived segment structure must be rebuilt.

---

# 54. Deleting Partial Refuel Inside Verified Segment

## Given

```text
Full A
Partial B = 15 L
Full C = 23 L
```

Initial:

```text
Fuel = 38 L
```

B is deleted.

## Expected

Current valid data implies:

```text
Fuel = 23 L
```

for A → C.

Derived analytics must be recalculated.

The application must not retain the old 38 L result.

---

# 55. Household Vehicle Access — Owner

## Given

```text
User A = Owner of Household X
Vehicle V belongs to Household X
```

## Expected

User A may access Vehicle V.

---

# 56. Household Vehicle Access — Member

## Given

```text
User B = Member of Household X
Vehicle V belongs to Household X
```

## Expected

User B may access Vehicle V according to Member permissions.

---

# 57. Cross-Household Access

## Given

```text
User A belongs to Household X
Vehicle V belongs to Household Y
```

## Expected

```text
ACCESS DENIED
```

Knowing the Vehicle ID does not grant access.

---

# 58. Duplicate Household Membership

## Given

```text
User A
already has membership in Household X
```

## When

Another identical active membership is created.

## Expected

```text
REJECT
```

---

# 59. Household Must Keep an Owner

## Given

```text
Household X
Owners = [User A]
```

## When

User A is demoted to Member or removed from the household.

## Expected

```text
REJECT
```

---

# 60. Multiple Household Owners

## Given

```text
Owners:
User A
User B
```

## When

User A becomes Member.

## Expected

```text
ACCEPT
```

User B remains Owner.

---

# 61. User Can Belong to Multiple Households

## Given

```text
User A belongs to Household X
```

## When

User A joins Household Y.

## Expected

Both memberships may coexist.

Resources remain isolated by Household.

---

# 62. Membership Removal Preserves Historical Attribution

## Given

User B recorded:

```text
Refuel R
```

while a Member of Household X.

Later:

```text
User B leaves Household X
```

## Expected

Refuel R remains.

Historical attribution remains identifiable where permitted by the final account/privacy model.

---

# 63. Vehicle Archive

## Given

Vehicle has:

```text
5 years of RefuelEvents
```

## When

Vehicle is no longer actively used.

## Expected

Vehicle becomes:

```text
Archived
```

Historical events remain available.

Reports remain possible.

---

# 64. Raw vs Derived — Cache Conflict

## Given

Raw events calculate:

```text
6.4 L/100km
```

A future cached result contains:

```text
6.7 L/100km
```

## Expected

The cache is stale.

The reproducible result from authoritative source observations wins.

---

# 65. OCR Suggestion Is Not Yet Source Data

## Given

OCR produces:

```text
Liters = 42.5
Total = 68.20 EUR
```

User has not confirmed the values.

## Expected

These values are:

```text
Suggested
```

not authoritative event observations.

---

# 66. OCR Confirmation

## Given

OCR suggests:

```text
42.5 L
68.20 EUR
```

User confirms both.

## Expected

The confirmed values may become recorded event observations.

Their source/provenance may later indicate:

```text
OCR-confirmed
```

---

# 67. Imported Transaction Preserves Currency

## Given

External import contains:

```text
38500 HUF
```

## Expected

OmniFleet stores:

```text
38500 HUF
```

It does not silently convert the value to the household's preferred reporting currency.

---

# 68. SoC-Based Energy Estimate

## Given

Usable battery capacity:

```text
60 kWh
```

SoC change:

```text
20% → 70%
```

## Expected

Estimated battery-energy difference:

```text
60 × 0.50
= 30 kWh
```

Quality:

```text
Estimated
```

It must not be treated as charger-measured energy.

---

# 69. Charger-Delivered Energy and SoC Differ

## Given

Battery:

```text
60 kWh usable
```

SoC:

```text
20% → 70%
```

SoC estimate suggests:

```text
≈ 30 kWh
```

Charger reports:

```text
33.5 kWh delivered
```

## Expected

Both values remain valid observations/derivations with different meanings.

OmniFleet must not overwrite:

```text
33.5 kWh
```

with:

```text
30 kWh
```

Charging losses and estimation differences may explain the gap.

---

# 70. PHEV Combined Spending

## Given

Selected period:

```text
Fuel spending = 80 EUR
Charging spending = 35 EUR
Distance = 900 km
```

## Expected

Combined energy spending:

```text
115 EUR
```

Combined energy spend per km:

```text
115 / 900
≈ 0.127778 EUR/km
```

Result:

```text
≈ 0.128 EUR/km
```

No attempt is made to add:

```text
liters + kWh
```

---

# 71. PHEV Mixed-Currency Spending

## Given

```text
Fuel = 80 EUR
Charging = 12000 HUF
```

No FX data exists.

## Expected

Combined energy spending:

```text
Unavailable
```

Separate totals remain available.

---

# 72. Database Referential Integrity

## Given

Attempt to create:

```text
Vehicle
household_id = nonexistent Household
```

## Expected

```text
REJECT
```

Application validation and/or foreign-key constraint prevents orphaned data.

---

# 73. Database Membership Uniqueness

## Given

Existing membership:

```text
(User A, Household X)
```

## When

The same membership is inserted again.

## Expected

```text
REJECT
```

Persistence must enforce uniqueness.

---

# 74. Session Does Not Contain Authoritative Role

## Given

Authenticated User A is:

```text
Member
```

Client submits request claiming:

```text
role = Owner
```

## Expected

The server ignores the claim.

Authorization uses the actual HouseholdMembership.

---

# 75. Client Cannot Choose Event Recorder

## Given

Authenticated:

```text
User A
```

CreateRefuel payload attempts:

```text
recorded_by_user_id = User B
```

## Expected

The server records:

```text
recorded_by = User A
```

or rejects the unsupported field.

Client input is not authoritative for identity.

---

# 76. Unauthenticated Vehicle Request

## Given

No valid authenticated session.

## When

```text
GET /api/vehicles
```

## Expected

```text
401 / unauthenticated
```

according to the finalized API convention.

---

# 77. Session Logout

## Given

Valid authenticated session exists.

## When

User logs out.

## Expected

Server-side session becomes invalid.

Reusing the previous session credential must not authenticate future requests.

---

# 78. Full-to-Full Calculation Quality Cannot Be User-Selected

## Given

Partial-only segment.

User attempts to label the calculation:

```text
Verified
```

## Expected

Not allowed.

Measurement quality is determined by calculation rules.

---

# 79. Missing Metric Must Not Become Zero

## Given

No compatible full-to-full segment exists.

## Expected

API/domain result:

```text
Consumption = Unavailable
```

Not:

```text
Consumption = 0 L/100km
```

---

# 80. Derived Metric Includes Method

## Given

Valid full-to-full segment calculates:

```text
6.48 L/100km
```

## Expected

Conceptual result includes:

```text
Value = 6.48
Unit = L/100km
Quality = Verified
Method = ice.full_to_full
Version = 1
```

This does not require immediate database persistence of all metadata.

---

# 81. Experimental Method May Change

## Given

Historical events contain:

```text
odometer
range_after
range_before
```

Driver-vs-Car algorithm v1 produces:

```text
0.84
```

Later algorithm v2 is introduced.

## Expected

Raw observations remain unchanged.

The experimental result may be recalculated.

The old formula must not redefine historical source data.

---

# 82. Different Currencies Cannot Produce Weighted Average

## Given

```text
30 L
45 EUR

40 L
26000 HUF
```

## Expected

A single weighted:

```text
currency/L
```

result is unavailable without FX conversion.

Separate currency-specific reporting remains valid.

---

# 83. Fuel and Electricity Cannot Share Unit Price Average

## Given

```text
1.60 EUR/L

0.40 EUR/kWh
```

## Expected

The system must not calculate:

```text
1.00 EUR/unit
```

or any similar meaningless average.

Different physical units remain separate.

---

# 84. Event Timestamp and Record Timestamp Remain Separate

## Given

```text
Occurred:
2026-09-10 18:30

Recorded:
2026-09-14 10:00
```

## Expected

Both values may be retained.

Analytics normally use:

```text
occurred_at
```

while auditing may use:

```text
recorded_at
```

---

# 85. Server Timezone Must Not Change Historical Meaning

## Given

A refuel occurs at a known real-world instant.

OmniFleet is later deployed on a server in another timezone.

## Expected

The persisted event still represents the same instant.

Changing server timezone must not shift historical event meaning.

---

# 86. Historical Algorithm Regression Test

## Given

A known fixture:

```text
Full A:
100000 km

Partial B:
100250 km
15 L

Full C:
100600 km
23 L
```

## Expected Forever Unless Method Version Changes

```text
Distance = 600 km
Fuel = 38 L
Consumption ≈ 6.333333 L/100km
Quality = Verified
Method = ice.full_to_full.v1
```

Any implementation change producing a different result should fail regression testing unless the calculation method is intentionally versioned.

---

# 87. Stage 0 Minimum Test Set

Before the ICE consumption engine is considered implemented, automated tests should cover at minimum:

```text
direct full-to-full
full-partial-full
multiple partials
partial-only unavailable
zero distance
decreasing odometer
fuel compatibility
weighted unit price
currency separation
historical correction
range ratio neutral
range ratio better
range ratio worse
missing range
PHEV range ratio unavailable
```

---

# 88. Household Creation Creates Owner Membership

## Given

```text
User A exists
```

## When

User A creates Household X.

## Expected

```text
Household X exists
User A has an Owner membership in Household X
Household X has at least one Owner
```

Household creation and the initial Owner membership succeed or fail together.

---

# 89. HEV Is Modeled as ICE

## Given

```text
Vehicle is a conventional non-plug-in hybrid
```

## Expected

Initial powertrain:

```text
ICE
```

A RefuelEvent is valid.

A ChargeEvent is rejected.

---

# 90. Stage 0 Scenario Philosophy

These scenarios are not intended to describe every future edge case.

They establish the initial expected behavior of the domain.

When a new real-world case appears:

```text
observe
↓
decide expected behavior
↓
document scenario
↓
add regression test
↓
implement
```

rather than:

```text
encounter case
↓
patch code
↓
hope behavior remains consistent
```

---

# Result

The OmniFleet Stage 0 scenarios define concrete expected behavior for:

* household ownership
* authorization
* vehicle capabilities
* ICE refueling
* EV charging
* PHEV behavior
* monetary calculations
* multi-currency handling
* verified consumption
* experimental range analytics
* historical corrections
* derived-data recalculation
* raw-data preservation

These scenarios should become the basis of the application's automated domain and integration test suite as the corresponding vertical slices are implemented.

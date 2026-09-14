# OmniFleet Domain Model

## Status

**Stage 0 — Draft**

This document defines the initial domain boundaries and terminology used by OmniFleet.

It intentionally describes the domain independently from the database schema, HTTP API, frontend implementation, or persistence technology.

The goal is to establish what the system believes exists before deciding how those concepts should be stored or exposed.

---

# 1. Domain Principles

OmniFleet follows several core modeling principles.

## 1.1 Raw observations are the source of truth

Values entered or observed during a refuel or charging session should be preserved as originally recorded.

Examples include:

* odometer reading
* liters added
* energy added
* transaction amount
* currency
* onboard range estimate
* State of Charge
* timestamp

Calculated metrics such as consumption or cost per kilometer are derived from these observations.

Where practical, derived values should remain reproducible from the original data.

---

## 1.2 Calculated values have different confidence levels

Not every calculated consumption value represents the same level of certainty.

OmniFleet distinguishes between:

* **verified** — based on a measurement method with sufficient information to calculate the value reliably
* **estimated** — derived from incomplete or approximate information
* **experimental** — based on a method whose usefulness must be validated against real-world data

The quality of a calculated metric is part of its meaning.

---

## 1.3 Vehicles belong to households

Vehicles are not directly owned by individual application users.

Instead, users participate in a household and vehicles belong to that household.

This allows multiple household members to interact with the same vehicles without duplicating ownership logic.

---

## 1.4 Roles belong to memberships

A user's role is contextual to a household.

A user is therefore not globally an `owner` or `member`.

Instead:

```text
User
  ↓
Household Membership
  ↓
Role
```

This allows the authorization model to remain flexible.

---

## 1.5 Vehicle powertrain determines supported events

A vehicle does not need to fit exclusively into a fuel or electricity data model.

Instead, its powertrain determines which energy events are valid.

Examples:

```text
ICE
→ Refuel

EV
→ Charge

PHEV
→ Refuel
→ Charge
```

This is particularly important for plug-in hybrid vehicles.

---

# 2. Core Domain

The initial OmniFleet domain contains the following concepts:

```text
Household
│
├── Membership
│      └── User
│
└── Vehicle
       │
       ├── Refuel Event
       │
       └── Charge Event
```

Additional domains such as locations, attachments, OCR and reporting will be introduced later without changing the meaning of these core concepts.

---

# 3. User

A `User` represents a person who can authenticate with OmniFleet.

A user exists independently from any individual household.

Conceptual properties:

```text
User
- id
- name
- email
- authentication credentials
- created_at
```

Authentication-specific details such as password hashes, sessions and tokens belong to the authentication/infrastructure model rather than the vehicle domain itself.

A user may theoretically belong to more than one household even if the initial UI only exposes a single-household workflow.

---

# 4. Household

A `Household` is the ownership and collaboration boundary within OmniFleet.

It groups:

* users
* vehicles
* vehicle activity

Conceptual properties:

```text
Household
- id
- name
- created_at
```

A household may contain:

```text
Household
├── User A
├── User B
├── Vehicle A
├── Vehicle B
└── Vehicle C
```

The household is the primary authorization boundary for vehicle data.

A user should not be able to access vehicles or vehicle activity outside a household they belong to.

---

# 5. Household Membership

A `HouseholdMembership` connects a User to a Household.

Conceptually:

```text
HouseholdMembership
- household
- user
- role
- joined_at
```

Initial roles:

```text
Owner
Member
```

## Owner

An Owner may eventually be allowed to:

* manage household settings
* invite or remove members
* create vehicles
* archive vehicles
* manage permissions

## Member

A Member may eventually be allowed to:

* view household vehicles
* record refuels
* record charging sessions
* view analytics and reports

Detailed authorization rules are intentionally deferred until the authentication stage.

The important domain decision is that the **role belongs to the membership**, not directly to the User.

---

# 6. Vehicle

A `Vehicle` represents a physical vehicle tracked by a household.

Conceptual properties may include:

```text
Vehicle
- id
- household
- make
- model
- year
- registration / plate
- powertrain
- fuel configuration
- battery configuration
- optional capacities
- status
- created_at
```

The exact storage schema is intentionally not defined in this document.

---

# 7. Vehicle Powertrain

The initial powertrain categories are:

```text
ICE
EV
PHEV
```

Additional powertrain configurations may be introduced later if there is a real-world requirement.

## ICE

Internal combustion engine vehicle.

Supported event type:

```text
Refuel
```

Possible fuel types may include:

```text
Petrol
Diesel
LPG
CNG
```

The final fuel-type enumeration will be defined separately.

---

## EV

Battery electric vehicle.

Supported event type:

```text
Charge
```

An EV does not generate fuel refuel events.

---

## PHEV

Plug-in hybrid vehicle.

Supported event types:

```text
Refuel
Charge
```

A PHEV must not be modeled as exclusively an ICE or EV vehicle.

A typical event stream may therefore look like:

```text
Refuel
Charge
Charge
Refuel
Charge
Refuel
```

Fuel and electricity remain separate energy measurements.

Metrics using incompatible units must not be combined directly.

For example:

```text
L/100km
```

and:

```text
kWh/100km
```

cannot be meaningfully averaged together.

Cost-based metrics such as:

```text
EUR/km
```

may later provide a common comparison layer.

---

# 8. Vehicle Status

Vehicles should support a lifecycle that does not require historical records to be deleted.

At minimum:

```text
Active
Archived
```

Archiving a vehicle should preserve:

* refuel history
* charging history
* calculations
* reports

The system should generally prefer archival over destructive deletion once activity exists.

Exact deletion rules will be defined later.

---

# 9. Refuel Event

A `RefuelEvent` represents fuel added to a vehicle.

It is an observation of a real-world event.

Conceptual input data may include:

```text
RefuelEvent
- vehicle
- recorded_by
- timestamp
- odometer
- fuel_type
- liters
- unit_price
- total_amount
- currency
- full_tank
- range_before
- range_after
- onboard_average_consumption
- notes
```

The fields above are not yet a database schema.

They describe the information OmniFleet may need to represent.

---

## 9.1 Required observations

For the initial product, the following are expected to be fundamental:

```text
vehicle
timestamp
odometer
liters
cost information
currency
full / partial indicator
```

The exact relationship between:

```text
unit_price
liters
total_amount
```

will be defined during the event-model stage.

The application may calculate one value from the others while still preserving user-entered observations appropriately.

---

## 9.2 Odometer

The odometer is an important ordering and distance measurement.

For normal vehicle operation:

```text
new odometer >= previous odometer
```

is expected.

However, the domain must eventually consider exceptional situations such as:

* incorrect historical entries
* odometer correction
* dashboard/instrument replacement
* imported legacy data

Therefore the application should not blindly encode business assumptions without documenting their correction workflow.

For the initial MVP, monotonically increasing readings are the expected normal case.

---

# 10. Full vs Partial Refuel

A refuel must record whether the vehicle was filled to a known reference level, initially represented as:

```text
full_tank = true / false
```

This distinction is critical for consumption analysis.

A partial refuel cannot automatically be treated as the amount of fuel consumed since the previous event.

Example:

```text
Fuel remaining before stop: unknown
Fuel added: 30 L
Fuel remaining after stop: unknown
```

Therefore:

```text
30 L
```

is an observed purchase quantity, not necessarily:

```text
fuel consumed since previous refuel
```

---

# 11. Range Observations

For vehicles that provide an estimated remaining range, OmniFleet may record:

```text
range_before
range_after
```

These values represent the vehicle's onboard prediction at specific moments.

They are **observations**, not verified measurements of remaining physical fuel or energy.

They may later be used by experimental analytics.

The original values should therefore remain preserved even if the calculation methodology changes.

---

# 12. Onboard Consumption Observation

If the vehicle exposes its own consumption estimate, OmniFleet may optionally record it.

For an ICE vehicle this might be:

```text
6.3 L/100km
```

This represents:

> what the vehicle reported at the time

and must remain conceptually separate from OmniFleet's own calculated consumption.

Possible future comparison:

```text
Vehicle-reported consumption
vs.
OmniFleet verified consumption
```

---

# 13. Charge Event

A `ChargeEvent` represents electrical energy added to a compatible vehicle.

Conceptual observations may include:

```text
ChargeEvent
- vehicle
- recorded_by
- started_at
- ended_at
- odometer
- start_soc
- end_soc
- energy_kwh
- total_amount
- currency
- charger_type
- power_kw
- notes
```

This model will be refined during the EV/PHEV stage.

Stage 0 only establishes that Charge is an independent energy event rather than a variation of Refuel.

---

# 14. State of Charge

EV and PHEV charging may contain:

```text
start_soc
end_soc
```

expressed as percentages.

Example:

```text
start_soc = 23%
end_soc   = 81%
```

State of Charge is a vehicle-reported battery state observation.

It does not necessarily represent an exact laboratory measurement of energy stored.

Actual energy added should therefore remain independently recorded where available:

```text
energy_kwh
```

---

# 15. Currency

Every monetary transaction preserves its original currency.

Conceptually:

```text
Money
- amount
- currency
```

Examples:

```text
72.41 EUR
38,500 HUF
```

OmniFleet must not silently convert historical transaction values into another currency.

Initial reporting may keep currencies separate.

Future normalized reporting may introduce exchange-rate observations containing:

```text
source_currency
target_currency
rate
rate_date
rate_source
```

The original transaction amount must remain unchanged.

---

# 16. Derived Data

Derived data is calculated from raw observations.

Examples include:

```text
distance traveled
L/100km
kWh/100km
cost/km
weighted average unit price
monthly spending
driver-vs-car efficiency
```

Derived values should generally be reproducible.

This means the system should avoid making calculated values the only remaining representation of an observation.

---

# 17. Measurement Quality

Calculated consumption metrics must carry enough context to explain how trustworthy they are.

Initial conceptual categories:

## Verified

A calculation based on sufficient real-world measurements.

Example:

```text
full-to-full ICE consumption
```

---

## Estimated

A useful calculation based on incomplete or indirect observations.

The result may be displayed, but must not be presented as equivalent to a verified measurement.

---

## Experimental

A metric based on a methodology being validated through real-world use.

Initial example:

```text
range-based Driver vs. Car efficiency
```

Experimental metrics may:

* change formula
* gain additional assumptions
* be removed
* become validated later

without requiring changes to the original raw observations.

---

# 18. Event Attribution

Vehicle events should preserve who recorded them.

Conceptually:

```text
recorded_by_user
```

This allows:

* household activity history
* auditability
* understanding who entered a measurement

It does not imply that the user owns the vehicle.

---

# 19. Time

Real-world events occur at a specific time.

OmniFleet should distinguish:

```text
event time
```

from:

```text
record creation time
```

Example:

A refuel may occur:

```text
2026-09-14 17:20
```

but be entered into OmniFleet:

```text
2026-09-15 08:10
```

These are different facts.

Persistence details and timezone handling will be specified later.

---

# 20. Future Domain Boundaries

Several known concepts are deliberately excluded from the core domain definition for now.

These include:

## Places

Reusable:

* fuel stations
* charging stations
* home charging
* other locations

---

## Attachments

Supporting documents such as:

* receipt images
* PDFs
* invoices

---

## OCR

Extraction of suggested values from attachments.

OCR output is not authoritative source data until confirmed by a user.

---

## Exchange Rates

External or manually supplied observations used for normalized reporting.

---

## Maintenance

Possible future vehicle events such as:

* service
* repair
* tires
* inspection
* insurance
* road tax

These are intentionally outside the current committed product scope.

---

# 21. Initial Invariants

The following are current Stage 0 assumptions.

They are expected to become executable test cases later.

### Household

A Vehicle belongs to exactly one Household.

### Membership

A Household Membership connects exactly one User and one Household.

### Role

A role belongs to Household Membership rather than User.

### ICE

An ICE vehicle may record Refuel Events.

### EV

An EV may record Charge Events.

### PHEV

A PHEV may record both Refuel and Charge Events.

### Refuel Quantity

Fuel added must be greater than zero.

### Charge Energy

Recorded energy added, when present, must not be negative.

### Odometer

Normal new vehicle events must not decrease the odometer relative to preceding valid vehicle observations.

### Currency

Every monetary transaction must identify its currency.

### Raw Data

Derived calculations must never replace the original event observations.

---

# 22. Questions Intentionally Left Open

Stage 0 should resolve these before implementation reaches the relevant feature.

### Household

* Can a user belong to multiple households in v1?
* Can ownership be transferred?
* Must every household always have at least one owner?

### Vehicle

* Exact representation of fuel configuration
* Exact representation of multi-fuel vehicles
* Whether HEV requires its own explicit powertrain type
* Whether tank/battery capacities are required or optional metadata

### Refueling

* Which price fields are authoritative when values disagree?
* Can historical odometer errors be corrected?
* How should deleted/corrected events affect later calculations?

### Charging

* How should charging losses be represented?
* Should charger-delivered kWh and battery-added kWh be distinct measurements?
* How should home charging prices be represented?

### Analytics

* Exact definition of estimated consumption
* Exact range-based efficiency methodology
* Calculation versioning strategy
* Rules for invalid or incomplete segments

These questions are not failures of the model.

They are explicitly tracked decisions that should be resolved when enough context exists.

---

# 23. Stage 0 Definition of Done for the Domain Model

This document can be considered sufficiently stable for implementation when:

* core terminology is unambiguous
* ownership relationships are defined
* supported vehicle event types are clear
* raw observations are distinguished from derived metrics
* measurement quality is defined
* important invariants are documented
* unresolved questions are explicitly visible rather than hidden inside code

The document is expected to evolve as OmniFleet encounters real-world data.

It is a domain contract, not an immutable specification.

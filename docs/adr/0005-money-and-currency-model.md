# ADR-0005: Money and Currency Model

* **Status:** Accepted
* **Date:** 2026-09-14
* **Stage:** Stage 0 — Domain & Engineering Foundation

## Context

OmniFleet tracks real-world monetary transactions related to vehicles.

Examples include:

* fuel purchases
* charging sessions
* future maintenance costs
* future insurance or tax records

These transactions may occur in different countries and currencies.

Typical examples:

```text
63.42 EUR
41,850 HUF
72.10 EUR
```

The monetary model must therefore support:

* exact monetary storage
* multiple currencies
* fuel and charging unit prices
* rounding differences
* derived effective prices
* future currency conversion
* historical reporting

The model must not assume that all transactions belong to a single currency.

It must also avoid binary floating-point behavior for money.

---

# Decision

OmniFleet will model money as:

```text
Money
- amount
- currency
```

Monetary values must use decimal arithmetic.

Binary floating-point types such as:

```text
float
double
f32
f64
```

must not be used as the authoritative representation of money.

Every monetary transaction preserves its original currency.

---

# Currency

Currency is part of the monetary value.

Examples:

```text
63.42 EUR
41850 HUF
```

The following are not equivalent:

```text
100 EUR
100 HUF
```

Therefore an amount without currency is incomplete.

Every persisted monetary transaction must identify its currency.

---

# Currency Representation

Currencies should use standardized ISO 4217-style currency codes where supported.

Examples:

```text
EUR
HUF
USD
CHF
```

The exact persistence representation is deferred.

Possible implementation:

```text
currency_code = "EUR"
```

rather than introducing application-specific numeric identifiers unnecessarily.

---

# Original Currency Is Preserved

A transaction is permanently associated with the currency in which it actually occurred.

Example:

```text
Refuel
total_amount = 41,850
currency = HUF
```

Future reporting may calculate a converted value.

However, it must not rewrite the original event into:

```text
105.32 EUR
```

as if the transaction originally occurred in EUR.

Conceptually:

```text
Original transaction
        ↓
Optional conversion
        ↓
Derived reporting value
```

The original remains authoritative.

---

# Monetary Precision

Money must use a fixed or arbitrary precision decimal representation.

Conceptually:

```text
Decimal
```

rather than floating point.

This avoids values such as:

```text
0.1 + 0.2 = 0.30000000000000004
```

appearing in financial calculations.

The exact Rust and PostgreSQL types will be decided in the persistence ADR.

A likely implementation direction is:

```text
PostgreSQL NUMERIC
+
Rust decimal type
```

but this ADR defines the requirement rather than the concrete library.

---

# Currency Minor Units

Currencies do not all use the same number of decimal places.

Examples:

```text
EUR
commonly 2 decimal places

HUF
normally used without fractional forints in consumer transactions
```

The domain must therefore not globally assume:

```text
money always has exactly 2 decimals
```

Internal decimal precision may be higher than the normal display precision.

Formatting rules belong to the currency/presentation layer.

---

# Refuel Monetary Model

A RefuelEvent may contain:

```text
liters
unit_price
total_amount
currency
```

These values have different meanings.

---

# Total Amount

`total_amount` represents the actual total amount paid for the transaction.

When explicitly supplied from:

* user input
* receipt
* confirmed import

it is considered the authoritative monetary total.

Example:

```text
42.17 L
1.579 EUR/L
66.58 EUR
```

Even if:

```text
42.17 × 1.579
```

does not produce exactly the stored total due to rounding rules, OmniFleet preserves:

```text
66.58 EUR
```

as the actual transaction amount.

---

# Unit Price

`unit_price` represents the observed price per quantity unit.

For fuel:

```text
EUR/L
HUF/L
```

For electricity:

```text
EUR/kWh
HUF/kWh
```

When directly supplied by the station, charger, receipt or user, it is an observed value.

When calculated from:

```text
total_amount / quantity
```

it becomes a derived effective unit price.

These two concepts must not silently be treated as identical.

---

# Quantity × Unit Price vs Total

The basic relationship is:

```text
quantity × unit_price ≈ total_amount
```

The approximation sign is intentional.

Differences may occur because of:

* rounding
* receipt precision
* tax calculation
* charging fees
* session fees
* discounts
* loyalty programs
* provider-specific billing rules

OmniFleet should therefore validate for plausibility, not mathematical identity.

---

# Example: Fuel Rounding

Given:

```text
liters = 42.17
unit_price = 1.579 EUR/L
```

The calculated value may contain more fractional precision than the actual receipt total.

If the receipt says:

```text
total_amount = 66.58 EUR
```

OmniFleet keeps:

```text
66.58 EUR
```

as the authoritative paid amount.

It must not silently overwrite it with a recomputed result.

---

# Missing Total Amount

If the user provides:

```text
quantity
+
unit_price
```

but no total amount, OmniFleet may derive:

```text
total_amount = quantity × unit_price
```

That value is system-derived.

It may be used as the effective transaction total until confirmed otherwise.

---

# Missing Unit Price

If the user provides:

```text
quantity
+
total_amount
```

OmniFleet may derive:

```text
effective_unit_price =
total_amount / quantity
```

The result should be treated as derived rather than an observed station price.

---

# Provenance

Where the distinction matters, the system should be capable of understanding whether a monetary value is:

```text
observed
derived
imported
confirmed
```

The exact persistence strategy for provenance is deferred.

The domain requirement is that derived values must not silently become equivalent to directly observed values.

---

# Charging Transactions

Charging-session pricing can be more complex than fuel pricing.

A charging provider may charge:

```text
energy fee
+
session fee
+
time fee
+
parking fee
-
discount
```

Therefore:

```text
total_amount
```

is more important than assuming:

```text
energy_kwh × unit_price
```

fully explains the invoice.

---

# Effective Charging Price

For analytics, OmniFleet may derive:

```text
effective_price_per_kwh =
total_amount / energy_kwh
```

when:

```text
energy_kwh > 0
```

This value answers:

> What did this charging session effectively cost per delivered kWh?

It does not necessarily represent the provider's advertised energy tariff.

---

# Charging Fee Breakdown

The initial product does not require detailed pricing components such as:

```text
energy_fee
time_fee
parking_fee
session_fee
subscription_discount
```

These may be introduced later if real usage demonstrates the need.

For the initial model:

```text
total_amount
```

is sufficient as the authoritative transaction cost.

---

# Free Transactions

A valid vehicle-energy transaction may cost:

```text
0
```

Examples:

* free workplace charging
* promotional charging
* complimentary charging
* free fuel obtained through an unusual but valid workflow

Therefore:

```text
total_amount >= 0
```

is valid.

The domain must not assume:

```text
total_amount > 0
```

for every event.

---

# Negative Monetary Values

Negative transaction totals are not valid for normal RefuelEvent or ChargeEvent creation.

Refunds, corrections and credits should eventually be modeled explicitly rather than disguised as negative refuel values.

Therefore for normal energy events:

```text
total_amount >= 0
```

The treatment of refunds is deferred.

---

# Multi-Currency Reporting

Without exchange-rate information, monetary totals from different currencies must not be combined.

Example:

```text
60 EUR
+
20,000 HUF
```

must not produce:

```text
Total = 20,060
```

or an arbitrary converted result.

Instead, reporting must initially group results by currency.

Example:

```text
Monthly spending:

EUR: 182.40
HUF: 54,800
```

---

# Household Currency

A household may later define a preferred reporting currency.

Example:

```text
EUR
```

This preference does not alter the original currency of any event.

It is only a reporting preference.

The preferred currency concept is intentionally deferred from the core transaction model.

---

# Exchange Rates

Cross-currency normalized reporting requires an explicit exchange-rate model.

Conceptually:

```text
ExchangeRate
- source_currency
- target_currency
- rate
- effective_date
- source
```

Example:

```text
HUF → EUR
rate = ...
effective_date = ...
```

Converted values are then derived from:

```text
original amount
+
exchange rate
```

---

# Historical Exchange Rates

If cross-currency reporting is introduced, OmniFleet should use a rate associated with the relevant historical date.

It must not silently convert all historical transactions using the current exchange rate.

Example:

```text
2026 transaction
```

should use an appropriate:

```text
2026 exchange rate
```

according to the eventual conversion policy.

The exact rate-selection policy is deferred.

---

# Exchange Rate Source

Possible future sources include:

```text
European Central Bank
external FX provider
manual user entry
```

The rate source should remain identifiable.

This is important for reproducibility.

A converted historical report should ideally be explainable as:

```text
40,000 HUF
×
stored/reference exchange rate
=
derived EUR amount
```

---

# Currency Conversion Is Derived Data

Converted amounts are never original transaction values.

Example:

```text
Original:
40,000 HUF
```

Derived:

```text
102.35 EUR
```

The second value may change if:

* conversion policy changes
* rate source changes
* rate data is corrected

The original:

```text
40,000 HUF
```

does not.

This follows ADR-0004.

---

# Cost Per Kilometer

Cost-per-distance metrics are derived.

Example:

```text
cost_per_km =
relevant cost / relevant distance
```

Currency remains part of the result.

Example:

```text
0.12 EUR/km
```

rather than:

```text
0.12 /km
```

For mixed-currency periods, cost/km must remain currency-specific unless explicit normalization exists.

---

# Weighted Average Unit Price

Average unit price must be quantity-weighted.

Incorrect:

```text
average of transaction unit prices
```

Correct for fuel:

```text
sum(total fuel cost)
/
sum(liters)
```

Correct for charging:

```text
sum(total charging cost)
/
sum(kWh)
```

provided the transactions belong to a compatible reporting scope.

---

# Example: Weighted Fuel Price

Given:

```text
Refuel A
20 L
1.50 EUR/L
30 EUR

Refuel B
50 L
1.60 EUR/L
80 EUR
```

Then:

```text
weighted average =
110 EUR / 70 L
≈ 1.5714 EUR/L
```

Not:

```text
(1.50 + 1.60) / 2
= 1.55 EUR/L
```

---

# Compatible Aggregation

Unit-price aggregation must only combine compatible units and currencies.

Valid:

```text
EUR/L Diesel
+
EUR/L Diesel
```

Potentially valid depending on report semantics:

```text
EUR/L Petrol
+
EUR/L LPG
```

but only if explicitly labeled as a combined fuel-cost metric.

Invalid:

```text
EUR/L
+
EUR/kWh
```

because the units are different.

Invalid without conversion:

```text
EUR/L
+
HUF/L
```

because the currencies differ.

---

# PHEV Cost Analytics

PHEVs may incur both:

```text
fuel cost
electricity cost
```

The native unit prices remain separate:

```text
EUR/L
EUR/kWh
```

However, monetary metrics may later combine them when they share the same currency.

Example:

```text
fuel spending
+
charging spending
=
combined vehicle energy spending
```

Likewise:

```text
combined energy cost / distance
```

may provide:

```text
EUR/km
```

without pretending liters and kWh are the same physical unit.

---

# Display Precision

Storage precision and display precision are separate concerns.

For example, OmniFleet may internally retain:

```text
1.579
```

while displaying:

```text
1.579 EUR/L
```

and may store or calculate:

```text
1.57142857...
```

while displaying:

```text
1.571 EUR/L
```

The UI should format values appropriately for the metric.

The persistence layer must not prematurely round values merely because the UI displays fewer decimal places.

---

# Rounding

Rounding should occur at clearly defined presentation or transaction boundaries.

OmniFleet should avoid repeated intermediate rounding.

For example:

```text
raw decimals
→ calculation
→ final display rounding
```

is preferred over:

```text
round input
→ round intermediate result
→ round again
→ aggregate rounded values
```

because repeated rounding introduces accumulated error.

---

# User Input

The UI may assist users by automatically calculating fields.

Example:

```text
liters = 40
unit_price = 1.60
```

UI displays:

```text
total = 64.00
```

The user may still replace the calculated total with the actual receipt value.

Example:

```text
actual receipt = 63.98
```

The explicit confirmed value wins.

---

# Monetary Validation

The system should reject structurally invalid values.

Examples:

```text
NaN
infinity
negative fuel price
invalid currency code
```

Plausibility warnings may eventually detect unusual values such as:

```text
fuel price = 100 EUR/L
```

but such checks should generally be warnings rather than hidden corrections.

Real-world data should not be silently changed merely because it appears unusual.

---

# Alternatives Considered

## Alternative A — Floating-Point Money

Example:

```text
f64
```

### Advantages

* easy to use
* native numeric operations
* common in general-purpose code

### Disadvantages

* binary representation errors
* unsuitable for authoritative financial values
* difficult exact equality and rounding behavior

### Decision

Rejected.

---

## Alternative B — Store Minor Units as Integer

Example:

```text
EUR cents
```

### Advantages

* exact arithmetic
* simple for currencies with fixed minor units

### Disadvantages

* awkward for unit prices requiring more precision
* not all currencies have identical minor-unit behavior
* energy/fuel unit prices often need three or more decimal places

### Decision

Not selected as the universal domain representation.

Integer minor units may still be useful in specific integrations.

---

## Alternative C — Decimal Money + Explicit Currency

Example:

```text
Money {
    amount: Decimal,
    currency: Currency
}
```

### Advantages

* precise monetary arithmetic
* supports high-precision unit prices
* works across multiple currencies
* clear domain semantics
* suitable for future conversion logic

### Disadvantages

* requires decimal-aware database and Rust types
* explicit rounding policies are still necessary

### Decision

Accepted.

---

# Invariants

## Currency

Every authoritative monetary amount has a currency.

---

## Precision

Authoritative monetary values use decimal rather than binary floating-point arithmetic.

---

## Original Transaction

Original transaction currency and amount are preserved.

---

## Total Amount

When explicitly confirmed, `total_amount` represents the actual paid transaction amount.

---

## Rounding

Minor rounding differences between:

```text
quantity × unit_price
```

and:

```text
total_amount
```

must not invalidate an otherwise legitimate transaction.

---

## Multi-Currency

Different currencies are not aggregated without explicit conversion data.

---

## Conversion

Converted monetary values are derived data.

---

## Free Transactions

A total amount of zero is valid.

---

## Negative Energy Transaction Cost

Normal RefuelEvents and ChargeEvents must not have negative transaction totals.

---

# Initial MVP Behavior

The first ICE vertical slice should support at least:

```text
liters
unit_price
total_amount
currency
```

The UI may derive one monetary field from the others.

The confirmed transaction total should be preserved.

Initial reporting should aggregate only within the same currency.

No automatic FX conversion is required for MVP.

Example:

```text
September

EUR:
182.30

HUF:
42,500
```

This is considered correct behavior.

---

# Validation Scenarios

## Scenario 1 — Normal EUR Refuel

```text
liters = 40
unit_price = 1.60 EUR/L
total_amount = 64.00 EUR

→ ACCEPT
```

---

## Scenario 2 — Receipt Rounding Difference

```text
liters = 42.17
unit_price = 1.579 EUR/L
total_amount = 66.58 EUR

→ ACCEPT
```

The total is preserved.

---

## Scenario 3 — Derived Total

```text
liters = 40
unit_price = 1.60
total_amount = missing
```

Then OmniFleet may derive:

```text
64.00 EUR
```

---

## Scenario 4 — Derived Unit Price

```text
liters = 40
total_amount = 64.00 EUR
unit_price = missing
```

Then OmniFleet may derive:

```text
1.60 EUR/L
```

---

## Scenario 5 — Free Charging

```text
energy = 11.2 kWh
total_amount = 0 EUR

→ ACCEPT
```

---

## Scenario 6 — Negative Cost

```text
total_amount = -12 EUR

→ REJECT
```

for a normal ChargeEvent or RefuelEvent.

Refund handling is a separate future domain concern.

---

## Scenario 7 — Mixed Currency Monthly Report

```text
Event A:
50 EUR

Event B:
20,000 HUF
```

Without FX data:

```text
Monthly:
50 EUR
20,000 HUF
```

No combined monetary total is produced.

---

## Scenario 8 — Weighted Average

```text
20 L @ 1.50 EUR/L
50 L @ 1.60 EUR/L
```

Then:

```text
weighted average ≈ 1.5714 EUR/L
```

rather than:

```text
1.55 EUR/L
```

---

## Scenario 9 — Complex Charging Price

```text
energy = 30 kWh
advertised energy price = 0.40 EUR/kWh

energy cost = 12 EUR
session fee = 2 EUR

total = 14 EUR
```

OmniFleet preserves:

```text
total_amount = 14 EUR
```

and may derive:

```text
effective price ≈ 0.4667 EUR/kWh
```

The effective price does not replace the provider tariff.

---

# Deferred Decisions

This ADR deliberately does not define:

* exact PostgreSQL numeric precision
* exact Rust decimal library
* rounding mode
* household preferred currency
* FX provider
* exchange-rate caching
* conversion-date policy
* refunds
* credits
* discounts as structured entities
* charging fee breakdown
* tax/VAT breakdown
* subscription pricing
* historical currency changes

These decisions can be made later without changing the fundamental monetary model.

---

# Consequences

## Positive

Money remains precise and explainable.

Original transaction values survive later conversion logic.

Multi-currency usage works naturally from the beginning.

Fuel and charging prices can use appropriate decimal precision.

Weighted averages and effective charging prices can be derived reliably.

---

## Negative

Reporting must explicitly handle currency boundaries.

Cross-currency totals are unavailable until an exchange-rate model exists.

Decimal arithmetic requires more deliberate implementation than primitive floating-point values.

Some transaction fields require provenance awareness when they are derived rather than entered.

These costs are accepted because financial accuracy is a core OmniFleet requirement.

---

# Result

OmniFleet represents money using decimal amounts with explicit currencies.

Original transaction amounts and currencies are preserved as source data.

Confirmed transaction totals represent the actual paid amount even when minor rounding differences exist.

Unit prices may be observed or derived.

Different currencies remain separate until explicit historical exchange-rate information is available.

Currency conversion, effective unit prices, weighted averages and cost-per-distance metrics are derived data.

This decision is accepted as part of the Stage 0 foundation.

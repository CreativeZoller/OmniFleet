# OmniFleet 🚗⚡

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Rust](https://img.shields.io/badge/Rust-orange.svg)](https://www.rust-lang.org/)
[![Vue.js](https://img.shields.io/badge/Vue%203-TypeScript-blue.svg)](https://vuejs.org/)

**OmniFleet** is an open-source, self-hosted vehicle tracking and analytics application designed for households, families, and multi-vehicle setups.

It supports **Internal Combustion Engine (ICE)**, **Electric (EV)**, and **Plug-in Hybrid (PHEV)** vehicles while focusing on accurate real-world running costs, consumption tracking, driving-efficiency analysis, and fast day-to-day data entry.

OmniFleet is built as an evolving product rather than around a fixed specification. Features are developed incrementally in usable vertical slices and adjusted based on real-world usage.

> **Project status:** Early development / pre-v1.0
> The roadmap below represents the current direction and may evolve as the application is used and validated with real vehicle data.

---

## ✨ Product Goals

OmniFleet aims to answer practical questions such as:

* How much does each vehicle actually cost to run?
* How is fuel or energy consumption changing over time?
* Am I driving more efficiently than before?
* How closely does real-world usage match the vehicle's onboard estimates?
* How much did we spend on a vehicle this month or year?
* Where do we usually refuel or charge?
* How do ICE, EV, and PHEV operating costs compare?
* How much does each kilometer actually cost?

The goal is not simply to store refuels and charging sessions, but to build a reliable long-term dataset around real vehicle usage.

---

## 🚙 Vehicle Support

OmniFleet is designed around a shared vehicle model capable of supporting different powertrains.

### ICE

Supports refueling events including:

* odometer
* liters
* unit price
* total cost
* fuel type
* full or partial refuel
* onboard estimated range
* onboard average consumption
* location
* currency
* receipt
* notes

### EV

Supports charging sessions including:

* odometer
* energy added in kWh
* start and end State of Charge
* charging duration (derived from start and end time)
* charging cost
* charger type
* charging power
* location
* currency
* receipt
* notes

### PHEV

Plug-in hybrid vehicles can contain both:

* refueling events
* charging events

This allows OmniFleet to analyze fuel and electricity costs independently while also providing combined vehicle operating-cost metrics where appropriate.

---

## 👨‍👩‍👧 Household & Multi-User Model

OmniFleet is designed around **households**, rather than assigning all permissions directly to individual users.

Conceptually:

```text
Household
├── Owner
├── Member
│
├── Vehicle A
├── Vehicle B
└── Vehicle C
```

Users have independent accounts and can belong to a household with a role such as:

* **Owner** — household administration, vehicle management, member management
* **Member** — vehicle usage, refuel/charge entry, reports and analytics

Vehicles belong to the household and activity records keep track of which user created them.

This model keeps family usage simple while leaving the architecture flexible enough for other small multi-user setups.

---

## 🧮 Consumption Analytics

OmniFleet distinguishes between different levels of measurement reliability instead of presenting every calculated value as equally precise.

### Verified Consumption

The most reliable ICE measurement uses the standard **full-to-full / tank-to-tank method**.

```text
Consumption (L/100km)
=
Fuel Used / Distance Traveled × 100
```

This produces a verified real-world measurement between compatible full refuels.

A verified ICE segment may include partial refuels between two full-tank anchors. Fuel added after the starting full refuel through the closing full refuel is counted. A partial refuel by itself is not treated as the amount of fuel consumed since the previous event.

---

### Estimated Consumption

Partial refuels cannot produce the same level of certainty because the exact amount of fuel remaining in the tank is generally unknown.

OmniFleet therefore keeps the original refuel data as the source of truth and may calculate estimated consumption metrics where enough information is available.

Estimated values are clearly distinguished from verified full-to-full measurements.

---

### Experimental Range-Based Efficiency

OmniFleet can optionally record the vehicle's onboard range estimate:

```text
range before refuel
range after refuel
```

Together with the next measurement point, this allows an experimental comparison between:

* how far the vehicle expected the available energy to take you
* how far you actually traveled before reaching the next measurement

A simplified segment ratio is:

```text
Efficiency Ratio =
(
  Range After Previous Refuel
  -
  Range Before Current Refuel
)
/
Distance Traveled
```

Interpretation:

```text
1.00   approximately matched the vehicle prediction

< 1.00 drove more efficiently than predicted

> 1.00 drove less efficiently than predicted
```

For example:

```text
0.83 = approximately 17% better than predicted
1.17 = approximately 17% worse than predicted
```

This metric is considered **experimental**.

Vehicle range estimators may change dynamically based on factors such as:

* recent driving behavior
* temperature
* traffic
* terrain
* HVAC usage
* manufacturer algorithms

For that reason, OmniFleet treats range-based analytics primarily as a trend and comparison tool rather than a replacement for verified consumption measurements.

---

## 💰 Cost Analytics

OmniFleet tracks vehicle operating costs alongside consumption.

Planned and supported metrics include:

* fuel cost per kilometer
* electricity cost per kilometer
* combined PHEV energy cost per kilometer
* monthly vehicle spending
* yearly vehicle spending
* weighted average fuel price
* weighted average electricity price
* distance traveled
* cost trends
* location-based spending

When averaging unit prices, OmniFleet uses **quantity-weighted averages** rather than averaging individual transaction prices.

For fuel:

```text
Weighted Average Price =
Total Fuel Cost / Total Liters
```

For electricity:

```text
Weighted Average Price =
Total Charging Cost / Total kWh
```

This avoids distorted averages when transaction quantities differ significantly.

---

## 💱 Multi-Currency

Transactions are stored in their **original currency**.

Examples:

```text
63.20 EUR
41,500 HUF
74.90 EUR
```

Initial reporting keeps currencies separate to avoid introducing incorrect historical conversions.

Future versions may support normalized cross-currency reporting using stored exchange-rate information such as:

* source currency
* target/base currency
* exchange rate
* rate date
* rate source

The original transaction value and currency will always remain preserved.

---

## 📊 Dashboard

The OmniFleet dashboard is designed around useful vehicle trends rather than raw data volume.

Planned dashboard elements include:

### KPI Cards

Examples:

* spending this month
* average consumption
* cost per kilometer
* distance driven
* average unit price
* efficiency trend

### Consumption Trend

Shows consumption measurements over time and can distinguish between:

* verified consumption
* estimated consumption
* onboard vehicle estimates
* experimental efficiency metrics

### Fuel / Energy Price Trend

Displays historical:

```text
EUR/L
HUF/L
EUR/kWh
HUF/kWh
```

depending on the selected vehicle and energy type.

### Monthly Spending

Vehicle costs aggregated by month.

### Recent Activity

A compact table containing recent refuels and charging sessions such as:

```text
Date
Vehicle
Location
Odometer
Liters / kWh
Unit Price
Total
Consumption
Receipt
```

### Date Ranges

Rather than relying on arbitrary time buckets, dashboard views are primarily based on event data with selectable ranges such as:

```text
3 months
6 months
12 months
All time
Custom range
```

Optional moving averages can be used to make long-term trends easier to read.

---

## 🧾 Receipt Management

Refuels and charging sessions can optionally contain supporting documents such as:

* receipt images
* invoices
* PDFs

Attachments are stored independently from the core transaction data.

Planned receipt-management capabilities include:

* secure uploads
* image and PDF support
* access control
* previews
* SHA-256 based duplicate detection
* attachment metadata
* storage abstraction for local or object storage

The manually confirmed transaction data remains the source of truth.

---

## 🔍 OCR-Assisted Entry

OCR is planned as an assistance feature rather than an automatic authority over stored data.

The intended workflow is:

```text
Take / upload receipt
        ↓
OCR extraction
        ↓
Suggested values
        ↓
User verification
        ↓
Save transaction
```

Potentially extracted fields include:

* date
* total amount
* liters
* kWh
* unit price
* fuel station
* charging provider

OCR results may also be compared with manually entered values to detect discrepancies.

OCR-generated values are never intended to silently overwrite confirmed vehicle data.

---

## 📱 Mobile-First Usage

Although OmniFleet is a web application, one of its primary usage scenarios is entering data immediately after refueling or charging.

The long-term mobile experience therefore targets:

* responsive UI
* fast refuel entry
* fast charging-session entry
* camera-based receipt capture
* remembered defaults
* recently used locations
* installable PWA
* offline drafts where practical

A common action such as recording a refuel should require as little interaction as possible.

---

## 🛠 Tech Stack

### Backend

* **Language:** [Rust](https://www.rust-lang.org/)
* **Web Framework:** [Axum](https://github.com/tokio-rs/axum)
* **Database Access:** [SQLx](https://github.com/launchbadge/sqlx)
* **Database:** PostgreSQL
* **Local Infrastructure:** Docker Compose
* **Authentication:** Session-based authentication
* **Password Hashing:** Argon2id
* **Logging / Tracing:** Rust tracing ecosystem

### Frontend

* **Framework:** [Vue 3](https://vuejs.org/)
* **Language:** TypeScript
* **API Style:** Composition API
* **UI Components:** [PrimeVue](https://primevue.org/)
* **State Management:** [Pinia](https://pinia.vuejs.org/)
* **Charts:** Chart.js via PrimeVue initially

The architecture intentionally avoids unnecessary infrastructure until real product requirements justify it.

---

## 🧱 Development Philosophy

OmniFleet is developed using **vertical product slices**.

Instead of building every database table first, then every API, and finally the UI, development aims to reach usable product states as early as possible.

The general loop is:

```text
Build a small usable feature
        ↓
Use it with real data
        ↓
Observe problems and friction
        ↓
Adjust the domain / UX
        ↓
Build the next slice
```

The roadmap is therefore a direction rather than a fixed specification.

Real-world usage may move features forward, postpone them, simplify them, or remove them entirely.

---

## 🗺 Project Roadmap

### Stage 0 — Domain & Calculation Foundation

Define the core rules before committing them deeply to the application.

* [x] Household and membership model
* [x] Vehicle and powertrain model
* [x] ICE refuel model
* [x] EV charge model
* [x] PHEV behavior
* [x] Currency model
* [x] Verified consumption calculations
* [x] Estimated consumption calculations
* [x] Experimental efficiency calculations
* [x] Calculation fixtures and unit-test scenarios
* [x] Architecture decision records / domain documentation

---

### v0.1 — Walking Skeleton

Establish the smallest complete application architecture.

* [ ] Rust workspace
* [ ] Axum API
* [ ] PostgreSQL
* [ ] SQLx migrations
* [ ] Docker Compose
* [ ] Vue 3 + TypeScript
* [ ] PrimeVue
* [ ] API communication
* [ ] Environment-based configuration
* [ ] Structured logging / tracing
* [ ] Error handling
* [ ] Health endpoint
* [ ] Initial CI workflow

Goal:

```text
Browser
   ↓
Vue
   ↓
Axum
   ↓
PostgreSQL
```

working end-to-end.

---

### v0.2 — ICE Refuel Vertical Slice

First genuinely usable OmniFleet workflow.

* [ ] User authentication
* [ ] Household creation
* [ ] Vehicle creation
* [ ] ICE vehicle support
* [ ] Add refuel
* [ ] Refuel validation
* [ ] Odometer validation
* [ ] Partial / full refuel support
* [ ] Range-before / range-after input
* [ ] Refuel history
* [ ] Edit / delete rules

At this point OmniFleet should already be usable for real refueling data.

---

### v0.3 — Consumption Engine

Turn raw refueling data into useful vehicle analytics.

* [ ] Full-to-full consumption
* [ ] Cost per kilometer
* [ ] Verified vs estimated measurements
* [ ] Vehicle onboard estimate comparison
* [ ] Experimental range-based efficiency
* [ ] Calculation quality metadata
* [ ] Calculation regression tests

---

### v0.4 — Dashboard

First complete MVP.

* [ ] KPI cards
* [ ] Consumption trend
* [ ] Unit-price trend
* [ ] Monthly spending
* [ ] Recent refuels table
* [ ] Date-range selection
* [ ] Weighted averages
* [ ] Vehicle filtering

**Milestone: first real MVP**

The application should now both collect data and answer useful questions about vehicle usage.

---

### v0.5 — Household & Multi-Vehicle

Expand the product from a single-user workflow into the intended household model.

* [ ] Household memberships
* [ ] Owner / Member roles
* [ ] User invitations
* [ ] Multiple vehicles
* [ ] Vehicle selector
* [ ] Per-vehicle permissions
* [ ] Activity attribution

---

### v0.6 — EV & PHEV

Introduce electric-energy tracking.

* [ ] EV vehicle support
* [ ] Charging sessions
* [ ] Start / end SoC
* [ ] kWh tracking
* [ ] Charging duration
* [ ] AC / DC charging
* [ ] Charger power
* [ ] kWh/100km analytics
* [ ] Electricity cost/km
* [ ] PHEV refuel + charge support
* [ ] Combined vehicle cost analytics

---

### v0.7 — Places

Reusable refueling and charging locations.

* [ ] Saved locations
* [ ] Fuel stations
* [ ] Charging stations
* [ ] Home charging
* [ ] Recent locations
* [ ] Location autocomplete
* [ ] Location-based analytics

---

### v0.8 — Receipt Management

Add supporting documents to vehicle transactions.

* [ ] Image upload
* [ ] PDF upload
* [ ] Secure attachment access
* [ ] Attachment previews
* [ ] SHA-256 duplicate detection
* [ ] Storage abstraction
* [ ] Local development storage
* [ ] Object-storage support

---

### v0.9 — Reporting & Export

Expand long-term analytics.

* [ ] Advanced vehicle reports
* [ ] Monthly / yearly comparisons
* [ ] Cost per vehicle
* [ ] Distance analytics
* [ ] Fuel / charging location analytics
* [ ] CSV export
* [ ] Cross-vehicle cost comparison

Cross-currency normalization may be introduced here once the exchange-rate model is properly defined.

---

### v0.10 — OCR-Assisted Entry

Reduce manual data-entry effort.

* [ ] Receipt OCR
* [ ] Suggested transaction values
* [ ] User confirmation flow
* [ ] OCR vs entered-value comparison
* [ ] Discrepancy detection
* [ ] OCR confidence metadata

---

### v0.11 — Mobile Experience

Optimize OmniFleet for real-world use at fuel and charging stations.

* [ ] PWA support
* [ ] Installable application
* [ ] Camera integration
* [ ] Quick-entry workflows
* [ ] Remembered defaults
* [ ] Offline drafts
* [ ] Mobile UX optimization

---

### v0.12 — Production Hardening

Prepare the project for a stable release.

* [ ] Backup strategy
* [ ] Restore testing
* [ ] Security headers
* [ ] CSRF protection
* [ ] Rate limiting
* [ ] Session hardening
* [ ] Audit logging
* [ ] Attachment limits
* [ ] Migration safety
* [ ] Structured metrics
* [ ] Error monitoring
* [ ] Calculation regression protection
* [ ] Documentation review

---

### v1.0 — Stable Release

The first stable OmniFleet release.

Expected core capabilities:

* household accounts
* multiple vehicles
* ICE / EV / PHEV tracking
* refuel and charging history
* consumption analytics
* cost analytics
* dashboard
* receipts
* reporting
* mobile-friendly workflows
* production-safe deployment

---

## 🔮 Possible Future Directions

These are deliberately **not part of the committed roadmap**.

They may be explored only when real usage demonstrates a clear need.

Possible areas include:

* automatic exchange-rate integration
* fuel-price integrations
* charging-network integrations
* maintenance tracking
* service history
* tires
* insurance
* taxes
* budgets
* monthly spending alerts
* vehicle total cost of ownership
* GPS-assisted station detection
* OBD-II imports
* manufacturer APIs
* email receipt imports
* richer EV analytics
* vehicle efficiency comparisons

OmniFleet should grow from actual usage rather than from accumulating features for their own sake.

---

## 🚀 Getting Started

OmniFleet is currently in early development.

Setup instructions will evolve together with the first development stages.

Expected local prerequisites:

* Rust
* Node.js
* Docker / Docker Compose
* PostgreSQL through the provided Docker environment

Clone the repository:

```bash
git clone https://github.com/your-username/omnifleet.git
cd omnifleet
```

Environment configuration and development commands will be documented as the v0.1 walking skeleton is implemented.

---

## 📚 Documentation

Architecture and domain decisions are documented alongside the codebase.

Stage 0 documentation:

```text
docs/
├── domain-model.md
├── calculations.md
├── scenarios.md
├── stage-0-handoff.md
└── adr/
    ├── 0001-household-and-membership-model.md
    ├── 0002-vehicle-and-powertrain-model.md
    ├── 0003-refuel-and-charge-event-model.md
    ├── 0004-raw-vs-derived-data.md
    ├── 0005-money-and-currency-model.md
    ├── 0006-consumption-measurement-quality.md
    ├── 0007-application-architecture.md
    ├── 0008-api-and-authentication-strategy.md
    └── 0009-data-persistence-and-migration-strategy.md
```

The calculation documentation will explicitly distinguish between:

* standard / established formulas
* OmniFleet-derived estimates
* experimental metrics
* assumptions and known limitations

---

## 🤝 Contributing

OmniFleet is currently evolving rapidly.

Contributions, discussions, test data, bug reports, and ideas are welcome, but major architectural or domain changes should be discussed before implementation.

The project favors:

* simple architecture
* explicit domain rules
* testable calculations
* maintainable code
* incremental delivery
* real-world validation
* avoiding premature optimization

---

## 📄 License

OmniFleet is licensed under the [MIT License](LICENSE).

You are free to use, modify, and distribute the software under the terms of the license.

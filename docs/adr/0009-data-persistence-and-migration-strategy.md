# ADR-0009: Data Persistence and Migration Strategy

* **Status:** Accepted
* **Date:** 2026-09-14
* **Stage:** Stage 0 — Domain & Engineering Foundation

## Context

OmniFleet stores long-lived vehicle history.

The dataset may contain:

* users
* households
* memberships
* vehicles
* refuels
* charging sessions
* monetary transactions
* future locations
* future attachments
* future calculated/reporting data

Much of this information remains valuable for years.

Historical vehicle data also participates in calculations where one incorrect or missing event may affect later analytics.

The persistence strategy therefore needs to prioritize:

* data integrity
* explicit relationships
* reproducible migrations
* precise numeric storage
* safe historical evolution
* understandable SQL
* database-level protection for simple invariants

OmniFleet does not need a database abstraction designed to support multiple database engines.

PostgreSQL is the chosen primary datastore.

---

# Decision

OmniFleet will use:

```text
PostgreSQL
+
SQLx
+
versioned SQL migrations
```

as its primary persistence model.

PostgreSQL is the authoritative persistent datastore for application data.

SQL will remain an explicit part of the application rather than being hidden behind a heavyweight ORM.

---

# PostgreSQL

OmniFleet will target PostgreSQL as its relational database.

The application will make reasonable use of PostgreSQL capabilities where they improve:

* integrity
* query clarity
* data types
* constraints
* indexing
* transactional behavior

The project does not aim to remain compatible with:

```text
MySQL
SQLite
MariaDB
```

through a lowest-common-denominator persistence abstraction.

If another database is ever supported, that should be driven by a concrete product requirement.

---

# SQLx

The Rust backend will use SQLx for database access.

The preferred direction is:

```text
explicit SQL
+
typed Rust boundaries
+
compile-time/query validation where practical
```

rather than an ORM that automatically maps the entire domain model to mutable database objects.

Database rows, domain entities and API DTOs are allowed to differ.

Conceptually:

```text
Database Row
     ↓
Mapping
     ↓
Domain Model
     ↓
Application/API
```

A table is not automatically a domain object.

---

# Database Schema Does Not Define the Domain

The domain decisions from earlier ADRs remain authoritative.

For example, ADR-0002 defines:

```text
Vehicle
├── Powertrain
├── Fuel Configuration
└── Electric Configuration
```

The final SQL implementation may use:

```text
one table
multiple tables
constraints
```

depending on what produces the clearest persistence model.

The database structure must preserve the domain meaning rather than redefine it for convenience.

---

# Primary Keys

Persistent entities should use opaque application identifiers.

The initial direction is UUID-based identifiers.

Conceptually:

```text
household_id
vehicle_id
refuel_id
```

should not expose business meaning.

Identifiers must not encode:

* household membership
* vehicle type
* creation date as business meaning
* authorization information

Identifiers are identity only.

---

# Why Opaque IDs

Sequential IDs are technically valid, but opaque IDs provide several useful properties:

* identifiers can be created outside a central sequence
* accidental enumeration becomes less convenient
* imported or distributed data can be merged more easily later
* internal row ordering is not confused with domain ordering

This does **not** replace authorization.

Knowing or guessing an identifier must never grant access.

ADR-0008 authorization rules still apply.

---

# Foreign Keys

Relationships defined by the domain should normally be protected with foreign keys.

Examples:

```text
household_memberships.household_id
→ households.id

household_memberships.user_id
→ users.id

vehicles.household_id
→ households.id

refuels.vehicle_id
→ vehicles.id
```

Foreign keys protect relational integrity even when application code contains a bug.

---

# Referential Actions

`ON DELETE CASCADE` must not be used automatically.

Cascade behavior must reflect domain semantics.

For example, deleting a Household should not casually erase years of:

```text
vehicles
refuels
charges
receipts
```

merely because a foreign key happens to exist.

Deletion behavior must be chosen explicitly for each relationship.

---

# Uniqueness Constraints

Simple uniqueness invariants should be enforced by PostgreSQL where practical.

Example from ADR-0001:

```text
one User
may have at most one active membership
in the same Household
```

Conceptually:

```text
UNIQUE (household_id, user_id)
```

or an equivalent constraint depending on the final membership lifecycle model.

Other examples may include:

```text
users.email
```

where account identity requires uniqueness.

---

# CHECK Constraints

Simple local invariants should also be protected by database constraints where practical.

Examples may include:

```text
liters > 0

energy_kwh > 0

start_soc_pct BETWEEN 0 AND 100

end_soc_pct BETWEEN 0 AND 100

total_amount >= 0
```

The application should still validate these rules before persistence so users receive meaningful domain errors.

Database constraints provide a second line of protection.

---

# What Does Not Belong in a CHECK Constraint

Rules involving historical state or multiple entities generally belong to application/domain logic.

Example:

```text
new odometer >= previous vehicle odometer
```

depends on other records.

Likewise:

```text
Household must always have at least one Owner
```

may require transactional application logic.

The database should not be forced into complex triggers merely to move all business logic into SQL.

---

# Database Triggers

Database triggers are not part of the default application architecture.

They may be introduced where they clearly solve a persistence-level problem, but business behavior should not become hidden across large amounts of implicit trigger logic.

Preferred default:

```text
Application/domain logic
+
transactions
+
constraints
```

rather than:

```text
application action
→ unknown database trigger chain
```

Any significant trigger-based behavior should receive explicit documentation.

---

# Transactions

Operations affecting multiple records atomically must use database transactions.

Example:

```text
Create Household
+
Create initial Owner Membership
```

must result in either:

```text
both succeed
```

or:

```text
neither exists
```

Conceptually:

```sql
BEGIN;

-- create household
-- create owner membership

COMMIT;
```

Failures result in rollback.

---

# Concurrency

Application validation alone is not sufficient where concurrent requests can violate an invariant.

Example:

Two requests simultaneously attempt to remove two different Owners from a Household containing exactly two Owners.

Both requests could independently observe:

```text
2 owners
```

and both decide removal is valid.

Persistence/application logic must therefore account for concurrency when maintaining critical invariants.

Possible mechanisms include:

* appropriate transaction isolation
* row locking
* atomic conditional operations
* database constraints where possible

The exact implementation is deferred until the relevant feature is built.

---

# Timestamp Storage

Persisted real-world timestamps will use timezone-aware timestamps.

The preferred PostgreSQL representation is conceptually:

```text
TIMESTAMPTZ
```

for event instants such as:

```text
occurred_at
recorded_at
created_at
updated_at
```

The application should treat stored timestamps as absolute instants.

---

# UTC and Presentation Time

Persistence should not depend on the server machine's local timezone.

Conceptually:

```text
user enters local event time
        ↓
time + offset / timezone context
        ↓
absolute instant
        ↓
database
```

The frontend may display that instant in the user's appropriate timezone.

The database value should not silently change meaning because OmniFleet is deployed in a different country.

---

# Event Time vs Creation Time

As established by ADR-0003:

```text
occurred_at
```

and:

```text
created_at / recorded_at
```

represent different facts.

Both may therefore exist in persistence.

Database insertion order must never substitute for event chronology.

Vehicle history is based primarily on real-world event time and relevant odometer information.

---

# Dates Without Times

Some future domain values may genuinely represent dates rather than instants.

Examples could include:

```text
vehicle first registration date
insurance validity date
```

Such values should use date semantics rather than fake midnight timestamps.

The system should not use one timestamp type for every temporal concept merely for convenience.

---

# Money

ADR-0005 requires decimal monetary arithmetic.

The database will therefore use PostgreSQL decimal/numeric storage for authoritative monetary values.

Conceptually:

```text
NUMERIC
```

rather than:

```text
REAL
DOUBLE PRECISION
```

The exact precision and scale will be defined when the schema is implemented.

---

# Why Precision Is Deferred

Different values have different precision requirements.

Examples:

```text
total amount
66.58 EUR

fuel unit price
1.579 EUR/L

derived effective price
1.571428...
```

One arbitrary global scale may either:

* waste precision
* truncate useful values
* create unnecessary restrictions

Therefore actual column precision will be chosen per semantic value during schema implementation.

The important Stage 0 decision is:

> authoritative financial arithmetic is decimal.

---

# Measurement Values

Vehicle measurements such as:

```text
liters
kWh
odometer
range
SoC
```

should use types appropriate to their physical meaning.

Examples:

```text
odometer
→ integer or precisely defined distance unit

SoC percentage
→ bounded numeric value

liters / kWh
→ decimal measurement
```

The application should avoid floating-point storage where exact decimal user-entered values matter.

---

# Units

Database column names should make physical units explicit where practical.

Preferred examples:

```text
odometer_km
range_before_km
energy_kwh
tank_capacity_l
power_kw
```

rather than:

```text
odometer
range
energy
capacity
power
```

This reduces ambiguity in SQL and migrations.

The domain layer may use stronger typed units later.

---

# Enumerations

Domain categories include concepts such as:

```text
ICE
EV
PHEV

Owner
Member

Petrol
Diesel
LPG
CNG
```

Persistence must represent them explicitly.

However, the project will not automatically use PostgreSQL native ENUM types for every domain enum.

---

# Enum Storage Strategy

Initial preference:

```text
textual values
+
CHECK constraints where appropriate
```

or another migration-friendly representation.

Example conceptually:

```text
powertrain TEXT
CHECK (powertrain IN ('ice', 'ev', 'phev'))
```

### Rationale

Domain categories may evolve during pre-v1 development.

Changing PostgreSQL native ENUMs can create more migration ceremony than simple constrained textual values.

Once a category proves stable, native database enum types may be reconsidered if they provide clear value.

---

# Booleans

Boolean fields should only represent genuinely binary concepts.

Example:

```text
is_full_tank
```

is currently intentionally binary.

However, if future real-world use shows a third meaningful state such as:

```text
unknown
```

the domain should evolve rather than abusing:

```text
false
```

to mean both:

```text
partial
```

and:

```text
unknown
```

Database convenience must not erase semantic differences.

---

# Nullability

`NULL` should mean:

```text
value is not present / not known / not applicable
```

and not act as an undocumented catch-all.

Fields required by the domain should normally be:

```text
NOT NULL
```

Optional observations may remain nullable.

Example:

```text
range_before_km
```

may legitimately be absent.

---

# Defaults

Database defaults should be used for values the database can safely determine.

Examples:

```text
created_at = now()
```

may be reasonable.

Defaults should not silently invent domain observations.

For example, missing:

```text
is_full_tank
```

should not automatically become:

```text
false
```

if absence and partial refueling have different meanings.

---

# Raw Data Preservation

ADR-0004 defines confirmed raw observations as the source of truth.

The persistence model must preserve those observations directly.

Derived values such as:

```text
distance_since_previous
consumption_l100
cost_per_km
efficiency_ratio
```

should not become authoritative fields on core event records.

---

# Derived Data

Derived values may initially be calculated on read.

Example:

```text
load refuel history
        ↓
calculate segments
        ↓
return analytics
```

At the expected early scale, this is sufficient.

---

# Persisted Derived Data

Future performance needs may justify:

```text
materialized views
calculation tables
cached summaries
```

If introduced, these values must remain:

```text
rebuildable
```

from authoritative source data.

Deleting the cache and rebuilding it must be conceptually safe.

---

# Database Views

Views may be used where they make reporting clearer.

For example, a future unified cost timeline may conceptually combine:

```text
refuels
UNION
charges
```

A view is acceptable if it remains a derived reporting representation.

It must not replace the underlying domain tables.

---

# Indexes

Indexes should be created based on expected access patterns.

Likely examples include:

```text
vehicles.household_id

refuels.vehicle_id
refuels.occurred_at

charges.vehicle_id
charges.started_at
charges.ended_at

memberships.user_id
memberships.household_id
```

The canonical ChargeEvent calendar-time column is deferred; see ADR-0003.

Composite indexes may later support queries such as:

```text
(vehicle_id, occurred_at)
```

The final index set should follow actual query patterns rather than speculative indexing of every column.

---

# Odometer Indexing

Vehicle-history calculations may frequently order events by:

```text
vehicle_id
+
odometer
```

or:

```text
vehicle_id
+
occurred_at
```

Indexes should be introduced when those queries exist.

Stage 0 does not require finalizing all indexes before real SQL exists.

---

# Migrations

Schema evolution will use version-controlled SQL migrations.

Conceptually:

```text
migrations/
├── 0001_create_users.sql
├── 0002_create_households.sql
├── 0003_create_memberships.sql
└── ...
```

Exact SQLx naming conventions will follow the tooling used during implementation.

---

# Migration History Is Immutable

Once a migration has been applied to a shared or production database, it must not be silently rewritten.

Incorrect schema evolution is fixed with a new migration.

Preferred:

```text
001 create table
002 add constraint
003 correct column
```

Not:

```text
edit migration 001 after production has already applied it
```

This keeps environments reproducible.

---

# Development Reset

During very early development, local disposable databases may be reset while schema design is still changing rapidly.

Once a migration participates in shared persistent environments, its history should be treated as immutable.

This gives early-stage flexibility without normalizing dangerous production migration habits.

---

# Forward Migrations

The primary migration strategy is forward evolution.

Production incidents should not assume every schema change can be safely reversed automatically.

Some changes are inherently destructive.

Example:

```text
DROP COLUMN
```

may lose information that a down migration cannot reconstruct.

Therefore rollback strategy should include:

* backups
* forward fixes
* deliberate operational procedures

rather than blindly relying on reversible migration scripts.

---

# Destructive Migrations

Destructive schema changes require extra care.

Examples:

```text
DROP COLUMN
DROP TABLE
change column semantics
remove supported enum value
```

Preferred evolution may involve:

```text
add new field
        ↓
migrate/backfill data
        ↓
update application
        ↓
verify
        ↓
remove obsolete field later
```

rather than one destructive deployment step.

---

# Data Backfills

Schema migration and data migration are related but not identical.

Large or logically complex transformations may require explicit backfill logic.

The project should avoid hiding major domain transformations in a mysterious single SQL statement merely to call them migrations.

For OmniFleet's expected initial scale, simple transactional backfills are likely sufficient.

---

# Schema Compatibility During Deployment

The initial self-hosted deployment may update application and database together.

Therefore elaborate zero-downtime multi-version schema compatibility is not required from v0.1.

However, migration design should avoid unnecessary destructive behavior that makes safe deployment difficult.

If OmniFleet later supports large hosted installations, this policy can evolve.

---

# Startup Migration Behavior

Whether production migrations run automatically when the API starts or through an explicit deployment step is intentionally deferred.

Both approaches have tradeoffs.

Whatever approach is selected must ensure:

```text
application binary
and
database schema
```

remain compatible.

---

# Backups

Database backups are required before OmniFleet can be considered production-ready.

The production-hardening stage must define:

```text
backup
restore
restore testing
```

A backup policy that has never been restored successfully is not considered sufficient.

Detailed backup tooling is outside Stage 0.

---

# Soft Delete

OmniFleet will **not use a universal `deleted_at` column on every table**.

Deletion behavior is domain-specific.

Different entities have different lifecycle semantics.

---

# Vehicles

Vehicles should normally be:

```text
Archived
```

rather than deleted after historical activity exists.

This preserves:

* refuels
* charges
* analytics
* reports

The persistence model should support the Vehicle lifecycle defined by the domain.

---

# Household Membership

Removing a membership must preserve historical activity attribution.

The exact representation may use concepts such as:

```text
inactive membership
left_at
membership history
```

and will be decided when membership persistence is implemented.

Historical records must not disappear because membership ended.

---

# Refuel and Charge Deletion

A global decision to soft-delete all vehicle events is not made in Stage 0.

Event correction/deletion must respect ADR-0004:

```text
change source event
        ↓
invalidate dependent analytics
```

The first implementation may support controlled event deletion through application logic.

If audit history becomes a requirement, event revision or soft-delete behavior can be added deliberately.

---

# Users

Account lifecycle is separate from historical vehicle attribution.

User deletion must not cascade into deleting household vehicle history.

Privacy/anonymization behavior is deferred to a dedicated future decision if required.

---

# No Generic Cascade Deletion

The following conceptual behavior is explicitly rejected:

```text
delete user
→ delete memberships
→ delete vehicles
→ delete refuels
→ delete charges
```

Historical vehicle data must not disappear through accidental relational cascading.

---

# Attachments

Future attachment records will store metadata in PostgreSQL.

Binary content should normally live in the selected storage backend.

Conceptually:

```text
attachments
- id
- storage_key
- mime_type
- size
- checksum
- ownership reference
```

The database remains responsible for metadata and authorization relationships.

---

# JSON / JSONB

`JSONB` may be useful for data that is genuinely:

* provider-specific
* semi-structured
* not part of the stable core domain

Examples may eventually include:

```text
raw OCR output
external provider metadata
```

However, JSONB must not become an escape hatch for avoiding schema design.

Core fields such as:

```text
odometer
liters
currency
powertrain
```

belong in typed relational columns.

---

# Database as Integrity Layer

PostgreSQL is not merely a serialization target.

The database should actively protect simple structural truths through:

```text
foreign keys
NOT NULL
UNIQUE
CHECK
transactions
```

This complements the Rust domain layer.

---

# Application as Domain Authority

The database does not need to understand every business concept.

Complex rules such as:

```text
is this a valid verified consumption segment?
```

belong to the domain/application layer.

The intended split is:

```text
Database
→ structural integrity

Domain/Application
→ business meaning
```

---

# Testing Migrations

Migrations should be exercised against a real PostgreSQL instance in development/CI.

Tests should eventually verify that:

```text
empty database
+
all migrations
=
valid current schema
```

Important constraints should also be covered by integration tests.

---

# Test Database

Integration tests that depend on PostgreSQL should run against PostgreSQL rather than substituting SQLite.

Using a different database engine for tests could hide differences in:

* SQL behavior
* constraints
* numeric semantics
* transaction behavior
* PostgreSQL features

The test environment should resemble the real persistence technology.

---

# Seed Data

Seed data may be provided for local development.

It must remain clearly separate from schema migrations.

Migrations define:

```text
required schema evolution
```

Seeds define:

```text
optional development/test data
```

Production correctness must not depend on development seed scripts.

---

# Alternatives Considered

## Alternative A — SQLite First

### Advantages

* extremely easy local setup
* embedded database
* fast tests

### Disadvantages

* different SQL behavior
* weaker parity with production
* migration differences
* encourages lowest-common-denominator schema design

### Decision

Rejected.

Docker Compose already makes local PostgreSQL practical.

---

## Alternative B — Heavy ORM

### Advantages

* convenient CRUD
* automatic mappings
* less handwritten SQL

### Disadvantages

* may hide important SQL behavior
* domain model can become persistence-shaped
* complex reporting often returns to custom SQL anyway

### Decision

Rejected for the initial architecture.

SQLx provides the desired balance of explicit SQL and Rust typing.

---

## Alternative C — PostgreSQL + SQLx

### Advantages

* strong relational integrity
* precise numeric support
* explicit SQL
* mature transaction model
* strong reporting capabilities
* suitable for long-term vehicle history

### Disadvantages

* requires running PostgreSQL in development
* developers must understand SQL
* persistence is PostgreSQL-specific

### Decision

Accepted.

---

# Invariants

## Primary Persistence

PostgreSQL is the authoritative persistent datastore.

---

## Migration History

Applied shared/production migrations are immutable.

---

## Money

Authoritative money uses decimal database storage.

---

## Time

Event instants use timezone-aware persistence.

---

## Relationships

Core relationships use foreign-key integrity where practical.

---

## Simple Domain Constraints

Simple local invariants are reinforced with database constraints.

---

## Historical Safety

Deletion must not accidentally cascade through historical vehicle data.

---

## Derived Data

Cached derived values are rebuildable and never override raw source observations.

---

## Database Portability

Cross-database portability is not an architectural requirement.

---

# Initial v0.1 Persistence Scope

The first walking skeleton does **not** need the final complete OmniFleet schema.

Only tables required by the current vertical slice should be introduced.

The initial sequence may approximately grow as:

```text
users
        ↓
sessions
        ↓
households
        ↓
household_memberships
        ↓
vehicles
        ↓
refuels
```

Charge, places, attachments and reporting persistence should be introduced when their vertical slices are built.

This follows ADR-0007.

---

# Why We Do Not Create Every Future Table Now

The Stage 0 domain model defines future concepts, but database tables should follow implemented product slices.

Creating every planned table before real usage would:

* freeze assumptions too early
* produce unused schema
* increase migration burden
* encourage speculative fields
* make later domain changes harder

The domain is intentionally designed ahead.

The concrete persistence schema evolves with the product.

---

# Validation Scenarios

## Scenario 1 — Duplicate Membership

```text
User A
already belongs to
Household X
```

Attempt another identical membership.

Result:

```text
database/application
→ reject duplicate
```

---

## Scenario 2 — Orphan Vehicle

Attempt to create:

```text
Vehicle
household_id = nonexistent household
```

Result:

```text
foreign key
→ reject
```

---

## Scenario 3 — Invalid Fuel Quantity

Attempt:

```text
liters = -20
```

Result:

```text
application validation
→ reject
```

and database constraints should prevent invalid persistence where appropriate.

---

## Scenario 4 — Decimal Money

Store:

```text
unit_price = 1.579
total_amount = 66.58
```

The database preserves decimal values without binary floating-point artifacts.

---

## Scenario 5 — Historical Time

An event occurs in one timezone but OmniFleet is hosted in another.

The persisted instant represents the same real-world moment regardless of server timezone.

---

## Scenario 6 — Migration Evolution

Production has applied:

```text
migration 001
migration 002
```

A problem is discovered in the schema.

Correct action:

```text
create migration 003
```

not:

```text
rewrite migration 001
```

---

## Scenario 7 — Vehicle Archive

A Vehicle has years of RefuelEvents.

User no longer uses the Vehicle.

Result:

```text
Vehicle archived
Historical events preserved
Reports remain possible
```

---

## Scenario 8 — Cache Corruption

A future cached consumption value disagrees with calculations from source events.

Result:

```text
cache can be discarded
and rebuilt
```

Raw events remain authoritative.

---

# Deferred Decisions

This ADR deliberately does not define:

* exact initial SQL schema
* exact UUID generation strategy
* exact NUMERIC precision/scale per column
* exact SQLx migration filenames
* exact session schema
* exact membership lifecycle fields
* ChargeEvent canonical calendar time (`started_at` vs `ended_at`)
* event soft-delete strategy
* audit-table design
* backup tooling
* automatic migration-at-startup policy
* production connection-pool sizing
* database hosting provider
* materialized-view strategy
* calculation cache schema

These decisions can be made during the relevant implementation stages without changing the persistence principles defined here.

---

# Consequences

## Positive

OmniFleet gets a strong relational integrity model.

Money and timestamps can be stored using appropriate database semantics.

Schema evolution remains reproducible.

Explicit SQL keeps persistence behavior visible.

Historical vehicle data receives protection from accidental cascading deletion.

The schema can evolve alongside vertical product slices rather than freezing speculative future features.

---

## Negative

The project intentionally becomes PostgreSQL-specific.

Developers need working SQL knowledge.

Careful migration discipline is required once persistent environments exist.

Some business invariants still require application-level transactional logic.

Historical corrections and deletion workflows will require deliberate implementation.

These costs are accepted because they provide predictable, maintainable persistence for OmniFleet's long-lived data.

---

# Result

OmniFleet will use:

```text
PostgreSQL
+
SQLx
+
version-controlled SQL migrations
```

as its persistence foundation.

PostgreSQL protects structural integrity through foreign keys, uniqueness constraints, checks and transactions where appropriate.

The Rust application remains responsible for higher-level domain rules.

Money uses decimal persistence.

Event instants use timezone-aware timestamps.

Applied migrations are treated as immutable history.

Deletion and archival behavior are domain-specific rather than implemented through a universal soft-delete mechanism.

Only persistence needed by the current vertical slice should be introduced; Stage 0 does not justify creating the entire future database schema in advance.

This decision is accepted as part of the Stage 0 foundation.

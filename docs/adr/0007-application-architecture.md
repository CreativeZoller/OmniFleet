# ADR-0007: Application Architecture

* **Status:** Accepted
* **Date:** 2026-09-14
* **Stage:** Stage 0 — Domain & Engineering Foundation

## Context

OmniFleet is a self-hosted web application with a moderately rich vehicle-tracking domain.

The application needs to support:

* household accounts
* multiple vehicles
* ICE, EV and PHEV data
* refuel and charge events
* consumption calculations
* reporting
* receipts
* future OCR
* future external integrations

The system is expected to begin with a small number of users and relatively small datasets.

Even many years of vehicle history for a household will normally result in thousands rather than millions of vehicle events.

The architecture therefore needs to optimize for:

* correctness
* maintainability
* understandable domain boundaries
* testability
* incremental development
* simple self-hosted deployment

rather than distributed-system scale.

---

# Decision

OmniFleet will begin as a **modular monolith** consisting of:

```text
Browser
   ↓
Vue Application
   ↓
HTTP JSON API
   ↓
Rust / Axum Application
   ↓
PostgreSQL
```

The backend is deployed as one application.

The frontend is one Vue application.

PostgreSQL is the primary persistent datastore.

Internal modules remain separated by domain responsibility, but they are not deployed as independent services.

---

# Why a Modular Monolith

The application contains multiple domain areas:

```text
Households
Vehicles
Refuels
Charges
Analytics
Receipts
Reporting
```

These concepts should remain structurally separated.

However, separating them into network services would introduce:

* service discovery
* distributed authorization
* network failure modes
* distributed transactions
* duplicated infrastructure
* deployment complexity
* tracing across services
* API versioning between internal components

without solving a current product problem.

A modular monolith keeps domain boundaries explicit while allowing direct in-process communication where appropriate.

---

# High-Level Architecture

Conceptually:

```text
┌───────────────────────────────┐
│           Vue Web             │
│                               │
│  views / features / UI state  │
└───────────────┬───────────────┘
                │
             HTTP/JSON
                │
┌───────────────▼───────────────┐
│         Rust / Axum           │
│                               │
│  application / domain logic   │
│                               │
│  household                    │
│  vehicle                      │
│  refuel                       │
│  charge                       │
│  analytics                    │
│  ...                          │
└───────────────┬───────────────┘
                │
               SQL
                │
┌───────────────▼───────────────┐
│          PostgreSQL           │
└───────────────────────────────┘
```

This is one system with explicit internal boundaries.

---

# Backend Responsibilities

The Rust backend is responsible for authoritative application behavior.

This includes:

* authentication
* authorization
* domain validation
* persistence
* consumption calculations
* monetary rules
* household boundaries
* vehicle compatibility
* API behavior

The frontend must not become the authoritative location for business rules.

For example:

```text
EV
→ RefuelEvent
→ invalid
```

must be rejected by the backend even if the frontend already prevents the user from submitting such a form.

---

# Frontend Responsibilities

The Vue frontend is responsible for:

* presentation
* navigation
* user interaction
* form state
* client-side convenience validation
* dashboard rendering
* API communication
* responsive/mobile behavior

The frontend may duplicate lightweight validation for UX purposes.

Example:

```text
liters must be > 0
```

may be checked immediately in the form.

However, backend validation remains authoritative.

---

# Domain Rules Must Not Depend on the UI

Core domain behavior must remain usable independently from Vue.

For example:

```text
calculate_full_to_full_consumption(...)
```

should not depend on:

* HTTP request objects
* Vue models
* PrimeVue
* browser state

This keeps domain logic:

* testable
* reusable
* deterministic where possible
* independent from presentation technology

---

# Backend Internal Structure

The exact Rust workspace may evolve, but the architecture should separate concerns conceptually.

A possible initial structure is:

```text
apps/
├── api/
└── web/

crates/
├── domain/
└── db/
```

or a similarly simple arrangement.

The project must not introduce crates merely to create architectural appearance.

New crates should exist only where they create a meaningful dependency or domain boundary.

---

# Domain Layer

The domain layer contains business concepts and rules.

Examples:

```text
Household
HouseholdMembership
Vehicle
Powertrain
RefuelEvent
ChargeEvent
Money
ConsumptionResult
MeasurementQuality
```

It may also contain pure calculations such as:

```text
full_to_full_consumption(...)
weighted_unit_price(...)
cost_per_km(...)
```

The domain layer should avoid direct dependencies on:

* Axum
* HTTP
* PostgreSQL
* SQLx
* Vue
* file storage
* external APIs

where practical.

---

# Application Layer

Application logic coordinates use cases.

Examples:

```text
CreateVehicle
RecordRefuel
EditRefuel
InviteHouseholdMember
CalculateVehicleHistory
```

Conceptually:

```text
HTTP Request
      ↓
Application Use Case
      ↓
Domain Rules
      ↓
Persistence
```

This layer may coordinate multiple domain operations and repositories.

---

# Infrastructure Layer

Infrastructure contains technical implementations such as:

```text
PostgreSQL persistence
SQLx queries
session storage
file storage
external APIs
OCR adapters
```

Infrastructure depends on the needs of the application.

The domain must not be designed around infrastructure-specific details.

---

# Dependency Direction

The intended conceptual dependency direction is:

```text
Infrastructure
      ↓
Application
      ↓
Domain
```

and:

```text
HTTP/API
      ↓
Application
      ↓
Domain
```

The domain should sit at the most stable center of the application.

It should not import outward-facing infrastructure merely because that makes one implementation easier.

---

# Pragmatism Over Architectural Purity

This decision does not require implementing textbook Clean Architecture with a large number of abstractions.

For example, OmniFleet does **not** automatically need:

```text
VehicleRepository
VehicleRepositoryImpl
VehicleService
VehicleServiceImpl
VehicleFactory
VehicleManager
VehicleCoordinator
```

for a simple operation.

The architecture should preserve meaningful boundaries without introducing ceremony.

A direct and readable implementation is preferred over unnecessary indirection.

---

# Vertical Slices

OmniFleet will be developed using vertical product slices.

Example:

```text
Create Vehicle
```

may involve:

```text
database migration
domain type
persistence
API endpoint
frontend form
validation
tests
```

before moving on to the next major product feature.

This is preferred over building:

```text
all database tables
↓
all repositories
↓
all endpoints
↓
all frontend pages
```

before anything becomes usable.

---

# Initial Product Development Flow

The intended sequence is approximately:

```text
v0.1
Walking Skeleton

↓

v0.2
ICE Refuel Vertical Slice

↓

v0.3
Consumption Engine

↓

v0.4
Dashboard
```

Each milestone should leave the system in a working state.

The roadmap may change based on actual usage.

---

# API Style

The frontend and backend will initially communicate through an HTTP JSON API.

Conceptually:

```text
GET    /api/vehicles
POST   /api/vehicles

GET    /api/refuels
POST   /api/refuels

PATCH  /api/refuels/{id}
```

The exact routes are defined later.

REST-style resource-oriented HTTP is sufficient for the initial product.

---

# GraphQL

GraphQL is not part of the initial architecture.

It would introduce additional:

* schema infrastructure
* client tooling
* authorization complexity
* caching concerns

without solving a known product requirement.

If future UI requirements make GraphQL materially advantageous, the decision may be revisited.

---

# Real-Time Communication

WebSockets are not part of the initial architecture.

Vehicle events are relatively infrequent.

Typical OmniFleet usage does not require continuous server push.

Normal:

```text
request
→ response
```

interaction is sufficient.

If future product behavior requires server-driven updates, technologies such as:

```text
Server-Sent Events
WebSockets
```

may be considered based on the concrete use case.

Real-time technology will not be added merely because it is available.

---

# Background Jobs

The first product versions do not require a distributed job-processing system.

Operations should initially remain synchronous where practical.

Future features may require background work, for example:

```text
OCR
large imports
external synchronization
scheduled exchange-rate retrieval
thumbnail generation
```

At that point, a job-processing mechanism may be introduced.

This does not imply that OmniFleet needs a message broker from the beginning.

---

# Message Brokers

The initial architecture explicitly avoids:

```text
Kafka
RabbitMQ
NATS
```

and similar distributed messaging infrastructure.

There is currently no workload requiring them.

If a future architectural requirement emerges, that decision should receive its own ADR.

---

# CQRS

OmniFleet will not initially use CQRS as a formal architecture.

Commands and queries may naturally use different code paths where useful, but the application does not require separate read/write models merely as an architectural pattern.

For example:

```text
POST /refuels
```

and:

```text
GET /dashboard
```

will naturally perform different operations.

This does not require a full CQRS infrastructure.

---

# Event Sourcing

OmniFleet will not use event sourcing as its persistence model.

Historical corrections and recalculable analytics can be supported using a conventional relational database.

The requirements defined in ADR-0004 do not require rebuilding all application state from immutable event streams.

A simpler relational model is preferred.

---

# Microservices

The initial application will not use microservices.

Concepts such as:

```text
auth service
vehicle service
analytics service
receipt service
```

will remain modules inside the same backend.

This avoids unnecessary distributed-system complexity.

If one area later has genuinely independent:

* scalability
* deployment
* reliability
* ownership

requirements, extracting it remains possible.

Extraction should respond to evidence rather than anticipation.

---

# Database

PostgreSQL is the primary relational datastore.

It stores authoritative application data such as:

```text
users
households
memberships
vehicles
refuels
charges
```

and future related entities.

PostgreSQL is expected to be sufficient for both operational and reporting workloads for the foreseeable scale of OmniFleet.

---

# SQLx

The Rust backend will use SQLx for PostgreSQL access.

The project favors explicit SQL and compile-time checked query workflows over introducing a heavyweight ORM abstraction.

SQL is considered part of the application's engineering surface rather than something that must always be hidden behind object persistence abstractions.

---

# Repository Pattern

A formal repository abstraction is not mandatory for every database operation.

For example, if a feature can remain clear with:

```text
application use case
→ SQLx query
```

there is no requirement to introduce an interface merely to satisfy an architectural pattern.

Repository abstractions may be introduced where they provide:

* testability
* meaningful domain isolation
* multiple implementations
* reusable persistence behavior

They should not be created automatically.

---

# Transactions

Operations that must preserve multiple related invariants should use database transactions.

Example:

```text
Create Household
+
Create initial Owner Membership
```

must behave atomically.

Conceptually:

```text
BEGIN

create household
create owner membership

COMMIT
```

If either operation fails:

```text
ROLLBACK
```

The database is part of enforcing application correctness, not merely passive storage.

---

# Database Constraints

Business invariants that can be safely represented in PostgreSQL should also be protected by database constraints where appropriate.

Examples:

```text
foreign keys
unique membership constraints
CHECK constraints
NOT NULL
```

Application-level validation still exists to provide better errors and handle rules too complex for database constraints.

The two layers complement each other.

---

# Migrations

Database schema changes will use versioned migrations.

Production database changes must never rely on manually editing schemas.

Conceptually:

```text
migration 001
migration 002
migration 003
```

The precise SQLx migration workflow is documented in the persistence ADR.

---

# Authentication

OmniFleet will use application-managed authentication.

The current direction is:

```text
email/password
Argon2id password hashing
server-managed sessions
secure cookies
```

The exact authentication architecture belongs to ADR-0008.

Application architecture must not assume stateless JWT authentication.

---

# File Storage

Receipts and future attachments should not be stored directly as large binary values inside core relational tables by default.

The architecture should support a storage abstraction such as:

```text
Attachment Storage

├── Local filesystem
└── S3-compatible object storage
```

PostgreSQL stores metadata and references.

This feature is implemented only when receipt management reaches the roadmap.

---

# OCR

OCR belongs outside the core domain.

Conceptually:

```text
Receipt
   ↓
OCR Adapter
   ↓
Suggested Data
   ↓
User Confirmation
   ↓
Domain Event
```

OCR output cannot bypass domain validation.

The backend domain remains authoritative regardless of the OCR technology used.

---

# External Integrations

Future external services should be accessed through narrow adapters.

Possible examples:

```text
exchange-rate provider
vehicle manufacturer API
charging network
fuel-price API
```

The domain should work without depending directly on any specific provider.

Provider-specific behavior belongs at the infrastructure boundary.

---

# Error Handling

The backend should have consistent application error handling.

Conceptually errors may include:

```text
ValidationError
AuthorizationError
NotFound
Conflict
InfrastructureError
```

These domain/application errors are translated into appropriate HTTP responses at the API boundary.

Domain logic should not return HTTP status codes directly.

---

# Logging and Tracing

The backend will use structured logging/tracing from the beginning.

Important operations should be observable without filling the codebase with arbitrary debug output.

Logging should capture useful context such as:

```text
request
operation
result
error category
```

while avoiding secrets and unnecessary personal data.

---

# Configuration

Runtime configuration should come from environment-based configuration.

Examples:

```text
DATABASE_URL
SESSION_SECRET
SERVER_BIND
storage configuration
```

Secrets must not be committed to the repository.

Development-specific defaults may be provided where safe.

---

# Testing Strategy

Testing should follow domain risk rather than arbitrary coverage numbers.

The highest priority is deterministic domain logic.

Examples:

```text
consumption calculations
money calculations
vehicle event compatibility
household invariants
measurement quality
```

These should have focused unit tests.

Persistence and API behavior should use integration tests where the database or HTTP boundary matters.

Frontend tests should focus on meaningful interaction and critical workflows.

---

# Calculation Tests

Calculation scenarios defined during Stage 0 should become executable tests.

For example:

```text
Full A
Partial B
Full C
```

should have documented expected behavior before implementation.

This prevents business logic from being invented accidentally inside API handlers or frontend components.

---

# Frontend Structure

The Vue application should be organized primarily around product features rather than technical file types alone.

Preferred direction:

```text
features/
├── auth/
├── vehicles/
├── refuels/
├── charges/
└── dashboard/
```

rather than requiring all logic to be separated globally into:

```text
components/
services/
models/
helpers/
```

regardless of domain relationship.

Shared components may still live in common areas where appropriate.

---

# State Management

Pinia will be available for application state.

However, not all data should automatically become global state.

Local component or feature state is preferred when data does not need application-wide coordination.

Global stores should represent genuinely shared state such as:

```text
authenticated user
active household
possibly selected vehicle
```

rather than becoming a cache for every API response.

---

# API Client

Frontend HTTP communication should pass through a small, typed API layer rather than scattering raw fetch calls throughout components.

Conceptually:

```text
UI
 ↓
feature API/client
 ↓
HTTP
```

This centralizes:

* request behavior
* response typing
* authentication behavior
* error translation

without introducing unnecessary frontend service hierarchies.

---

# Server-Side Rendering

Server-side rendering is not required for the initial product.

OmniFleet is primarily an authenticated application rather than a public content site requiring SEO.

A standard Vue SPA is sufficient.

Public project documentation or marketing pages may be handled separately if ever needed.

---

# Self-Hosted Deployment

Self-hosting is a primary product goal.

The initial deployment model should remain understandable.

Conceptually:

```text
OmniFleet API
PostgreSQL
Frontend
```

with optional attachment storage later.

Docker-based deployment is expected to be the primary convenient route.

The application should not require Kubernetes or cloud-specific infrastructure to run.

---

# Cloud Independence

OmniFleet should remain deployable without depending on a single cloud provider.

Cloud services may be supported through standard interfaces such as:

```text
PostgreSQL
S3-compatible storage
HTTP APIs
```

but the core application should not require proprietary cloud architecture.

---

# Scaling Strategy

OmniFleet will optimize based on measured bottlenecks.

The default scaling assumption is:

```text
single backend instance
+
PostgreSQL
```

This is expected to comfortably support the initial product.

Possible future optimizations include:

```text
database indexes
query improvements
pagination
cached derived analytics
background jobs
object storage
```

before considering distributed services.

---

# Performance Principle

Correct and understandable implementation comes before speculative optimization.

For example, a household with:

```text
3 vehicles
4 energy events per vehicle per month
10 years
```

would still produce only a small operational dataset.

PostgreSQL can handle this easily.

Architecture must therefore not pretend that OmniFleet has hyperscale requirements before they exist.

---

# Security Boundary

The backend is the trust boundary.

The frontend is considered an untrusted client.

Every operation must enforce:

```text
authentication
authorization
validation
```

server-side.

Client-side hiding of UI elements is not authorization.

---

# API Compatibility

Before v1.0, the internal frontend/backend API may evolve rapidly.

The application does not initially promise long-term public API stability.

After a stable external API is intentionally published, versioning guarantees may be defined separately.

This allows the pre-v1 domain and API to evolve without unnecessary backward-compatibility burden.

---

# Documentation

Significant architectural changes should be documented through ADRs.

The purpose is not to document every implementation detail.

ADRs should capture decisions that:

* constrain future design
* affect multiple features
* have meaningful alternatives
* would otherwise be difficult to understand later

---

# Explicitly Avoided Premature Complexity

The initial architecture deliberately avoids introducing the following without demonstrated need:

```text
microservices
Kubernetes
Kafka
RabbitMQ
CQRS infrastructure
event sourcing
GraphQL
Redis
distributed caching
service mesh
multiple databases
WebSockets everywhere
generic plugin architecture
complex dependency injection frameworks
```

This list is not a declaration that these technologies are bad.

It is a declaration that they currently solve no proven OmniFleet problem.

---

# Alternatives Considered

## Alternative A — Layered Monolith Without Domain Modules

Example:

```text
controllers/
services/
repositories/
models/
```

### Advantages

* familiar
* simple starting structure

### Disadvantages

* domain boundaries become unclear as the application grows
* unrelated features become coupled through generic layers
* feature ownership becomes harder to understand

### Decision

Not preferred.

Some technical layers will exist, but feature/domain boundaries remain important.

---

## Alternative B — Microservices

### Advantages

* independent deployment
* isolated scaling
* strong service boundaries

### Disadvantages

* large operational overhead
* distributed authorization
* network failure modes
* significantly harder development
* no current scaling requirement

### Decision

Rejected.

---

## Alternative C — Modular Monolith

### Advantages

* simple deployment
* explicit domain boundaries
* easy local development
* direct transactions
* straightforward testing
* future extraction remains possible

### Disadvantages

* internal discipline is required to preserve module boundaries
* deployment remains coupled

### Decision

Accepted.

---

# Invariants

## Single Authoritative Backend

Business rules are enforced server-side.

---

## Domain Independence

Core domain calculations should not depend directly on HTTP, Vue or PostgreSQL where practical.

---

## One Primary Database

PostgreSQL is the primary authoritative datastore.

---

## Modular Boundaries

Major product domains should remain structurally separated inside the monolith.

---

## No Premature Distribution

A module becomes an independent service only when a demonstrated requirement justifies it.

---

## Vertical Delivery

Features should be developed end-to-end rather than implementing entire technical layers in advance.

---

## Simple Deployment

The default self-hosted architecture must remain runnable without orchestration platforms such as Kubernetes.

---

# Initial v0.1 Architecture

The Walking Skeleton should be approximately:

```text
omnifleet/
│
├── apps/
│   ├── api/
│   └── web/
│
├── crates/
│   ├── domain/
│   └── db/
│
├── docs/
│
└── deployment / development config
```

With runtime flow:

```text
Vue
 ↓
Axum
 ↓
SQLx
 ↓
PostgreSQL
```

The first milestone only needs enough architecture to prove this complete path works.

---

# Validation Scenarios

## Scenario 1 — Domain Test

Given a pure consumption calculation:

```text
32.4 L
500 km
```

the calculation can be tested without:

```text
HTTP
PostgreSQL
Vue
```

---

## Scenario 2 — Backend Validation

Frontend prevents creating a RefuelEvent for an EV.

A malicious client bypasses the frontend and calls the API directly.

Then:

```text
backend
→ rejects the operation
```

---

## Scenario 3 — Household Creation

Creating:

```text
Household
+
Initial Owner Membership
```

uses one database transaction.

Partial creation must not remain after failure.

---

## Scenario 4 — New Dashboard

Dashboard reporting is introduced.

This does not require:

```text
new analytics microservice
new database
message broker
```

unless actual measurements prove such infrastructure necessary.

---

## Scenario 5 — Future OCR

OCR is introduced later.

It integrates as:

```text
infrastructure adapter
→ suggestion
→ application/domain validation
```

rather than changing the core source-of-truth model.

---

# Deferred Decisions

This ADR deliberately does not define:

* exact Rust crate structure
* exact module names
* Axum router structure
* API endpoint naming
* DTO conventions
* database schema
* authentication implementation
* file-storage implementation
* deployment compose file
* production reverse proxy
* observability vendor
* CI provider details
* background-job implementation

These are implementation-level or later architectural decisions.

---

# Consequences

## Positive

OmniFleet starts with an architecture proportionate to its real requirements.

The application remains straightforward to develop, test, deploy and self-host.

Domain boundaries can remain clear without introducing distributed-system complexity.

The system can evolve based on measured requirements.

---

## Negative

A modular monolith relies partly on engineering discipline to prevent modules from becoming tightly coupled.

A single backend deployment means modules cannot initially be deployed independently.

Future extraction of a module may require refactoring if genuine independent service requirements emerge.

These costs are accepted because they are significantly smaller than premature distributed architecture.

---

# Result

OmniFleet will begin as a **modular monolith**.

The primary architecture is:

```text
Vue 3 + TypeScript
        ↓
HTTP JSON API
        ↓
Rust + Axum
        ↓
SQLx
        ↓
PostgreSQL
```

The backend owns authoritative business rules.

Core domain logic remains independent from transport and persistence concerns where practical.

Development proceeds through usable vertical slices.

OmniFleet explicitly avoids premature microservices, distributed messaging, CQRS, event sourcing and similar infrastructure until real requirements justify them.

The architecture favors simple, explicit and testable code over unnecessary abstraction.

This decision is accepted as part of the Stage 0 foundation.

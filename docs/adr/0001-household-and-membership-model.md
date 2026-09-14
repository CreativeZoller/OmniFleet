# ADR-0001: Household and Membership Model

* **Status:** Accepted
* **Date:** 2026-09-14
* **Stage:** Stage 0 — Domain & Engineering Foundation

## Context

OmniFleet is intended to support multiple users and multiple vehicles in a shared family or household environment.

A simple implementation could assign each vehicle directly to a user:

```text
User
└── Vehicle
```

and store a global role such as `owner` or `member` on the user.

That model becomes problematic as soon as multiple people need access to the same vehicle.

For example:

```text
Zoltán
Anna
│
└── Hyundai i30
```

The vehicle should not need to be duplicated or transferred between individual user accounts simply because both household members use it.

The role of a user is also contextual.

A person is not inherently an `owner` or `member` of OmniFleet. They hold a role within a specific shared group.

The data model should therefore separate:

* user identity
* household membership
* authorization role
* vehicle ownership boundary

before authentication and persistence are implemented.

---

## Decision

OmniFleet will use a **Household-based ownership model**.

The core relationship is:

```text
User
  │
  ▼
Household Membership
  │
  ▼
Household
  │
  ├── Vehicle A
  ├── Vehicle B
  └── Vehicle C
```

Vehicles belong to a `Household`, not directly to a `User`.

Users gain access to household resources through a `HouseholdMembership`.

Roles belong to that membership.

---

## Domain Model

Conceptually:

```text
User
- id
- name
- email
- ...

Household
- id
- name
- ...

HouseholdMembership
- household_id
- user_id
- role
- ...

Vehicle
- id
- household_id
- ...
```

The exact database representation will be defined later.

This ADR defines the relationships and invariants rather than SQL column types.

---

## User

A `User` represents an authenticated person.

Users exist independently from households.

A user may theoretically belong to multiple households.

The domain therefore must not assume:

```text
user.household_id
```

as the ownership model.

Instead, household participation is represented through memberships.

---

## Household

A `Household` is the primary collaboration and ownership boundary in OmniFleet.

It owns shared resources such as vehicles.

Conceptually:

```text
Household
├── Membership
│   └── User A
├── Membership
│   └── User B
│
├── Vehicle A
└── Vehicle B
```

A vehicle belongs to exactly one household at a time.

The household is also the primary authorization boundary.

A user cannot access a vehicle merely because they know its identifier.

They must have an active membership in the household that owns the vehicle.

---

## Household Membership

A `HouseholdMembership` represents the relationship between one User and one Household.

It contains the user's role within that household.

Initial roles are:

```text
Owner
Member
```

A membership must uniquely identify the combination:

```text
(household, user)
```

A user cannot have multiple simultaneous memberships in the same household.

---

## Owner Role

An `Owner` represents a household administrator.

The intended capabilities include:

* manage household settings
* invite members
* remove members
* promote or demote members
* create vehicles
* archive vehicles
* manage household-level configuration
* access household reports and analytics
* record vehicle activity

Exact endpoint-level authorization will be defined when authentication and API design are implemented.

---

## Member Role

A `Member` represents a regular household participant.

The intended capabilities include:

* view household vehicles
* record refuels
* record charging sessions
* view vehicle history
* view dashboard analytics
* view reports

Members do not manage household membership or household-level administrative settings.

---

## Household Creation

Creating a household must also create its initial Owner membership.

The operation should behave conceptually as one atomic action:

```text
Create Household
        +
Create Owner Membership
```

A newly created household must never exist without an Owner.

---

## Owner Invariant

Every active household must have at least one active Owner.

Therefore OmniFleet must reject operations that would leave a household without an Owner.

Examples:

```text
Household owners: 1

Remove owner
→ REJECT
```

and:

```text
Household owners: 1

Demote owner to Member
→ REJECT
```

If there are multiple owners:

```text
Owner A
Owner B
```

then one may be removed or demoted while the other remains.

---

## Ownership Transfer

OmniFleet does not require a separate special `transfer ownership` domain operation.

Ownership can be transferred using normal membership operations:

```text
Member
  ↓
Promote to Owner

Old Owner
  ↓
Demote to Member
```

The invariant requiring at least one Owner prevents the household from entering an invalid state during the process.

The UI may later provide a dedicated transfer workflow for convenience.

---

## Multiple Households

The domain and persistence model should allow a user to belong to more than one household.

Example:

```text
User
├── Family Household
└── Shared Project Household
```

This capability does **not** mean that the initial OmniFleet UI must expose complex multi-household workflows.

For the first product versions, the UI may assume one primary or currently active household.

However, the underlying model must not make multiple households impossible.

### Rationale

Supporting membership as a many-to-many relationship from the beginning is inexpensive.

Retrofitting it later after introducing:

```text
users.household_id
```

would require significantly more migration and authorization work.

---

## Active Household

If a user belongs to multiple households, the application may later introduce the concept of an:

```text
active household
```

for navigation and UI context.

The active household is a presentation/application concern.

It does not change resource ownership.

Authorization must always be determined from the actual household relationship of the requested resource.

---

## Vehicle Ownership

A Vehicle belongs directly to one Household:

```text
Vehicle
  ↓
Household
```

It does not belong to:

```text
Vehicle
  ↓
User
```

A user's ability to access the vehicle comes from:

```text
User
  ↓
HouseholdMembership
  ↓
Household
  ↓
Vehicle
```

This relationship should be used consistently throughout authorization logic.

---

## Event Attribution

Although vehicles belong to households, individual activity must still preserve who recorded it.

For example:

```text
RefuelEvent
- vehicle
- recorded_by_user
```

or:

```text
ChargeEvent
- vehicle
- recorded_by_user
```

The user recording an event does not become the owner of that event or vehicle.

`recorded_by` exists for:

* attribution
* auditability
* household activity history
* correction workflows

---

## Membership Removal and Historical Data

Removing a user from a household must not destroy historical vehicle data.

For example:

```text
2026-09-10
Anna recorded a refuel

2027-02-01
Anna leaves the household
```

The historical refuel should still retain the fact that Anna originally recorded it.

Membership removal therefore must not cascade into deletion of:

* refuels
* charges
* historical attribution
* calculations
* reports

The exact persistence strategy for inactive or removed memberships will be defined with the database model.

---

## User Deletion

User-account deletion and privacy handling are separate concerns from household membership removal.

Historical vehicle records must not disappear simply because a user account is no longer active.

The eventual implementation may need to choose between:

* retaining the user reference
* anonymizing attribution
* using a historical actor representation

This decision is intentionally deferred until account lifecycle and privacy requirements are designed.

---

## Authorization Principle

Authorization should be resource-based.

Conceptually:

```text
requesting user
      ↓
membership?
      ↓
resource household
      ↓
role allows operation?
```

For example, when accessing:

```text
GET /vehicles/{vehicle_id}
```

the application should resolve the vehicle's household and verify membership.

It should not trust a household identifier supplied by the client as proof of authorization.

---

## Alternatives Considered

### Alternative A — Vehicle belongs directly to User

```text
User
└── Vehicle
```

#### Advantages

* simplest initial schema
* straightforward authorization
* suitable for strictly single-user applications

#### Disadvantages

* poor representation of shared family vehicles
* awkward vehicle sharing
* unclear ownership when multiple people use a vehicle
* future household migration would affect large parts of the application

#### Decision

Rejected.

OmniFleet is explicitly intended for shared multi-user vehicle tracking.

---

### Alternative B — Global role on User

Example:

```text
User
- role = owner | member
```

#### Advantages

* extremely simple authorization model

#### Disadvantages

A role has no meaning without context.

A person could theoretically be:

```text
Owner in Household A
Member in Household B
```

A global role cannot represent that.

#### Decision

Rejected.

Roles belong to HouseholdMembership.

---

### Alternative C — User contains household_id

Example:

```text
User
- household_id
```

#### Advantages

* supports shared households
* simpler than a membership table

#### Disadvantages

* permanently limits one user to one household
* combines identity and organizational membership
* complicates future invitations and household switching
* makes role history harder to model

#### Decision

Rejected.

The persistence model should support many-to-many household membership from the beginning.

---

### Alternative D — Household Membership Model

```text
User
       \
        HouseholdMembership
       /
Household
   │
   └── Vehicles
```

#### Advantages

* models family sharing naturally
* keeps identity independent from ownership
* roles have the correct context
* supports future multi-household users
* provides a clean authorization boundary
* avoids vehicle duplication

#### Disadvantages

* introduces one additional domain relationship
* authorization queries require membership validation
* household context must be handled in the UI

#### Decision

Accepted.

The additional complexity is small and reflects the real domain more accurately.

---

## Invariants

The following invariants must eventually be enforced through application logic and, where appropriate, persistence constraints.

### Membership uniqueness

A User may have at most one active membership in the same Household.

```text
unique(household_id, user_id)
```

---

### Vehicle ownership

A Vehicle belongs to exactly one Household.

---

### Resource access

A User may access household resources only through a valid HouseholdMembership.

---

### Owner requirement

An active Household must always contain at least one active Owner.

---

### Role scope

Roles apply to HouseholdMemberships, never globally to Users.

---

### Event attribution

A recorded vehicle event preserves the identity of the user who created it where possible.

---

### Historical preservation

Removing a membership must not delete historical vehicle activity.

---

## Initial MVP Behavior

The domain supports multiple households, but early OmniFleet versions do not need to expose all related workflows.

The initial product may assume:

```text
User registers
      ↓
Creates Household
      ↓
Becomes Owner
      ↓
Creates Vehicle
```

Later:

```text
Owner
  ↓
Invites Member
  ↓
Member joins Household
```

Multi-household switching can be introduced when there is a demonstrated product need.

The underlying domain remains compatible with it from the beginning.

---

## Deferred Decisions

This ADR deliberately does not define:

* invitation token implementation
* invitation expiration
* authentication technology
* session structure
* exact role middleware
* account deletion behavior
* privacy/anonymization workflow
* audit-log implementation
* household deletion behavior
* whether additional roles are eventually needed

These decisions do not change the core household ownership model and can therefore be resolved later.

---

## Consequences

### Positive

The model gives OmniFleet a clear ownership hierarchy:

```text
Household
  ↓
Vehicle
  ↓
Vehicle Events
```

while user access remains:

```text
User
  ↓
Membership
  ↓
Household
```

This cleanly separates:

* identity
* ownership
* authorization
* activity attribution

It also allows the application to expand beyond a single-user system without redesigning the core data model.

### Negative

Every household-scoped resource operation requires authorization through membership.

This introduces additional application logic compared with direct user ownership.

The UI will also eventually need a household context when users belong to multiple households.

These costs are considered acceptable because they represent actual domain requirements rather than accidental technical complexity.

---

## Implementation Guidance

The future persistence model will likely resemble:

```text
users

households

household_memberships
    user_id
    household_id
    role

vehicles
    household_id
```

This is illustrative only.

Exact fields, indexes, foreign keys and deletion strategies will be specified in the persistence ADR.

---

## Validation Scenarios

These scenarios should later become integration or domain tests.

### Scenario 1 — Household creation

```text
Given:
User A exists

When:
User A creates Household X

Then:
Household X exists
User A has an Owner membership in Household X
Household X has at least one Owner
```

### Scenario 2 — Shared vehicle access

```text
Given:
User A is Owner of Household X
User B is Member of Household X
Vehicle V belongs to Household X

Then:
User A can access Vehicle V
User B can access Vehicle V
```

### Scenario 3 — Unauthorized vehicle access

```text
Given:
User C is not a member of Household X
Vehicle V belongs to Household X

Then:
User C cannot access Vehicle V
```

### Scenario 4 — Last Owner removal

```text
Given:
Household X has exactly one Owner

When:
The Owner is removed or demoted

Then:
The operation is rejected
```

### Scenario 5 — Multiple Owners

```text
Given:
Household X has Owner A and Owner B

When:
Owner A becomes a Member

Then:
The operation succeeds
Owner B remains Owner
```

### Scenario 6 — Historical attribution

```text
Given:
User B recorded a refuel for Vehicle V
User B is later removed from the Household

Then:
The Refuel still exists
Its original recorder remains historically identifiable
```

### Scenario 7 — Multiple Household capability

```text
Given:
User A belongs to Household X

When:
User A joins Household Y

Then:
Both memberships can coexist
Resources remain isolated by Household
```

---

## Result

OmniFleet will use **Household as the resource ownership boundary** and **HouseholdMembership as the user authorization relationship**.

Vehicles belong to Households.

Roles belong to Memberships.

Vehicle events preserve individual user attribution without transferring ownership to the recording user.

The data model will support multiple household memberships from the beginning, while the initial product UI may remain focused on a single active household.

This decision is accepted as part of the Stage 0 foundation.

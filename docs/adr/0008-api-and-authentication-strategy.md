# ADR-0008: API and Authentication Strategy

* **Status:** Accepted
* **Date:** 2026-09-14
* **Stage:** Stage 0 — Domain & Engineering Foundation

## Context

OmniFleet is primarily an authenticated self-hosted web application.

The initial product consists of:

```text
Vue 3 frontend
        ↓
HTTP JSON API
        ↓
Rust / Axum backend
```

Most application data is private household data.

Examples include:

* vehicles
* refuels
* charging sessions
* receipts
* costs
* household membership
* analytics

The authentication and API model therefore needs to prioritize:

* secure browser authentication
* simple self-hosted deployment
* server-side authorization
* predictable API behavior
* protection against common browser attacks
* minimal authentication complexity

The application does not currently need:

* public third-party API clients
* mobile-native authentication
* distributed microservices
* stateless authentication across independent services

---

# Decision

OmniFleet will initially use:

```text
email + password
        ↓
server-managed session
        ↓
secure HTTP cookie
```

Authentication will be stateful.

The browser will receive an opaque session identifier stored in a secure cookie.

The session itself is managed and validated by the backend.

OmniFleet will **not use JWT access tokens as the primary browser authentication mechanism** for the initial product.

---

# Authentication Flow

Conceptually:

```text
User submits credentials
        ↓
Backend validates credentials
        ↓
Backend creates session
        ↓
Browser receives session cookie
        ↓
Future requests automatically include cookie
        ↓
Backend resolves authenticated User
```

The frontend does not need to manually store or attach authentication tokens.

---

# Registration

The initial registration flow may be:

```text
POST /api/auth/register
```

with input conceptually similar to:

```text
email
password
name
```

Successful registration creates:

```text
User
```

but household creation is a separate application operation.

A first-run product flow may guide the new user immediately into:

```text
Create Household
        ↓
Become Owner
```

as defined in ADR-0001.

---

# Login

Conceptually:

```text
POST /api/auth/login
```

The backend:

1. finds the User
2. verifies the password
3. creates a new authenticated session
4. sends the session cookie
5. returns safe authenticated-user information

Authentication failures must not reveal unnecessary account information.

For example, the API should avoid exposing meaningful differences between:

```text
email does not exist
```

and:

```text
password incorrect
```

to unauthenticated clients.

---

# Password Storage

Passwords must never be stored in plaintext or reversibly encrypted.

OmniFleet will use:

```text
Argon2id
```

for password hashing.

Conceptually:

```text
password
   ↓
Argon2id
   ↓
password hash
```

Only the resulting password hash and the parameters necessary for verification are persisted.

Password hashing configuration must be treated as security-sensitive application configuration and may evolve over time.

---

# Password Hash Upgrades

Password hashing parameters may become outdated.

The authentication implementation should allow:

```text
successful login
        ↓
detect old hash parameters
        ↓
rehash password
        ↓
store upgraded hash
```

without requiring all users to manually reset their passwords.

The exact implementation is deferred.

---

# Sessions

A successful login creates a server-managed session.

Conceptually:

```text
Session
- id
- user_id
- created_at
- expires_at
- last_seen_at
- revoked_at
```

The exact persistence model is deferred.

The browser should receive only an opaque session identifier.

The cookie must not contain authoritative user information such as:

```text
user_id
role
household_id
permissions
```

that the server blindly trusts.

---

# Session Cookie

The authentication cookie should use secure browser settings appropriate to the deployment.

Expected characteristics include:

```text
HttpOnly
Secure in HTTPS environments
appropriate SameSite policy
restricted Path
```

`HttpOnly` prevents normal frontend JavaScript from reading the session identifier.

The application should not store authentication secrets in:

```text
localStorage
sessionStorage
Pinia persisted state
```

for the normal browser authentication flow.

---

# Why Sessions Instead of JWT

JWT authentication was considered.

For OmniFleet's current architecture, JWT introduces little benefit.

The application has:

* one backend
* one primary browser frontend
* no independent microservices
* no current third-party API authentication requirement

Server-managed sessions make several important operations straightforward:

```text
logout
session revocation
account suspension
session expiration
security response
```

The server remains the authority.

---

# JWT Is Not Forbidden Forever

This ADR does not claim JWT is inherently unsuitable.

A future requirement such as:

```text
native mobile client
public API
third-party integrations
independent services
```

may justify a separate token-based authentication model.

That should be introduced for a concrete use case rather than preemptively.

---

# Session Expiration

Sessions must have finite validity.

The exact expiration policy will be decided during implementation.

Possible concepts include:

```text
idle timeout
absolute lifetime
```

A session should not remain valid indefinitely merely because a cookie still exists.

---

# Logout

Logout must invalidate the server-side session.

Conceptually:

```text
POST /api/auth/logout
```

causes:

```text
session revoked
+
authentication cookie removed
```

Deleting only the browser cookie is insufficient if the corresponding session remains valid on the server.

---

# Session Revocation

The architecture should permit future features such as:

```text
logout all devices
revoke individual session
force logout after password change
force logout after security event
```

without replacing the authentication system.

The exact UI is deferred.

---

# Current User Endpoint

The frontend needs a reliable way to restore authentication state after page reload.

Conceptually:

```text
GET /api/me
```

returns information about the currently authenticated user.

For example:

```json
{
  "id": "...",
  "name": "...",
  "email": "..."
}
```

Household context may also be returned or queried separately.

The cookie remains the credential.

---

# Frontend Authentication State

The frontend may store:

```text
authenticated user
active household
UI-level permission information
```

in Pinia or feature state.

This is presentation state.

It is not authoritative authorization data.

For example:

```text
frontend says:
user is owner
```

does not mean the backend may skip checking the actual membership.

---

# Trust Boundary

The backend is the security boundary.

The frontend is treated as an untrusted client.

Every protected operation must enforce server-side:

```text
authentication
authorization
validation
```

even if the frontend already prevents invalid behavior.

---

# Authentication vs Authorization

These concepts remain separate.

## Authentication

Answers:

> Who is making this request?

Result:

```text
Authenticated User
```

## Authorization

Answers:

> Is this user allowed to perform this operation on this resource?

Authorization depends on domain relationships such as:

```text
User
  ↓
HouseholdMembership
  ↓
Household
  ↓
Vehicle
```

as defined by ADR-0001.

---

# Household Authorization

Authorization must be based on the resource's real ownership.

Example:

```text
GET /api/vehicles/{vehicle_id}
```

The backend should conceptually perform:

```text
load vehicle
        ↓
find vehicle.household
        ↓
check authenticated user's membership
        ↓
authorize operation
```

The client cannot establish authorization by supplying:

```text
household_id
```

in the request.

---

# IDOR Protection

Object identifiers must never be treated as access control.

Example:

```text
User A can access:
vehicle_id = 100

User A tries:
vehicle_id = 101
```

If Vehicle 101 belongs to another Household:

```text
→ access denied
```

even if the identifier is valid.

Every household-owned resource query must preserve the household authorization boundary.

---

# Owner vs Member Authorization

Roles are defined by ADR-0001.

Conceptually:

```text
Owner
```

may perform administrative actions such as:

```text
invite member
remove member
change role
archive vehicle
manage household settings
```

while:

```text
Member
```

may perform normal vehicle operations such as:

```text
record refuel
record charge
view analytics
```

The exact permission matrix will evolve with features.

Authorization logic should remain explicit rather than relying on frontend visibility.

---

# Permission Checks

The application should prefer meaningful permission checks such as:

```text
can_manage_household
can_manage_vehicle
can_record_vehicle_event
can_view_vehicle
```

where that improves clarity.

However, OmniFleet does not initially require a generic enterprise-grade RBAC framework.

The current role model is intentionally small:

```text
Owner
Member
```

The application should not introduce a large permission engine before requirements justify it.

---

# CSRF

Because OmniFleet uses cookie-based authentication, state-changing requests require protection against Cross-Site Request Forgery.

The implementation must use an appropriate CSRF strategy.

State-changing methods include:

```text
POST
PUT
PATCH
DELETE
```

Possible implementation mechanisms include:

* CSRF tokens
* origin validation
* SameSite cookies combined with explicit request protection

The exact mechanism will be selected during implementation.

`SameSite` alone should not be treated as the complete conceptual security model.

---

# CORS

In normal production deployment, the frontend and backend should preferably be served under the same site or controlled origins.

Example:

```text
https://omnifleet.example.com
```

with API routes under:

```text
https://omnifleet.example.com/api/...
```

This simplifies:

* cookies
* CORS
* CSRF protection
* self-hosting

During development, separate frontend and API origins may be used.

CORS configuration must explicitly allow only intended origins.

Wildcard authenticated CORS is not acceptable.

---

# HTTPS

Production deployments handling authentication must use HTTPS.

Secure authentication cookies rely on encrypted transport.

The application itself may sit behind a reverse proxy responsible for TLS termination.

Example:

```text
Internet
   ↓
HTTPS Reverse Proxy
   ↓
OmniFleet
```

The exact reverse proxy is not defined by this ADR.

---

# API Style

The initial API will be resource-oriented HTTP with JSON payloads.

Examples:

```text
GET    /api/vehicles
POST   /api/vehicles

GET    /api/vehicles/{id}
PATCH  /api/vehicles/{id}

POST   /api/vehicles/{id}/refuels
GET    /api/vehicles/{id}/refuels
```

The exact URL structure may evolve.

Consistency is more important than rigid REST purity.

---

# JSON

JSON is the primary API representation.

Frontend and backend DTOs should use explicit structures rather than exposing database rows directly.

Conceptually:

```text
Database Model
     ≠
API DTO
     ≠
Domain Type
```

They may look similar, but they serve different purposes.

---

# API DTOs

API request and response types should represent the contract needed by clients.

For example:

```text
CreateRefuelRequest
```

should not automatically be identical to:

```text
Refuel database row
```

because fields such as:

```text
id
recorded_by
recorded_at
calculated metrics
```

may be server-controlled.

---

# Server-Controlled Fields

Clients must not be trusted to provide authoritative values such as:

```text
recorded_by_user_id
household ownership
role
recorded_at
calculation quality
```

These values are determined by the server from authenticated state and domain rules.

---

# Error Responses

API errors should use a predictable structure.

Conceptually:

```json
{
  "error": {
    "code": "INVALID_ODOMETER",
    "message": "The odometer cannot be lower than the previous reading."
  }
}
```

The exact schema may evolve.

The API should distinguish meaningful categories such as:

```text
validation
authentication
authorization
not found
conflict
infrastructure failure
```

---

# HTTP Status Semantics

The API should use HTTP status codes consistently.

Examples:

```text
200 / 201
successful request

400
malformed request

401
authentication required

403
authenticated but not authorized

404
resource unavailable / not found

409
state conflict

422
domain validation failure
```

The final mapping should remain consistent across endpoints.

---

# Resource Enumeration

For security-sensitive household resources, the API may sometimes deliberately avoid revealing whether an inaccessible resource exists.

For example:

```text
GET /vehicles/{id}
```

for another household's vehicle may return a generic:

```text
404
```

rather than revealing:

```text
403 — yes, that vehicle exists
```

The exact convention should remain consistent.

---

# Validation

API input passes through multiple levels of validation.

Conceptually:

```text
HTTP shape validation
        ↓
application validation
        ↓
domain validation
        ↓
database constraints
```

Example:

```text
liters = -10
```

may be rejected before persistence.

But database constraints may also protect against invalid values where practical.

---

# Password Validation

Password policy should focus on usable security rather than arbitrary complexity rules.

The implementation should avoid requirements such as:

```text
must contain exactly:
uppercase
lowercase
number
symbol
```

unless there is a demonstrated reason.

Minimum length and secure hashing are more important.

Exact password policy is deferred.

---

# Rate Limiting

Authentication endpoints should eventually be protected against abusive repeated attempts.

Examples:

```text
login
registration
password reset
```

The first walking skeleton does not need a complex distributed rate-limiting system.

A practical application-level strategy is sufficient for the expected deployment scale.

---

# Password Reset

Password reset is not required for the first technical walking skeleton but is required before a mature production release.

The future workflow should use:

```text
single-use
time-limited
random reset token
```

rather than exposing existing credentials.

The details are deferred.

---

# Email Verification

Email verification is not required for the first local/self-hosted MVP.

It may become necessary when:

* invitations depend on verified identity
* public registration exists
* password reset uses email
* externally hosted deployments are supported

The feature should be introduced when the product workflow requires it.

---

# Household Invitations

Household invitations belong to the application layer.

Conceptually:

```text
Owner
  ↓
Invite email/user
  ↓
Invitation
  ↓
User accepts
  ↓
HouseholdMembership
```

An invitation is not itself a membership.

The final invitation-token design is deferred until the household feature is implemented.

---

# Active Household Context

A User may eventually belong to multiple households.

The frontend may maintain:

```text
active household
```

for navigation and UX.

This does not establish authorization.

The backend must still validate membership for every requested resource.

The active household may be passed as routing or query context where useful, but it must never override actual resource ownership.

---

# API Versioning

Before v1.0, OmniFleet does not guarantee long-term API stability.

Therefore the application does not currently require:

```text
/api/v1/
```

solely for appearance.

A versioned external/public API may be introduced later if OmniFleet exposes a supported contract to third-party consumers.

Internal frontend/backend evolution should remain flexible during pre-v1 development.

---

# API Documentation

The API should become machine-documentable.

OpenAPI generation or maintained API specifications may be introduced as endpoints stabilize.

However, API documentation should follow real implemented contracts rather than requiring the entire future API to be designed before vertical slices are built.

---

# Public API

No unauthenticated general-purpose public vehicle-data API is planned for the initial product.

Private household data is authenticated by default.

Explicit public-sharing features, if ever introduced, require a separate security decision.

---

# Service-to-Service Authentication

Not required.

OmniFleet begins as a modular monolith.

There are no internal network services that require separate authentication credentials.

Future external workers or services may require their own mechanism if introduced.

---

# Refresh Tokens

The session-based browser authentication model does not require JWT-style:

```text
access token
+
refresh token
```

handling.

Session expiration and renewal can be handled directly by the server.

This is another reason stateful sessions are simpler for the current product.

---

# Authentication Failure Behavior

Expired or invalid sessions should result in a consistent unauthenticated response.

Conceptually:

```text
401 Unauthorized
```

The frontend can then:

```text
clear local authenticated state
        ↓
show login flow
```

without guessing authentication state.

---

# Security Logging

Security-relevant events should eventually be loggable.

Examples:

```text
successful login
failed login
logout
password change
membership change
role change
session revocation
```

Logs must not contain:

```text
passwords
session secrets
CSRF secrets
```

or unnecessary sensitive payloads.

---

# Secrets

Security secrets belong in runtime configuration.

Examples:

```text
SESSION_SECRET
CSRF secret if required
database credentials
email-provider credentials
```

They must never be committed to the repository.

Development environments may use explicitly documented non-production values where appropriate.

---

# Alternatives Considered

## Alternative A — JWT in localStorage

### Advantages

* simple client-side mental model
* common in SPA tutorials
* stateless server verification

### Disadvantages

* exposes tokens to JavaScript
* complicates secure revocation
* unnecessary for a single backend
* token refresh adds complexity
* encourages client-managed authentication state

### Decision

Rejected.

---

## Alternative B — JWT in HttpOnly Cookie

### Advantages

* protects token from direct JavaScript access
* supports stateless verification

### Disadvantages

* still requires CSRF considerations
* revocation remains more complex
* statelessness provides no meaningful current benefit
* session semantics are simpler for OmniFleet

### Decision

Rejected for initial browser authentication.

---

## Alternative C — Server-Managed Session Cookie

### Advantages

* straightforward browser security model
* easy revocation
* easy logout
* simple session management
* no client-side token storage
* fits single-backend architecture

### Disadvantages

* requires session persistence
* backend performs session lookup
* horizontal scaling may later require shared session storage or database-backed sessions

### Decision

Accepted.

At OmniFleet's expected scale, these disadvantages are negligible.

---

# Invariants

## Backend Authority

The backend is authoritative for authentication and authorization.

---

## Session Secrecy

Browser JavaScript does not need direct access to the session credential.

---

## Resource Authorization

Every household-owned resource operation validates HouseholdMembership.

---

## Role Context

Owner/Member roles are determined from HouseholdMembership, not from client input.

---

## Server-Controlled Identity

Clients cannot choose the `recorded_by` user for authenticated operations.

---

## CSRF Protection

State-changing cookie-authenticated requests require CSRF protection.

---

## No LocalStorage Auth Token

Primary browser authentication secrets must not be stored in browser localStorage.

---

## Password Security

Passwords are stored only as secure Argon2id hashes.

---

# Initial v0.1 Authentication Scope

The walking skeleton only needs enough functionality to prove:

```text
register
   ↓
login
   ↓
authenticated request
   ↓
logout
```

A minimal implementation should support:

```text
POST /api/auth/register
POST /api/auth/login
POST /api/auth/logout
GET  /api/me
```

plus protected API behavior.

Advanced features such as:

```text
email verification
password reset
session management UI
device history
MFA
```

are intentionally deferred.

---

# Validation Scenarios

## Scenario 1 — Successful Login

```text
Given:
valid account exists

When:
correct email and password are submitted

Then:
session is created
secure session cookie is returned
GET /api/me succeeds
```

---

## Scenario 2 — Invalid Credentials

```text
Given:
user submits invalid credentials

Then:
authentication fails
no session is created
response does not unnecessarily reveal account existence
```

---

## Scenario 3 — Protected Resource

```text
Given:
no authenticated session

When:
GET /api/vehicles

Then:
request is rejected as unauthenticated
```

---

## Scenario 4 — Valid Household Access

```text
Given:
User A is a Member of Household X
Vehicle V belongs to Household X

When:
User A requests Vehicle V

Then:
request is authorized
```

---

## Scenario 5 — Cross-Household Access

```text
Given:
User A belongs to Household X
Vehicle V belongs to Household Y

When:
User A requests Vehicle V

Then:
access is denied
```

---

## Scenario 6 — Client Forges Role

```text
Given:
User A is a Member

When:
client sends:
role = Owner

Then:
backend ignores client role
actual HouseholdMembership role is used
```

---

## Scenario 7 — Client Forges Recorder

```text
Given:
User A is authenticated

When:
CreateRefuel request contains:
recorded_by_user_id = User B

Then:
backend does not trust that field
recorded_by is derived from authenticated User A
```

---

## Scenario 8 — Logout

```text
Given:
authenticated session exists

When:
user logs out

Then:
server-side session is invalidated
cookie is removed
old session cannot access protected resources
```

---

## Scenario 9 — Expired Session

```text
Given:
session has expired

When:
protected API request is sent

Then:
backend returns unauthenticated response
frontend returns to login state
```

---

## Scenario 10 — CSRF Attempt

```text
Given:
authenticated browser session exists

When:
an unauthorized third-party origin attempts
a protected state-changing request

Then:
the request fails CSRF/origin protection
```

---

# Deferred Decisions

This ADR deliberately does not define:

* session database schema
* session expiration duration
* exact SameSite value
* exact CSRF implementation
* password minimum length
* password reset implementation
* email verification
* MFA
* invitation tokens
* session-device management UI
* exact API response envelope
* OpenAPI tooling
* reverse proxy configuration
* production rate-limiting mechanism

These decisions can be made during implementation without changing the chosen authentication architecture.

---

# Consequences

## Positive

Authentication remains simple and appropriate for an authenticated browser application.

Credentials do not need to be stored in frontend JavaScript-accessible storage.

Sessions can be revoked immediately.

Household authorization naturally follows the domain model from ADR-0001.

The API remains flexible during pre-v1 product development.

---

## Negative

The backend must maintain session state.

Cookie authentication requires deliberate CSRF protection.

Future native or third-party API clients may require an additional authentication mechanism.

Multiple backend instances would require shared session persistence.

These costs are accepted because they match OmniFleet's actual current requirements better than premature token infrastructure.

---

# Result

OmniFleet will initially use:

```text
Email + Password
        ↓
Argon2id
        ↓
Server-Managed Session
        ↓
Secure HttpOnly Cookie
```

The backend is the trust boundary.

All household resources are authorized server-side through HouseholdMembership.

The frontend may use authentication and role information for UX but never as the source of authorization truth.

Cookie-authenticated state-changing operations require CSRF protection.

JWT authentication, refresh tokens and public API authentication are deferred until a concrete future use case requires them.

This decision is accepted as part of the Stage 0 foundation.

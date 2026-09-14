# Stage 0 Handoff

## Status

Stage 0 complete

The Stage 0 documents are sufficient to begin v0.1. Remaining open items are implementation defaults or later vertical-slice decisions, not blockers for the Walking Skeleton.

---

## Documentation Reviewed

All listed files exist and were read in full:

```text
readme.md

docs/
├── domain-model.md
├── calculations.md
├── scenarios.md
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

`README.md` is present as `readme.md` on a case-insensitive filesystem.

No listed file was missing. `docs/architecture.md` was mentioned in the README documentation tree but does not exist; application architecture lives in ADR-0007. The README tree was updated to match the repository.

### Inventory

| Area | Coverage |
| ---- | -------- |
| Household ownership | Covered |
| Membership and roles | Covered |
| Authorization boundaries | Covered |
| Vehicle identity | Covered |
| ICE / EV / PHEV modeling | Covered |
| Multi-fuel vehicles | Covered |
| Refuel events | Covered |
| Charge events | Covered |
| Raw vs derived data | Covered |
| Money and currencies | Covered |
| Consumption calculations | Covered |
| Calculation quality | Covered |
| Experimental calculations | Covered |
| Application architecture | Covered |
| API strategy | Covered |
| Authentication | Covered |
| Persistence | Covered |
| Migrations | Covered |
| Representative test scenarios | Covered |

Notes on coverage, not gaps in Stage 0:

* Arbitrary partial-only ICE L/100km remains intentionally **Unavailable** until a documented Estimated method exists.
* EV consumption quality is conservative; charger kWh is not automatically Verified vehicle consumption.
* Exact Owner/Member endpoint permission matrices will evolve with each vertical slice. The authorization boundary is defined.

---

## Core Decisions

OmniFleet is a modular monolith:

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

| Topic | Decision | Authority |
| ----- | -------- | --------- |
| Ownership | Household owns Vehicle. User access is through HouseholdMembership. Role belongs to membership, not User. | ADR-0001 |
| Powertrain | ICE → RefuelEvent. EV → ChargeEvent. PHEV → both. HEV/mild hybrid initially ICE. | ADR-0002 |
| Events | RefuelEvent and ChargeEvent remain separate. No generic EnergyEvent. | ADR-0003 |
| Source of truth | Raw observations are authoritative. Derived values are reproducible. Caches are non-authoritative. | ADR-0004 |
| Money | Decimal + explicit currency. Confirmed `total_amount` is the paid amount. Weighted price = Σ amount / Σ quantity. | ADR-0005 |
| Quality | Verified / Estimated / Experimental. Vehicle-reported consumption is an external observation, not Verified. | ADR-0006 |
| ICE consumption | Full-to-full, including partials between full anchors. Starting full quantity is excluded. | calculations.md |
| Driver vs. Car | Displayed range depletion / actual distance. Experimental. Not applied to PHEVs. | calculations.md |
| Auth | Email/password, Argon2id, server-managed session, secure HttpOnly cookie. Backend is the trust boundary. | ADR-0008 |
| Persistence | PostgreSQL, SQLx, versioned SQL migrations. Applied shared/production migrations are immutable. | ADR-0009 |

Product development remains iterative vertical slices. The roadmap is direction, not a rigid contract.

---

## Consistency Review

### Expected direction — confirmed

* Household ownership, membership roles, and resource authorization match across domain-model, ADR-0001, ADR-0008, and scenarios.
* Powertrain event compatibility matches ADR-0002, ADR-0003, and scenarios 13–17.
* Partial refuels are valid purchases and are not treated as consumed fuel unless they sit inside a full-to-full segment.
* Driver vs. Car is a range-depletion ratio, not exact fuel consumption.
* Charger-delivered kWh is not treated as identical to battery-stored energy.
* PHEV fuel and electricity stay separate physical quantities; combined EUR/km is allowed when currencies match.
* Clients cannot supply authoritative role, household ownership, or `recorded_by_user_id`.
* Database constraints protect simple structural invariants; domain logic protects complex rules.

### Problems found and corrected

1. **Stale open questions in `domain-model.md`.** Section 22 still asked questions already decided by ADR-0001, ADR-0002, ADR-0003, ADR-0005, ADR-0006, and `calculations.md` (multi-household membership, last-Owner invariant, HEV as ICE, authoritative `total_amount`, range-ratio methodology).
2. **Charge energy invariant mismatch.** `domain-model.md` allowed `energy_kwh >= 0` (“must not be negative”). ADR-0003, ADR-0009, `calculations.md`, and scenario 32 require `energy_kwh > 0` when supplied.
3. **Weighted-price formula drift in ADR-0004.** The example used `quantity × unit_price`, which can disagree with confirmed `total_amount`. ADR-0005 requires `sum(total_amount) / sum(quantity)`.
4. **ChargeEvent time-field mismatch.** ADR-0003 ChargeEvent uses `started_at` / `ended_at`. Event ordering, monthly spending, and ADR-0009 index examples used `occurred_at` as if ChargeEvents had that field. Neither document had chosen start vs end as the canonical calendar time. This was recorded as deferred rather than invented.
5. **README documentation inventory was stale.** It still listed a non-existent `architecture.md` and showed Stage 0 as incomplete.
6. **README omitted full→partial→full verified consumption** and listed charging duration as if it were an independent source observation.
7. **Scenario coverage gaps.** Household creation and HEV-as-ICE existed in ADRs but not in `docs/scenarios.md`. Last-Owner protection covered demotion only.

### Mathematical review

Recalculated examples were arithmetically correct, including:

| Example | Result |
| ------- | ------ |
| Full-to-full `32.4 L / 500 km` | `6.48 L/100km` |
| Full→partial→full `(15+23) / 600 × 100` | `≈ 6.3333 L/100km` |
| Multiple partials `50 / 800 × 100` | `6.25 L/100km` |
| Starting-full exclusion `30 / 500 × 100` | `6.0 L/100km` |
| Distance-weighted average `(18+49)/(300+700)×100` | `6.7 L/100km`, not `6.5` |
| Weighted fuel price `110 / 70` | `≈ 1.5714 EUR/L`, not `1.55` |
| Effective charging `14 / 30` | `≈ 0.4667 EUR/kWh` |
| Driver vs. Car `415 / 500` | `0.83` Experimental |
| SoC estimate `60 kWh × 50%` | `30 kWh` Estimated |
| PHEV combined spend `115 / 900` | `≈ 0.1278 EUR/km` |

Receipt rounding `42.17 × 1.579 = 66.58643` vs stored `66.58` is correctly treated as a valid difference, with `total_amount` preserved.

No formula was changed merely because another approach could also be valid.

### Scenario coverage

`docs/scenarios.md` now has representative Stage 0 cases for household authorization, last-Owner protection, multi-household membership, multi-fuel vehicles, ICE/EV event compatibility, partial refueling, historical corrections, multi-currency, free charging, invalid odometer, missing range, PHEV, raw vs derived data, and session authorization.

Intentionally not exhaustive. CSRF, invitation tokens, and odometer-reset workflows remain in ADRs / deferred lists.

### Duplicate / premature decisions

Repetition of ownership, raw-vs-derived, money, and quality rules is useful context. The previous unclear source of authority was the stale domain-model open-question list; that is now split into resolved vs deferred.

The documents explicitly reject microservices, Kafka/RabbitMQ, Redis, CQRS infrastructure, event sourcing, Kubernetes, GraphQL, generic plugin systems, and ceremony-heavy repository/service layers. No such technologies should be added in v0.1.

Suggested crate layout in ADR-0007 is illustrative, not a frozen workspace contract.

---

## Corrections Made

| File | Why |
| ---- | --- |
| `docs/domain-model.md` | Marked Accepted; aligned charge-energy invariant; recorded Stage 0 resolutions; pointed authorization and money rules at the ADRs; documented HEV as ICE. |
| `docs/adr/0003-refuel-and-charge-event-model.md` | Stopped treating ChargeEvent `occurred_at` as defined; deferred canonical charge timestamp. |
| `docs/adr/0004-raw-vs-derived-data.md` | Weighted price now uses confirmed totals. |
| `docs/adr/0009-data-persistence-and-migration-strategy.md` | Index example no longer assumes `charges.occurred_at`. |
| `docs/calculations.md` | Monthly charging spend timestamp deferred; status Accepted. |
| `docs/scenarios.md` | Last-Owner removal; household creation; HEV-as-ICE; status Accepted. |
| `readme.md` | Stage 0 marked complete; documentation tree corrected; partial-refuel and charging-duration wording aligned. |
| `docs/stage-0-handoff.md` | This handoff. |

No application code, migrations, or v0.1 implementation files were added.

---

## Blocking Decisions

None.

v0.1 can begin by choosing ordinary implementation defaults for UUID generation, NUMERIC precision, session TTL, CSRF mechanism, crate layout, and cookie flags. Those choices do not change the Stage 0 domain or architecture.

---

## Deferred Decisions

Safe to resolve in the relevant later slice:

* Exact PostgreSQL NUMERIC precision/scale and Rust decimal library
* Exact UUID generation strategy
* Session schema, expiry, SameSite, CSRF implementation
* Password policy, email verification, password reset, MFA
* Invitation tokens and household deletion
* Account deletion / attribution anonymization
* Event audit/versioning, soft-delete, odometer-reset workflow
* ChargeEvent canonical calendar time (`started_at` vs `ended_at`)
* EV verified-consumption classification and charging-loss correction
* Arbitrary partial-only ICE estimation
* PHEV combined-spend segment boundaries and advanced PHEV analytics
* FX provider, historical rate policy, household preferred currency
* OCR, attachments, Places
* Automatic migration-at-startup vs explicit migrate step
* Backup/restore tooling
* Exact Axum router, DTO, and crate names
* Public API versioning

---

## v0.1 Implementation Boundary

Stage 0 ends here. v0.1 must not be treated as unfinished Stage 0 documentation work.

Do **not** introduce the following as part of Stage 0:

```text
Cargo.toml
package.json
Docker Compose implementation
database migrations
Axum routes
Vue components
SQLx queries
application code
```

Those belong to v0.1.

Create only the persistence and API surface required by the Walking Skeleton. Do not create the full future schema (charges, places, attachments, reporting caches) in advance.

---

## v0.1 Walking Skeleton Goal

Establish the smallest complete application path:

```text
Browser
   ↓
Vue
   ↓
Axum
   ↓
PostgreSQL
```

Intended v0.1 scope (from the project README / ADR-0007 / ADR-0008):

* Rust workspace and Axum API
* PostgreSQL via Docker Compose
* SQLx migrations
* Vue 3 + TypeScript + PrimeVue
* Environment-based configuration
* Structured logging / tracing
* Error handling
* Health endpoint
* Initial CI workflow
* Enough authentication to prove `register → login → authenticated request → logout`

v0.1 does **not** need the ICE refuel vertical slice, consumption engine, dashboard, household invitations, EV/PHEV events, receipts, or OCR. Those start at v0.2+.

---

## Stage 0 Definition of Done

- [x] Core domain terminology defined
- [x] Household ownership defined
- [x] Vehicle and powertrain model defined
- [x] Refuel and charge event boundaries defined
- [x] Raw vs derived data defined
- [x] Money and currency model defined
- [x] Consumption methodology documented
- [x] Measurement quality documented
- [x] Representative scenarios documented
- [x] Application architecture defined
- [x] API/authentication direction defined
- [x] Persistence/migration direction defined
- [x] Cross-document consistency reviewed
- [x] Remaining blocking decisions resolved
- [x] Future decisions explicitly deferred

---

## Next Stage

v0.1 — Walking Skeleton

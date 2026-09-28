# XYLO Availability Phase 3 — Reservation Lifecycle Integration

## Forensic Audit & Preparation Report

| Field | Value |
|---|---|
| Step | Phase 3 Forensic Audit + Preparation (Step 2) |
| Status | **COMPLETE — investigation and preparation only** |
| Code changes | **None.** No production code, schema, migration, API, DTO, or frontend was modified. |
| Phases 1 / 2 | Treated as LOCKED. Not reopened. |
| Scope boundary | Ends at Stage A (Review the audit). Stages B–F are out of scope. |

Evidence labels used throughout: **VERIFIED FACT** (confirmed in repository/schema/tests) · **OBSERVED BEHAVIOR** (current code path) · **INFERENCE** (reasonable, requires confirmation) · **BUSINESS DECISION REQUIRED** · **ARCHITECTURAL DECISION REQUIRED**.

---

## 1. Executive Summary

### 1.1 The single most important finding

**VERIFIED FACT — The Reservation → Availability integration does not exist in production code.**

The Phase 3 seam has been *designed and built on the Availability side*, but **not one Reservation code path consumes it**.

* `ReservationAvailabilityPort` is defined at `apps/api/src/modules/reservations/application/ports/reservation-availability.port.ts:48-75`.
* It is implemented by `AvailabilityAssertionService` (`apps/api/src/modules/availability/application/services/availability-assertion.service.ts:44`) and bound in the DI container at `apps/api/src/modules/availability/availability.module.ts:23`.
* Grep across `apps/api/src/modules/reservations` for `RESERVATION_AVAILABILITY_PORT`, `assertInTransaction`, `releaseInTransaction`, `replaceInTransaction`, `findCurrentInTransaction`, `setPopulationInTransaction` returns **only the port definition file itself** plus one Postgres spec. **Zero production call sites.**
* `ReservationOperationJournal` is registered (`reservations.module.ts:95,114`) but **never injected** outside `__tests__/reservation-operation-journal.spec.ts`.
* `AvailabilityController` exposes **only `GET /snapshot` and `GET /reconciliation`** (`availability.controller.ts:12,20`). There is no write endpoint, and `AvailabilityAssertionService.assert()` has no caller anywhere in the API.

**Phase 3 integration is at 0% wired.**

### 1.2 The second most important finding

**VERIFIED FACT — Phase 1 fail-closed and Phase 2 assertion capacity are structurally incompatible with today's Reservation data.**

`ReservationConsumptionAdapter` returns `status: 'UNRESOLVED'` whenever **any** candidate row exists, because the schema has no room quantity (`reservation-consumption.adapter.ts:23`). `AvailabilitySnapshotService` propagates that into `bookingEligibility: 'UNKNOWN'` and `sellableAvailable: 0` (`availability-snapshot.service.ts:80,100-102`). `AvailabilityAssertionService.unassertableReason()` then returns `UNRESOLVED_CAPACITY` (`availability-assertion.service.ts:699-710`) and the assertion is persisted **REJECTED**.

Consequences, all **VERIFIED FACT**:

1. Any stay date carrying **one or more** consuming-status reservations makes the snapshot unresolved → **every assertion for that room type/date is rejected**.
2. Worse: the adapter also fails closed for stored statuses `RESERVED`, `PROSPECT`, `IN_HOUSE` (`reservation-consumption.adapter.ts:20,26-27`), and **`reservations.reservation_status` defaults to `"RESERVED"`** (`packages/db/schema.prisma`, `model reservations` at `:9579`). The live create payload (`QuickBookForm` → `reservationApi.create`) **does not send `reservation_status`** — the only code that sets `'CONFIRMED'` on create is the *dead* Zustand store (`apps/web/store/reservationStore.ts:334`). Therefore **most live-created reservations land as `RESERVED`, which alone makes the snapshot UNRESOLVED**.
3. The consumption adapter is **not population-aware**: it scans `reservations` regardless of `reservation_availability_state.population`. Phase 2 capacity is `physical − gba − allotment − reservationConsumption` (`snapshot-calculator.ts:14,21`) and is then compared against `availability_assertion_balances.asserted_quantity`. Once assertions exist, the same inventory unit is counted **twice** (once in `reservationConsumption`, once in `asserted_quantity`) unless the adapter learns to exclude `ASSERTION_MANAGED` rows.

**INFERENCE:** with today's data, Phase 3 could be wired end-to-end and would still reject essentially every assertion. This is a hard blocker independent of the wiring work.

### 1.3 State of the repository vs. the state asserted by this task brief

**CONFLICT — reported, not resolved.**

This brief states Phase 3 is at the *forensic-audit* stage, that the business-rules decision sheet and locked domain specification are still ahead of us, and that **Reservation room quantity "remains an explicitly unresolved business/domain issue and MUST NOT be guessed or silently resolved during this step."**

The repository contains artifacts that appear to be a *later* stage of the same Phase 3:

| Artifact | Path | Timestamp | Self-declared status |
|---|---|---|---|
| Business-rules decision sheet | `docs/enterprise/availability-phase3-business-rules-decision-sheet.md` | 2026-09-27 03:40 | "**All business-rule decisions resolved**" (13/13), ends "**READY FOR DOMAIN SPECIFICATION**" |
| Domain specification | `docs/enterprise/availability-phase3-domain-specification.md` | 2026-09-27 03:52 | "Version 0.1 · **DRAFT — READY FOR REVIEW**" |
| Phase 3 DB migration | `packages/db/migrations/20260928000000_availability_phase3_reservation_foundation/migration.sql` | 2026-09-28 | Creates `reservation_availability_operations`, `reservation_availability_state`, supersedes chain, movement operation id |
| Phase 3 port + journal + specs | `reservation-availability.port.ts`, `reservation-operation-journal.ts`, two spec files | — | Implemented and unit/Postgres-tested |

All of these are **untracked in git** (last commit is `c854f79 Phase 0 - Enterprise Platform Foundation complete`).

The decision sheet explicitly asserts the quantity rule that this brief forbids me to resolve:

> "One Reservation represents one room-type inventory unit; quantity is always 1." — `availability-phase3-business-rules-decision-sheet.md:31`

and `AvailabilityAssertionService.assertInTransaction` **hard-codes `quantity: 1`** (`availability-assertion.service.ts:212`) with the comment `/** Execute a one-room Reservation assertion inside the caller's transaction. */` (`:198`).

**I have not adopted, reopened, or reconciled this.** It is logged as Conflict **C-0** in §17 and must be settled in Stage A/B before anything proceeds.

### 1.4 Bottom line

* The Availability side (Phase 1 + Phase 2) is genuinely built, tested, and locked.
* The Reservation side of Phase 3 is **designed but entirely unwired**.
* Three structural blockers stand between "unwired" and "safe to wire": **C-0** (state conflict), **B-1** (fail-closed consumption/quantity), **B-2** (population assignment missing).
* Several Reservation defects materially affect the integration and are recorded as audit inputs, **not** as fix authorizations.

**The project is NOT ready to move to the Business-Rules Decision stage as a clean slate — but it IS ready to move to Stage A (Review the audit), with C-0 as the first agenda item.** See §22.

---

## 2. Current Reservation Lifecycle

### 2.1 Canonical state machine

**VERIFIED FACT** — `packages/shared/src/reservation-state-machine.ts`:

* `RESERVATION_STATUSES` (storage vocabulary) `:1-14`: `RESERVED, PROSPECT, PENDING, CONFIRMED, GUARANTEED, CHECKED_IN, IN_HOUSE, CHECKED_OUT, CANCELLED, NO_SHOW, NO-SHOW, WAITLIST`.
* `CANONICAL_RESERVATION_STATUSES` `:18-28`: `PROSPECT, PENDING, CONFIRMED, GUARANTEED, CHECKED_IN, CHECKED_OUT, CANCELLED, NO_SHOW, WAITLIST`.
* Transitions `:32-42`:

```
PROSPECT    → PENDING, CONFIRMED, CANCELLED, WAITLIST
PENDING     → CONFIRMED, GUARANTEED, CANCELLED, WAITLIST
WAITLIST    → CONFIRMED, PENDING, CANCELLED
CONFIRMED   → GUARANTEED, CHECKED_IN, CANCELLED, NO_SHOW, CHECKED_OUT
GUARANTEED  → CHECKED_IN, CANCELLED, NO_SHOW, CHECKED_OUT
CHECKED_IN  → CHECKED_OUT, CONFIRMED
CHECKED_OUT → CHECKED_IN
CANCELLED   → CONFIRMED          (reinstate)
NO_SHOW     → CONFIRMED          (reinstate)
```

* Terminal: `CHECKED_OUT`, `CANCELLED` (`:44`). Check-in eligible from `CONFIRMED|GUARANTEED` (`:46`).
* Normalisation `:56-66`: `NO-SHOW→NO_SHOW`, `IN-HOUSE→CHECKED_IN`, **`RESERVED→CONFIRMED`**.
* Storage form `reservation-status.service.ts:12-18,59-62`: `NO_SHOW` is written as **`NO-SHOW`**; everything else is written canonically.

**OBSERVED BEHAVIOR — vocabulary drift:**

* `model reservations.reservation_status` defaults to **`"RESERVED"`** (`schema.prisma:9579`), a value that *normalises* to `CONFIRMED` but is *not* the string the Phase 1 consumption adapter matches.
* `IN_HOUSE` appears in `RESERVATION_STATUSES` but not in `CANONICAL_RESERVATION_STATUSES`; `normalizeStatus('IN_HOUSE')` **throws** `Unknown reservation status` (`:65`) because only `IN-HOUSE` (hyphen) is mapped.
* Two storage spellings for no-show coexist: `NO_SHOW` (canonical) vs `NO-SHOW` (what is actually written). `reinstate-reservation.handler.ts:27` gates on `'NO_SHOW'` while storage writes `'NO-SHOW'` → **VERIFIED FACT: `POST /reservations/:id/reinstate` can never reinstate a stored no-show** (only reachable via the Front Office handler, see §2.3).

### 2.2 The Reservation write surface

**VERIFIED FACT** — `apps/api/src/modules/reservations/api/controllers/reservations.controller.ts`, 74 routes across one controller. Lifecycle-relevant subset:

| Route | Line | Command | Handler |
|---|---|---|---|
| `POST /reservations` | 76 | `CreateReservationCommand` | `create-reservation.handler.ts` |
| `PUT /reservations/:id` | 84 | `UpdateReservationCommand` | `update-reservation.handler.ts` |
| `POST /reservations/:id/cancel` | 91 | `CancelReservationCommand` | `cancel-reservation.handler.ts` |
| `POST /reservations/:id/no-show` | 98 | `ProcessNoShowCommand` | `process-no-show.handler.ts` |
| `POST /reservations/batch/status` | 105 | `BatchUpdateStatusCommand` | `batch-update-status.handler.ts` |
| `DELETE /reservations/:id` | 112 | `DeleteReservationCommand` | `delete-reservation.handler.ts` |
| `POST /reservations/:id/waitlist` | 141 | `JoinWaitlistCommand` | `join-waitlist.handler.ts` |
| `POST /reservations/:id/waitlist/promote` | 149 | `PromoteFromWaitlistCommand` | `promote-from-waitlist.handler.ts` |
| `POST /reservations/:id/confirm` | 156 | `ConfirmReservationCommand` | `confirm-reservation.handler.ts` |
| `POST /reservations/:id/guarantee` | 163 | `GuaranteeReservationCommand` | `guarantee-reservation.handler.ts` |
| `POST /reservations/:id/reinstate` | 170 | `ReinstateReservationCommand` | `reinstate-reservation.handler.ts` |
| `POST /reservations/:id/extend` | 177 | `ExtendStayCommand` | `extend-stay.handler.ts` |
| `PUT /reservations/:id/room-type` | 184 | `ChangeRoomTypeCommand` | `change-room-type.handler.ts` |
| `PUT /reservations/:id/rate` | 191 | `ChangeRateCommand` | `change-rate.handler.ts` |
| `POST /reservations/auto-cancel/sweep` | 503 | `ExecuteAutoCancelSweepCommand` | `execute-auto-cancel-sweep.handler.ts` |
| `POST /reservations/mass-update/execute` | 455 | `MassUpdateExecuteCommand` | `mass-update-execute.handler.ts` |
| `POST /reservations/room-moves/:moveId/execute` | 233 | `ExecuteScheduledRoomMoveCommand` | `execute-scheduled-room-move.handler.ts` |

Front Office owns the operational transitions separately (`apps/api/src/modules/front-office/`): `CheckInGuest/FinalizeCheckIn`, `CheckOutCommand`, `ExtendStayCommand`, `UpgradeRoomCommand`, `TransferRoomCommand`, `ReverseCheckInCommand`, `ReinstateReservationCommand`, plus walk-in.

### 2.3 VERIFIED FACT — command-name collisions in the shared CommandBus

`CommandBus.register()` (`common/cqrs/command-bus.ts:20-27`) keys handlers by `getHandledCommandName()` and **logs a warning and overwrites on duplicate** — last writer wins.

Two exact collisions exist:

| Command name | Reservations handler | Front Office handler |
|---|---|---|
| `ExtendStayCommand` | `reservations/…/extend-stay.handler.ts:20` | `front-office/…/extend-stay.handler.ts:4` |
| `ReinstateReservationCommand` | `reservations/…/reinstate-reservation.handler.ts:20` | `front-office/…/reinstate.command.ts:4` |

Both modules register in `onModuleInit` (`reservations.module.ts:250-311`, `front-office.module.ts:208-241`). `ReservationsModule` is imported at `app.module.ts:66` and `FrontOfficeModule` at `:68`.

**INFERENCE (requires runtime confirmation):** Nest initialises modules in imports order, so `FrontOfficeModule` registers second and **overwrites** the Reservations handlers. If correct, `POST /reservations/:id/extend` and `POST /reservations/:id/reinstate` execute **Front Office** handlers, not the Reservations handlers quoted in §2.2. This must be confirmed during Stage A because it changes which code path owns the extension/reinstatement availability effect.

**ARCHITECTURAL DECISION REQUIRED:** whether command names must become globally unique, namespaced, or whether the two `ExtendStay`/`Reinstate` commands must be formally merged.

### 2.4 OBSERVED BEHAVIOR — lifecycle transitions that are currently silent no-ops

**VERIFIED FACT** — `ReservationRepository.update()` (`reservation.repository.ts:518-601`) only reads **camelCase** keys: `checkIn`, `checkOut`, `roomType`, `rateCode`, `adults`, `children`, `status`, `paymentStatus`, `paymentSource`, `roomNumber`, `specialRequests|special_requests`, `rate`. The guard is `:556 if (hasInventoryChanges || hasSimpleUpdates)`; the transaction only opens at `:557`.

Handlers that pass **snake_case**, therefore writing nothing and touching no inventory:

| Handler | Passes | Line | Read by `update()`? | Result |
|---|---|---|---|---|
| `confirm-reservation.handler.ts` | `reservation_status` | 28 | no (`data.status` only) | `{success:true, modified:false}` |
| `guarantee-reservation.handler.ts` | `reservation_status`, `guarantee_type`, `guarantee_ref` | 30–34 | no | silent no-op |
| `reinstate-reservation.handler.ts` | `reservation_status` | 31 | no | silent no-op |
| `change-room-type.handler.ts` | `room_type` | 30 | no (`data.roomType`) | silent no-op |
| `extend-stay.handler.ts` | `departure_date` | 35 | no (`data.checkOut`) | silent no-op |
| `change-rate.handler.ts` | `rate_code`, `nightly_rate`, `total_rate` | 30–34 | no | silent no-op |

Compounding it: `update-reservation.dto.ts:4-17` declares `arrival_date, departure_date, room_type, rate_code, room_rate, …` (snake), while the repository reads camel — so **`PUT /reservations/:id` with a date or room-type change is also a silent no-op**. The frontend *does* send snake_case (`ChangeDatesDialog.tsx:122`, `ChangeRoomTypeDialog.tsx:106`).

And the frontend's "Mark No-Show" sends `PUT /reservations/:id {reservation_status:'NO-SHOW'}` (`reservation.api.ts:186`, `front-office.api.ts:213`) — **not** `POST /:id/no-show` — so it is likewise a silent no-op.

> **Scope note.** These are Reservation defects. Per §5 of the brief they are **audit inputs, not fix authorizations**. They are in scope here *only because* they determine whether the future Availability operation is even reachable from the real lifecycle path. They must be triaged in Stage A as "prerequisite / parallel track / ignore".

---

## 3. Reservation → Availability Dependency Map

### 3.1 The intended seam (built, unwired)

```
ReservationsController / FrontOfficeController
        │
        ▼
  ReservationsService  ── getHotelId() / getRequestContext().userId
        │
        ▼
   CommandBus  ── pipes: idempotency → validation → logging → authorization → transaction
        │              (cqrs.module.ts:37-41)
        ▼
  Command Handler  ── validate status transition (ReservationStatusService)
        │
        ▼
  ReservationRepository  ── prisma.$transaction  (reservation.repository.ts:433/557/605/636/671/808/835/877)
        │
        ├──►  [MISSING] ReservationOperationJournal.claim(tx, …)
        ├──►  [MISSING] RESERVATION_AVAILABILITY_PORT.assertInTransaction / releaseInTransaction / replaceInTransaction
        ├──►  [PRESENT] CrsEngineService → InventoryDomainService → legacy `availability` table
        └──►  EventBus.publish(event, tx)  ── IntegrationEvent → outbox (in-tx)
```

**VERIFIED FACT** — every box marked `[MISSING]` has zero production call sites. `[PRESENT]` is the legacy path that is live today.

### 3.2 The seam's contract (Availability side, built and tested)

**VERIFIED FACT** — `reservation-availability.port.ts:48-75`:

| Method | Input | Semantics |
|---|---|---|
| `assertInTransaction(tx, ReservationAssertionRequest)` | `operationId`, `availabilityOperationKey`, `context`, `roomType`, `arrivalDate`, `departureDate` | Requires `population === 'ASSERTION_MANAGED'` **and** `current_assertion_id === null` (`availability-assertion.service.ts:204-206`); quantity hard-coded to **1** (`:212`); links `current_assertion_id` on success (`:242-247`) |
| `releaseInTransaction(tx, ReservationAssertionRelease)` | exact `assertionId` + explicit `stayDates[]` + `actorId` | Validates link (`:257-259`), validates each date has net movement `+1` (`:270-273`), locks balances in sorted order (`:275`), writes `REVERSAL` movements with `operation_id` (`:278-286`), sets `RELEASED`/`PARTIALLY_RELEASED`, nulls or keeps `current_assertion_id` (`:291-295`) |
| `replaceInTransaction(tx, ReservationAssertionReplacement)` | current assertion + new room type/dates | Same-room-type overlapping dates are **transferred** without a balance round-trip (`:345-348`); room-type change ⇒ **no transfer**, full release + re-assert (`:345-347`); locks old + new keys in one sorted pass (`:350-354`); creates new assertion as `REPLACEMENT_PENDING` → `ACTIVE`, old → `SUPERSEDED` (`:374-411`) |
| `findCurrentInTransaction(tx, identity)` | — | `LEGACY` ⇒ no assertion; `ASSERTION_MANAGED` + no link ⇒ **throws `CURRENT_ASSERTION_MISSING` if status ∈ {CONFIRMED, GUARANTEED, CHECKED_IN}** (`:435-437`) |
| `setPopulationInTransaction(tx, identity, population, currentAssertionId?)` | — | Upserts `reservation_availability_state` (`:464-468`) |

**VERIFIED FACT** — `assertInTransaction` requires an existing `reservation_availability_state` row. **Nothing in the repository creates that row for a new reservation.** `setPopulationInTransaction` is only exercised by tests.

### 3.3 The seam's DB foundation (already migrated)

**VERIFIED FACT** — `packages/db/migrations/20260928000000_availability_phase3_reservation_foundation/migration.sql`:

* `reservations_id_hotel_id_uq` unique index (`:5-6`) — also present in `schema.prisma`.
* `availability_assertions.supersedes_assertion_id` + FK + index (`:8-25`).
* Status check extended to include `REPLACEMENT_PENDING`, `SUPERSEDED` (`:14-16`).
* **Partial unique index `availability_assertions_one_active_reservation_uq`** on `(hotel_id, reference_type, reference_id)` where status ∈ {ACTIVE, PARTIALLY_RELEASED} and `reference_type='RESERVATION'` (`:27-31`) ⇒ at most one live assertion per reservation per property.
* **`reservation_availability_operations`** — durable operation journal; PK `id`; `UNIQUE(hotel_id, operation_key)`; status ∈ {IN_PROGRESS, SUCCEEDED, REJECTED, FAILED}; FKs to `reservations(id,hotel_id)` and to old/new assertions (`:33-69`).
* **`reservation_availability_state`** — PK `(reservation_id, hotel_id)`; `population ∈ {LEGACY, ASSERTION_MANAGED}`; `LEGACY ⇒ current_assertion_id IS NULL`; FK to reservation and to assertion; **partial unique on `current_assertion_id`** (`:71-96`).
* `availability_assertion_movements.operation_id` + FK + **partial unique `(hotel_id, operation_id, assertion_id, stay_date, movement_type)`** ⇒ movement-level idempotency per operation (`:98-114`).
* **Deferred constraint trigger `reservation_availability_state_link_trg`** (`:121-144`) proving `current_assertion_id` references *this* reservation's own active assertion.

`schema.prisma` mirrors this: `model reservation_availability_state` at `:17385`, `model reservation_availability_operations` at `:17401`, `availability_state` / `availability_operations` relations on `model reservations`.

**VERIFIED FACT** — no data backfill or population cutover is present in the migration (`:1-3` explicitly states this). **Every existing reservation therefore has no `reservation_availability_state` row ⇒ population is unset ⇒ `assertInTransaction` would throw `RESERVATION_POPULATION_MISMATCH` for all of them.**

### 3.4 The legacy seam (live today)

```
ReservationRepository.cancel / processNoShow / delete
        │  (inside prisma.$transaction)
        ▼
CrsEngineService.releaseInventory  ── crs-engine.service.ts:581-586
        │  opens ITS OWN prisma.$transaction  ← NOT the caller's
        ▼
InventoryDomainService.release  ── inventory.domain-service.ts:180-185
        UPDATE "availability" SET reserved=GREATEST(0,reserved-$3), available=available+$3
                                   AND reserved>0

ReservationRepository.update (camelCase inventory keys)
        ▼
CrsEngineService.modifyReservation  ── crs-engine.service.ts:429; own $transaction at :450 and :523
        diff dates/room type → reserve(new) / release(old) / reserve(added)

CrsEngineService.confirmReservation  ── :293; $transaction :343; reserve(…, rooms:1) at :390
        ↑ only reached from POST /rates/engine/book (rates-inventory.controller.ts:107)
        ↑ NEVER from POST /reservations
```

**VERIFIED FACT** — `InventoryDomainService` self-declares at `inventory.domain-service.ts:21` *"Single writer for the `availability` counter"*. A repo-wide grep for raw `INSERT/UPDATE "availability"` confirms **only this file** writes that table.

**VERIFIED FACT** — `CrsEngineService.releaseInventoryTx` (`:588-591`), the transaction-taking variant, has **zero callers**. The live path opens a nested, separate transaction.

---

## 4. Complete Reservation Event Matrix

Columns per the brief. "Required Future Assertion Operation" is **proposed**, not decided.

| # | Reservation Event | Current Behavior (evidence) | Availability Impact today | Existing Mechanism | Required Future Assertion Operation | Transaction Boundary today | Idempotency Need | Open Decision |
|---|---|---|---|---|---|---|---|---|
| 1 | **create** (`POST /reservations`) | `create-reservation.handler.ts:21-46` → `repo.create` (`reservation.repository.ts:409-516`); single `prisma.$transaction` `:433-514`; writes `reservations`, `reservation_name` upsert, `special_requests`, `reservation_daily_elements`, `folio_header`, `folio_payments`; publishes `ReservationCreatedDomainEvent` in-tx `:501-513`. **No inventory reserve call.** | **None.** | none (CRS `confirmReservation` reserves but is a different route) | **DECIDE:** at which status an assertion is created; `journal.claim(ASSERT)` + `port.assertInTransaction` + `setPopulation(ASSERTION_MANAGED)` | reservation tx exists; assertion must join it | **HIGH** — create is the most-retried mutation (see §15.3) | D-1: create-time status & assertion timing |
| 2 | **confirm** (`POST /:id/confirm`) | `confirm-reservation.handler.ts:26-29` → `repo.update(id,{reservation_status:'CONFIRMED'})` → **silent no-op** (§2.4) | **None.** | none | assert (if create did not) or no-op if already asserted | n/a until no-op fixed | MEDIUM | D-1, plus P-1 (no-op fix is a prerequisite) |
| 3 | **guarantee** (`POST /:id/guarantee`) | `guarantee-reservation.handler.ts:28-35` → snake_case → **silent no-op** | **None.** | none | per D-1 (guarantee may be the consuming threshold) | n/a | MEDIUM | D-1 |
| 4 | **cancel** (`POST /:id/cancel`) | `cancel-reservation.handler.ts:38-119`; `repo.cancel` `:603-632`; `reservation_status='CANCELLED'` `:606`; `crs.releaseInventory(full stay)` `:609-613`; status history `:618-622`; `ReservationCancelledDomainEvent` in-tx `:629`. Penalty + settlement run **after** the tx `(:48-113)`. | **Legacy release of the whole `[arrival, departure)` range, rooms:1** | `CrsEngineService.releaseInventory` → `InventoryDomainService.release` | `journal.claim(RELEASE)` + `port.releaseInTransaction(exact assertionId, stayDates = full stay)` | status flip and release are in the *same* outer tx, but the release opens its **own** inner tx ⇒ **non-atomic** (see §9.2) | **HIGH** — cancel is retried by the client (`mutations.retry:1`) | D-2 (release date set), D-9 (atomicity) |
| 5 | **no-show** (`POST /:id/no-show`) | `process-no-show.handler.ts:26-29`; `repo.processNoShow` `:634-658`; status `'NO-SHOW'` `:637`; `crs.releaseInventory(full stay)` `:640-644`; event in-tx `:655` | **Legacy release of the full stay** | same as cancel | `journal.claim(RELEASE)` + `port.releaseInTransaction(stayDates = **future nights only**)` | same non-atomic structure as cancel | **HIGH** — must not double-release (locked rule) | **D-3:** locked rule says *"release remaining future nights, not the elapsed arrival night"* — current code releases **including** the arrival night ⇒ **conflict** |
| 6 | **batch status** (`POST /batch/status`) | `batch-update-status.handler.ts:19-20` → `repo.batchUpdateStatus` `:699-725`: validates transitions `:716-721`, then a **single `updateMany`** `:723`. **No release, no history, no event.** | **None — inventory leak when target is `CANCELLED`** | none | per-row `journal.claim` + release, atomic per Reservation | **no transaction at all** | **HIGH** — brief requires "atomic per Reservation with explicit partial results" | **D-4:** batch atomicity semantics (decision sheet row 11 claims RESOLVED — see C-0) |
| 7 | **check-in** (`POST /check-in/finalize`) | `check-in-commit.service.ts:82` sets `reservation_status='CHECKED_IN'`, `room_number`; status history `:94-99`. No inventory call. | **None** | none | **per locked rule: no second consumption** ⇒ expected operation is **verify current assertion exists**, not assert | check-in tx | LOW | D-1 confirm |
| 8 | **check-out, on time** (`POST /front-office/check-out`) | `check-out.handler.ts:206-227`: `reservation_status='CHECKED_OUT'`; `rooms.room_status='DIRTY'` `:232`. No departure-date change. | **None** | none | **per decision-sheet Decision 4: no release operation** — commitment ends at the exclusive departure date | FO tx | LOW | — |
| 9 | **check-out, early departure** | `check-out.handler.ts:117-142` posts refund/penalty only; `:207 updateData.departure_date = new Date()`; `:214` writes shortened `departure_date`. | **Shortens the stay in the DB with NO release of the unused future nights** | none | `journal.claim(REPLACE or RELEASE-subset)` + `port.releaseInTransaction(stayDates = unused future nights)` — must be atomic with the date shortening | FO tx (`client.reservations.update` inside `prisma.withTransaction`) | **HIGH** | **D-5:** confirm Decision 3 (early departure releases immediately) and its atomicity with the FO checkout tx |
| 10 | **check-out, overstay** | `check-out.handler.ts:48-113`: posts overstay charges, then `departure_date = today` at `:95` in a `withTransaction`, best-effort with `catch` `:109-111`. | **Extends the stay in the DB with NO assertion for the added night** | none | `journal.claim(ASSERT)` + `port.replaceInTransaction` (or assert-then-commit) **before** the date commit | nested `withTransaction`, failure swallowed | **HIGH** | **D-6:** Decision 5 (secure the night *before* committing the extension) — current code commits first and swallows failure ⇒ **direct conflict** |
| 11 | **modify dates** (`PUT /:id {arrival_date, departure_date}`) | `update-reservation.handler.ts:19` → `repo.update` with **snake_case** ⇒ **silent no-op**; if camelCase were used, `crs.modifyReservation` runs **outside** the reservation tx (`:545-554` before `:557`) | **None today**; legacy path would be non-atomic | `CrsEngineService.modifyReservation` `:429-537` (diff → reserve/release) | `journal.claim(REPLACE)` + `port.replaceInTransaction` (handles overlapping transfer + partial release + partial assert in one tx) | **non-atomic** even in the legacy path | **HIGH** — §12 case A | D-7, P-1 |
| 12 | **modify room type** (`PUT /:id/room-type`) | `change-room-type.handler.ts:30` snake_case ⇒ **silent no-op**. Front Office `upgrade-room.handler.ts:61` does a **read-only `inventoryDomain.isAvailable` guard**, then `:74` updates `room_type`. | **None** | FO guard only | `port.replaceInTransaction` with different `roomType` ⇒ transfer set is empty (`availability-assertion.service.ts:345-347`), full release + full re-assert | FO tx (atomic for the DB write; no inventory at all) | MEDIUM | **D-8:** upgrade vs. room-type change; complimentary/operational upgrade (decision-sheet row 13) |
| 13 | **modify room quantity** | **Does not exist.** `reservations` has no quantity column; `CreateReservationDto` has no rooms/quantity field; no route; no frontend mutation. | n/a | n/a | n/a | n/a | n/a | **D-0 (blocking)** — see §5 |
| 14 | **modify occupancy (adults/children)** | `repo.update` camelCase `adults`/`children` **does** reach `hasInventoryChanges` (`:532-533`) but produces empty `diffDates`/`addedDates` ⇒ `crs.modifyReservation` issues **zero SQL** | **None** (correct) | none | none — occupancy is not inventory | n/a | LOW | — |
| 15 | **extend stay** (`POST /:id/extend`) | Reservations handler `extend-stay.handler.ts:35` snake_case ⇒ **no-op**; subject to the `ExtendStayCommand` collision (§2.3). Front Office `extend-stay.handler.ts:58` `isAvailable` guard (read-only) then `:69` `departure_date=newEnd`. | **None.** Guard only. | FO `InventoryDomainService.isAvailable` | `journal.claim(ASSERT)` + assert added nights **then** commit dates | FO tx | MEDIUM | D-6 conflict applies here too |
| 16 | **shorten stay** | No dedicated command. Achieved only via early-departure checkout (row 9) or a date `PUT` (row 11, no-op). | none | none | `port.releaseInTransaction(subset)` | — | HIGH | D-5 |
| 17 | **reinstate** (`POST /:id/reinstate`) | Reservations handler gates on `'NO_SHOW'` but storage is `'NO-SHOW'` ⇒ never fires; also passes snake_case ⇒ no-op even if it fired. Subject to `ReinstateReservationCommand` collision (§2.3). Front Office `reinstate.handler.ts` exists as the likely live handler. | **None — reinstating does not re-reserve.** | none | `port.setPopulationInTransaction` + re-evaluate + assert **before** restoring a consuming state | — | MEDIUM | **D-10:** confirm Decision 6 (reinstatement re-evaluates Availability atomically first) |
| 18 | **delete** (`DELETE /:id`) | `repo.delete` `:660-697`; refuses `CHECKED_IN` `:666`; releases via `crs` if not already CANCELLED/NO-SHOW `:672-677`; hard-deletes reservation + folio rows `:679-694`. | Legacy release; **hard delete destroys the assertion's `reference_id` target** (`reservation_availability_operations_reservation_fk` is `ON DELETE RESTRICT`, migration `:57-59`) | `crs.releaseInventory` | n/a if delete is disallowed for assertion-managed rows | same non-atomic structure | MEDIUM | **D-11:** decision-sheet row 12 says active reservations are not hard-deleted — current code *does* hard-delete non-checked-in reservations ⇒ **conflict** |
| 19 | **join waitlist** (`POST /:id/waitlist`) | `join-waitlist.handler.ts:27,60` → `repo.joinWaitlist` `:834-875`: inserts `reservation_waitlist`, sets reservation status `WAITLIST`, in one tx. | **None** (correct — waitlist is non-consuming) | none | none (verify non-consuming) | single tx | LOW | — |
| 20 | **promote from waitlist** (`POST /:id/waitlist/promote`) | `promote-from-waitlist.handler.ts:26,28` → `repo.promoteFromWaitlist` `:876-889`: `WAITLIST→PROMOTED` + `reservations.reservation_status='CONFIRMED'` in one tx. **No inventory check.** | **None — promotion to a consuming status with no capacity check** | none | `journal.claim(ASSERT)` + `port.assertInTransaction` **inside** the same tx, failing closed | single tx (good) | MEDIUM | **D-12:** does promotion require a capacity gate? |
| 21 | **auto-cancel sweep** | `execute-auto-cancel-sweep.handler.ts:103` selects `CONFIRMED, GUARANTEED`; `:167-171` raw `UPDATE … SET reservation_status='CANCELLED'`. **No release, no event, no history.** HTTP-invoked only (no cron). | **None — inventory leak** | none | per-row release, atomic per Reservation | **raw SQL, no transaction around the batch** | HIGH | D-4 |
| 22 | **mass-update execute** | `mass-update-execute.handler.ts` — `checkInventory` queries a **non-existent table/column** (`:191-196`, `SELECT available_count FROM inventory`); skippable via `overrideAvailableInventory` `:72-78`. Job hotel never compared to caller hotel `:30-48`. | **None (query always fails)** | phantom query | n/a until the guard is defined | job-based | MEDIUM | **D-13:** what is the inventory gate for mass updates? |
| 23 | **scheduled room move execute** | `execute-scheduled-room-move.handler.ts:60` `repo.update({roomType})` — **camelCase ⇒ the legacy CRS path DOES fire** (`:546`). | **Legacy `crs.modifyReservation`** — the only reservation command that reliably reaches legacy inventory today | `CrsEngineService.modifyReservation` | same-type move ⇒ **no assertion change** (locked rule); type change ⇒ `replaceInTransaction` | non-atomic | MEDIUM | D-8 |
| 24 | **walk-in (Front Office)** | `front-office.controller.ts` → `walkIn` command. Creates a reservation through the FO path. | Check for `hotel_id='default'` fallback in this path — **see finding F-3** | — | same as create (row 1) | FO tx | HIGH | D-1 |
| 25 | **transfer room (FO)** | `transfer-room.handler.ts:55,67,76,106` — vacated room state, `room_moves`, `reservation_room_assignment_audit`, `reservation_changes`. | **None — physical assignment only** (correct) | none | none for same room type; replace for type change | FO tx | LOW | D-8 |
| 26 | **reverse check-in** | `reverse-check-in.handler.ts:65` `rooms.room_status='VACANT'`. | **None** | none | verify assertion still present (no release — reservation stays CONFIRMED/GUARANTEED) | FO tx | LOW | D-1 |
| 27 | **change rate** (`PUT /:id/rate`) | `change-rate.handler.ts:30-34` snake_case ⇒ **silent no-op**. | none | none | none (rate is not inventory) | n/a | LOW | — |
| 28 | **group/block-linked reservation** | `model reservations` carries `group_id`, `block_code`, `allotment_id`, `group_block_id`, `allotment_voucher_id`, `pickup_type`. `group-allotment` module writes only `inventory_policy` — **no availability write**. Phase 1 snapshot already subtracts `gbаRemaining + allotmentRemaining` (`snapshot-calculator.ts:14`). | Group/allotment consumption is handled **upstream in Phase 1**, not by assertions | Phase 1 sources | **none in Phase 3** (Phase 4 boundary) | — | — | **D-14:** confirm the reservation-side group assertion boundary belongs to Phase 4 |

---

## 5. Reservation Room Quantity — Phase 3 Decision Required

### 5.1 What is currently implemented

| Surface | Finding | Label |
|---|---|---|
| `model reservations` (`packages/db/schema.prisma:9579`) | Fields: `arrival_date, departure_date, room_number, room_type, rate_code, room_rate, market_code, source_code, adults, children, reservation_status, guarantee_type, …`. **No `quantity`, `rooms`, `room_count`, `num_rooms`.** | **VERIFIED FACT** |
| `model reservation_name` (`:9344`) | Composite Int PK, no quantity field | **VERIFIED FACT** |
| `CreateReservationDto` (`api/dto/create-reservation.dto.ts`) | 42 fields; **no rooms/quantity field** | **VERIFIED FACT** |
| `UpdateReservationDto` | No rooms/quantity field | **VERIFIED FACT** |
| `AvailabilityAssertionService.assertInTransaction` | `quantity: 1` hard-coded (`availability-assertion.service.ts:212`), comment *"one-room Reservation assertion"* (`:198`) | **VERIFIED FACT** |
| `CrsEngineService.confirmReservation` | `inventoryDomain.reserve(tx, {dates, rooms: 1})` (`crs-engine.service.ts:390`) | **VERIFIED FACT** |
| `CrsEngineService.releaseInventory` | `inventoryDomain.release(tx, {…, rooms: 1})` (`:584`) | **VERIFIED FACT** |
| No `reservation_rooms` / `reservation_room` / `room_assignments` model | Nearest: `room_assignment` (`:10238`), `reservation_room_assignment_audit`, `hk_room_assignments` — all physical assignment, not quantity | **VERIFIED FACT** |

### 5.2 What the database actually stores

**VERIFIED FACT — nothing.** There is no column anywhere in the `reservations` graph that expresses "how many rooms this reservation occupies." Multi-room bookings are representable **only** as multiple `reservations` rows.

### 5.3 What the Reservation API exposes

**VERIFIED FACT — no quantity in, no quantity out.** The create/update DTOs have no such field; there is no `PUT /:id/rooms`-style route.

**VERIFIED FACT (frontend)** — `QuickBookForm.tsx` collects a `Rooms` input at `:51,211,527-533` and builds `roomRates[]` at `:330-338`, but `handleSubmit` (`:457-482`) **never includes any rooms/quantity key in the payload**. `CopyReservationDialog.tsx:27` holds a `rooms` state that is never transmitted. **The quantity is collected and discarded.**

### 5.4 What Availability currently receives

| Source | Receives | Evidence |
|---|---|---|
| Phase 1 `ReservationConsumptionAdapter` | **Nothing usable** — returns `quantity: null`, `status: 'UNRESOLVED'` whenever any candidate row exists | `reservation-consumption.adapter.ts:23` — *"Reservation schema/domain has no persisted room quantity; locked procedure requires number of rooms, so candidate rows cannot be converted safely to room consumption"* |
| Phase 2 `assertInTransaction` | `quantity: 1` (assumed, not received) | `availability-assertion.service.ts:212` |
| Legacy `InventoryDomainService` | `rooms: 1` (assumed) | `crs-engine.service.ts:390,584` |

### 5.5 What Availability needs

**VERIFIED FACT** — `SnapshotCalculator.calculate` requires a scalar `reservationConsumption: number` per stay date (`snapshot-calculator.ts:2,14`). `AvailabilityAssertionRequest` requires `quantity: number > 0` (`assertion.contracts.ts:6`, validated at `availability-assertion.service.ts:653-655`).

### 5.6 What remains unresolved

**BUSINESS DECISION REQUIRED — D-0 (blocking).** The brief forbids resolving this during this step. The evidence presents *three mutually exclusive* readings:

| Reading | Source | Implication |
|---|---|---|
| **(a) Quantity is always 1; multi-room = multiple Reservations** | `docs/audit/Reservations/07_DECISIONS.md:1269` — *"One reservation maps to one room, with multiple guest records attached."* (Status: APPROVED); matches `assertInTransaction`'s hard-coded `1` and `crs`'s `rooms: 1` | Adapter can resolve `quantity = candidates.length`; blocker B-1 disappears |
| **(b) Quantity is a first-class, per-Reservation value > 1** | `docs/audit/Reservations/03_PROCEDURES.md:252` — required creation info includes *"number of rooms"*; `:338` modification category *"room quantity"*; `:430` **"PROCEDURE 04 — CHANGE ROOM TYPE / ROOM QUANTITY … Room-type and room-count changes are inventory-impacting"** with sequence `Validate requested room type/count → Check availability → … → Commit inventory change atomically` | Requires a schema column, DTO field, API surface, and `quantity > 1` assertion support — **but Phase 2 quantity must not be reopened** |
| **(c) Quantity is unresolved and must stay unresolved** | This brief's Phase 2 locked rule 13 | Phase 3 **cannot** resolve B-1 → Phase 3 cannot go live |

**CONFLICT:** reading (a) is recorded as RESOLVED in the pre-existing decision sheet (`availability-phase3-business-rules-decision-sheet.md:31`), reading (b) is an APPROVED Reservation specification, and reading (c) is the current instruction. **All three cannot hold.**

**Separately verified facts that bear on the decision:**

* `adults`/`children` exist and default to `1`/`0`; the decision sheet asserts guest counts do not affect inventory quantity.
* Shares/accompanying guests are sub-entities of one Reservation (`07_DECISIONS.md` ADR, Decision 1) — they do not multiply quantity.
* Group reservations carry `group_id`/`block_code`/`allotment_id` but these feed Phase 1's `gbaRemaining`/`allotmentRemaining`, not a reservation-level quantity.
* The `room_number` field is a **physical** assignment, not quantity (`model reservations`, `room_number String?`).

**Recommended Stage A framing:** D-0 is not a *Quantity* decision alone — it is the gate on blocker **B-1**. Nothing else in Phase 3 can be locked until it is settled.

---

## 6. Assertion Creation Semantics

### 6.1 The question

**"At what Reservation lifecycle point does a reservation become an Availability assertion?"**

### 6.2 Evidence from each of the four required sources

**(1) Existing XYLO behavior** — **VERIFIED FACT: there is none.** No reservation operation creates an assertion.

**(2) Documented business rules** — `docs/enterprise/availability-phase3-business-rules-decision-sheet.md:35` (pre-existing, see C-0):

> "Confirmed and Guaranteed Reservations consume inventory. Waitlist does not. PENDING consumption depends on operation context: an active hold consumes; a no-availability walk-in PENDING does not."
> `:36` "Check-in creates no second inventory consumption."

**(3) Existing reservation implementation** — the Phase 1 consumption adapter (`reservation-consumption.adapter.ts:5`):

```ts
const CONSUMING_CANDIDATE_STATUSES = ['PENDING', 'CONFIRMED', 'GUARANTEED', 'CHECKED_IN'];
const NON_CONSUMING_STATUSES       = ['CANCELLED', 'WAITLIST'];
```

**(4) Availability Phase 1/2 locked rules** — `availability-assertion.service.ts:435-437` (`findCurrentInTransaction`) enforces:

```ts
if (['CONFIRMED', 'GUARANTEED', 'CHECKED_IN'].includes(reservation.reservation_status)) {
  throw new AppError('CURRENT_ASSERTION_MISSING', …);
}
```

### 6.3 The contradiction

**VERIFIED FACT — three different consuming-status sets are in play:**

| Set | Source | PENDING? | PROSPECT? | CHECKED_OUT? |
|---|---|---|---|---|
| `{PENDING, CONFIRMED, GUARANTEED, CHECKED_IN}` | Phase 1 adapter `:5` | **yes** | no | no |
| `{CONFIRMED, GUARANTEED, CHECKED_IN}` | Phase 3 `findCurrentInTransaction` `:435` | **no** | no | no |
| `{CONFIRMED, GUARANTEED}` + contextual PENDING | pre-existing decision sheet `:35` | **contextual** | unstated | unstated |

Also **VERIFIED FACT:** the adapter does not recognise `RESERVED`, `PROSPECT`, or `IN_HOUSE` at all — they fall into `unresolvedStatuses` and **force `UNRESOLVED`** (`:20,26-27`), i.e. fail closed rather than being classified.

**INFERENCE (strong):** the intended Phase 3 semantics are *"assertion presence is required exactly for `CONFIRMED`, `GUARANTEED`, `CHECKED_IN`"* — this is the only set encoded in the Phase 3 foundation code. But it is **not** what Phase 1 currently consumes, and it contradicts the pre-existing decision sheet's treatment of PENDING.

**BUSINESS DECISION REQUIRED — D-1.** The exact consuming set, and within it:
* D-1a: does `PENDING` consume? (adapter says yes, foundation code says no, decision sheet says "depends")
* D-1b: does `PROSPECT` consume? (unstated everywhere; currently forces fail-closed)
* D-1c: how are storage spellings `RESERVED`, `IN_HOUSE`, `NO-SHOW` mapped before evaluation?
* D-1d: is the assertion created **at create** (so `PENDING`/`CONFIRMED` both assert) or **at confirm/guarantee**?
* D-1e: does `CHECKED_OUT` hold its assertion until departure, or release at checkout? (Decision 4 says no release at normal checkout → assertion naturally expires with the stay range.)

### 6.4 Proposed answer (for Stage B, not decided here)

**INFERENCE:** given (i) `findCurrentInTransaction`'s explicit list, (ii) the adapter's `CHECKED_IN` inclusion, (iii) Decision 4's "no release at normal checkout", and (iv) Decision 6's "check-in creates no second consumption" — the coherent model is:

> A Reservation enters the **consuming population** when it reaches `CONFIRMED` (or when it is created already in a consuming state), it **holds exactly one assertion across `CONFIRMED → GUARANTEED → CHECKED_IN → CHECKED_OUT`**, and the assertion's stay-range expiry — not a checkout event — ends the commitment.

This requires D-1a/D-1b to be settled and the adapter's status set to be aligned with `findCurrentInTransaction`'s — the latter being an **Availability Phase 1 change**, which is why it is raised as a blocker rather than assumed.

---

## 7. Assertion Release Semantics

### 7.1 What release does today (Phase 2, built)

**VERIFIED FACT** — `releaseInTransaction` (`availability-assertion.service.ts:251-297`):

1. Requires an exact `assertionId` that is the reservation's `current_assertion_id` (`:257-259`) — throws `ASSERTION_LINK_MISMATCH`.
2. Validates the assertion's `reference_type='RESERVATION'` and `reference_id = reservationId` (`:496-507`).
3. Validates every requested date is inside `[arrival, departure)` (`:509-521`) and has net movement `+1` (`:270-273`) — throws `ASSERTION_DATE_NOT_ACTIVE` otherwise.
4. Locks balances in **sorted (hotel, roomType, date) order** (`:536-565`).
5. Applies `-1` balance delta with a `gte` guard (`:567-578`) and appends a `REVERSAL` movement carrying `operation_id` (`:278-286`).
6. Sets assertion status `RELEASED` (no dates left) or `PARTIALLY_RELEASED` (`:289-291`).
7. Nulls `current_assertion_id` only when fully released (`:292-295`).

**VERIFIED FACT:** release is **partial-date capable** and **non-destructive** — the assertion row survives as history, and `availability_assertions_one_active_reservation_uq` excludes `RELEASED`, so a *new* assertion can be created afterwards. A released assertion can never be reactivated.

### 7.2 Which Reservation events should release

| Event | Proposed release behaviour | Basis | Status |
|---|---|---|---|
| cancel | release **all** stay dates | universal | **BUSINESS DECISION REQUIRED — D-2** (confirm) |
| no-show | release **remaining future nights only**, not the elapsed arrival night | pre-existing decision sheet `:37` (locked Night Audit rule) | **CONFLICT with current code** — `crs.releaseInventory` releases `[arrival, departure)` wholesale (`crs-engine.service.ts:581-586` + `dateRange` `:593-603`) |
| delete | release before delete, or disallow delete for assertion-managed rows | `reservation_availability_operations_reservation_fk` is `ON DELETE RESTRICT` (migration `:57-59`) | **D-11** |
| batch cancel / auto-cancel | release per row, atomic per reservation | brief §12 | **D-4** |
| early departure / shorten | release **unused future nights** | decision sheet Decision 3 | **D-5** |
| normal checkout | **no release** | decision sheet Decision 4 | consistent with stay-range expiry |
| check-in / reverse check-in | **no release** | decision sheet row 6 | — |
| reinstate | release already happened; reinstatement must **re-assert**, not un-release | decision sheet Decision 6; `port` has no "unrelease" | **D-10** |
| waitlist join | release if previously consuming | waitlist is non-consuming | **D-12** |
| room-type change | release old type + assert new type (no transfer) | `replaceInTransaction` `:345-347` | **D-8** |
| date change | release dropped dates only; transfer overlap | `replaceInTransaction` `:345-348` | **D-7** |

### 7.3 Release timing hazard

**VERIFIED FACT:** every legacy release today happens **after or alongside** the status write but in a **separate transaction** (§9.2). In the Phase 2 port the release is explicitly `InTransaction` — designed to be atomic with the Reservation mutation. The design intent is clear; the wiring is absent.

---

## 8. Reservation Modification Semantics

### 8.1 Dates change — example `10–15` → `12–18`

**Built behaviour (Phase 2), VERIFIED FACT** — `replaceInTransaction` (`availability-assertion.service.ts:299-417`):

1. Loads and validates the exact current assertion (`:302-309`).
2. Normalises the new request; if the idempotency key already exists it **replays** and requires the replay to be the current link (`:319-328`).
3. Fetches the Phase 1 snapshot for the **new** range; any unresolved/blocked day ⇒ `REPLACEMENT_REJECTED` **before any mutation** (`:330-336`).
4. Computes `oldActiveDates` from the movement journal (`:338-343`).
5. `transferDates = oldActiveDates ∩ newDates` **only if room type is unchanged** (`:345-348`); for the example (same type): **transfer = {12,13,14}**, **released = {10,11}**, **newly asserted = {15,16,17}**.
6. `targetDatesNeedingCapacity = newDates \ transferDates` = `{15,16,17}` — capacity is checked **only** for these (`:349,356-366`).
7. Locks the **union** of old and new keys in one sorted pass (`:350-354`) — this is the anti-deadlock measure.
8. Creates the new assertion as `REPLACEMENT_PENDING` with `supersedes_assertion_id` (`:374-379`).
9. Applies `+1` only to non-transferred new dates; writes `ASSERTION` movements for all new dates (`:381-392`).
10. Applies `-1` only to non-transferred old dates; writes `REVERSAL` movements for all old dates (`:393-406`).
11. Old → `SUPERSEDED`, new → `ACTIVE`, `current_assertion_id` → new (`:407-415`).

**Failure behaviour:** any rejection at step 3 or 6 throws **before** step 8 ⇒ the old assertion remains authoritative (confirmed by test *"leaves the old assertion authoritative when replacement capacity is unavailable"*, `reservation-availability-participant.spec.ts:146`).

**Atomicity:** everything happens in the **caller's** transaction — the Reservation date write and the assertion replacement commit or roll back together. **This is exactly the model Phase 3 must adopt.**

### 8.2 Room type change — `STD → DLX`

**VERIFIED FACT:** `normalized.roomType === oldAssertion.room_type` is false ⇒ `transferDates = []` (`:345-347`). Effect: **every** `DLX` date is capacity-checked and `+1` asserted; **every** `STD` date is `-1` released. Net effect on the *property* is neutral only if both room types have slack — otherwise the replacement fails closed and **the reservation keeps `STD` unchanged**.

**OBSERVED BEHAVIOR gap:** the current reservation path (`change-room-type.handler.ts:30`) is a silent no-op, and the Front Office `upgrade-room` path performs only a read-only `isAvailable` guard (`upgrade-room.handler.ts:61`) then writes `room_type` unconditionally (`:74`). **Neither performs a transfer.**

### 8.3 Quantity change — `1 room → 3 rooms`

**BUSINESS DECISION REQUIRED — D-0.** Under reading (a) (quantity always 1) this operation **does not exist**: three rooms are three Reservations, each with its own assertion, and there is no "delta" to compute. Under reading (b) it requires a quantity column, a delta assertion (`+2` per night), and `assertInTransaction`'s hard-coded `1` to be revisited — which touches Phase 2. **Neither may be decided in this step.**

### 8.4 Combined change — `STD 10–15 qty 1` → `DLX 12–18 qty 3`

Under reading (a): this decomposes into three independent Reservation-level replacements, each of which is already atomic. The **cross-reservation** atomicity question (all three or none) is **ARCHITECTURAL DECISION REQUIRED** and is not addressed by any existing artifact.

### 8.5 Where modification stands today

**VERIFIED FACT:** the Reservation-side modification paths (`PUT /:id`, `ChangeRoomTypeCommand`, `ExtendStayCommand`) are all **silent no-ops** (§2.4). The only modification path that reliably reaches legacy inventory is `ExecuteScheduledRoomMoveCommand` (`execute-scheduled-room-move.handler.ts:60` → camelCase `roomType`). So today's "modification" availability effect is effectively limited to scheduled room moves.

---

## 9. Transaction Ownership Analysis

### 9.1 Current transaction map

| Path | Outer tx | Inner tx | Atomic? |
|---|---|---|---|
| `repo.create` | `prisma.$transaction` `:433-514` | — | ✅ reservation side self-contained; **no inventory at all** |
| `repo.update` | `prisma.$transaction` `:557-598` (only if `hasInventoryChanges \|\| hasSimpleUpdates`) | `crs.modifyReservation` runs **before** it, in its own `$transaction` at `crs-engine.service.ts:450,523` | ❌ **legacy inventory commits before the reservation write; outer rollback orphans it** |
| `repo.cancel` | `prisma.$transaction` `:605-630` | `crs.releaseInventory` → own `$transaction` `crs-engine.service.ts:583` | ❌ **non-atomic** |
| `repo.processNoShow` | `:636-656` | same | ❌ **non-atomic** |
| `repo.delete` | `:671-696` | same | ❌ **non-atomic** |
| `repo.batchUpdateStatus` | **none** — single `updateMany` `:723` | — | ❌ no tx |
| auto-cancel sweep | **none** — raw `UPDATE` `:167-171` | — | ❌ no tx |
| FO `check-out` | `prisma.withTransaction` for the departure-date write `:94-108`; failure **swallowed** `:109-111` | — | ⚠️ partial |
| FO `extend-stay` / `upgrade-room` | FO tx | read-only `isAvailable` outside | ⚠️ guard is TOCTOU |
| **Phase 2 port (unwired)** | **caller's tx** — all five methods take `tx: Prisma.TransactionClient` | — | ✅ by design |

**VERIFIED FACT:** `CrsEngineService.releaseInventoryTx` (`crs-engine.service.ts:588-591`) exists precisely to fix the non-atomicity and has **zero callers**.

### 9.2 Ownership conclusion

**ARCHITECTURAL DECISION REQUIRED — D-9.** Two candidate owners:

* **(A) Reservation transaction owns the Availability mutation.** The port is already designed for this (`assertInTransaction(tx, …)`), the Phase 3 foundation tables have composite FKs into `reservations(id, hotel_id)`, and the deferred trigger assumes same-transaction visibility. **This is what the existing artifacts assume.**
* **(B) Availability transaction owns it, driven by an outbox event.** Rejected by evidence: release/assert must be *fail-closed and synchronous* with the state change (a cancel that commits while its release fails is exactly the leak Phase 3 exists to eliminate), and Phase 2 provides no async consumption path.

**INFERENCE:** (A) is the intended design; the decision to record is whether *every* mutation path — including the currently transaction-less `batchUpdateStatus` and auto-cancel sweep — must be promoted to an interactive transaction as a Phase 3 prerequisite.

### 9.3 Lock ordering and deadlock risk

**VERIFIED FACT** — Phase 2 already enforces deterministic ordering:

* Balance upserts in `stay.dates` order (`availability-assertion.service.ts:88-96`).
* Balance locks sorted by `(hotelId, roomType, stayDate)` (`:546-547`).
* Replacement locks the **union** of old and new keys in that same sorted order (`:350-354`), then takes `FOR UPDATE` (`:358-362`).

**INFERENCE:** for Phase 3 the safe global order is **`reservation_availability_operations` claim → `reservation_availability_state` row → assertion balances (sorted) → reservations row**. Any Reservation-side lock taken *before* the sorted balance locks risks deadlock against a concurrent replacement. This ordering must be written into the Phase 3 spec — **it is not currently stated anywhere.**

### 9.4 Multi-date and partial failure

**VERIFIED FACT:** the port's operations are all-or-nothing inside the caller's tx. The journal (`reservation-operation-journal.ts:30-69`) claims with `skipDuplicates: true`, then re-reads and:

* `inserted.count === 1` ⇒ `CLAIMED`
* existing `IN_PROGRESS` ⇒ throws `OPERATION_IN_PROGRESS` (409) — **an abandoned in-progress claim blocks all retries forever**, because nothing ever marks it FAILED.
* existing terminal ⇒ `REPLAY` with the stored result, or `IDEMPOTENCY_CONFLICT` if `request_hash`/`reservation_id`/`operation_type` differ (`:53-55`).

**OBSERVED BEHAVIOR gap:** there is no reconciliation path for `IN_PROGRESS` rows left behind by a crashed process. Since the claim and the completion happen in the *same* transaction, a crash rolls both back — so `IN_PROGRESS` should be unreachable in practice. **INFERENCE:** that invariant depends entirely on claim+complete sharing one tx, and must be stated explicitly.

---

## 10. Idempotency Analysis

### 10.1 What exists

| Mechanism | Location | Status |
|---|---|---|
| Assertion idempotency key + request hash | `availability-assertion.service.ts:60-61,114-117,762-773`; unique `(hotel_id, idempotency_key)` | ✅ built, **engages only if the port is called** |
| Same key + same hash ⇒ replay; different hash ⇒ `IDEMPOTENCY_CONFLICT` (409) | `:768-773` | ✅ matches locked rule 8/9 |
| In-transaction recheck closing the same-key race | `:113-117` | ✅ |
| Durable-record re-read after an ambiguous commit | `:184-195` | ✅ |
| `reservation_availability_operations` journal claim | `reservation-operation-journal.ts:30-69` | ⚠️ built, **never called** |
| Movement uniqueness per `(hotel_id, operation_id, assertion_id, stay_date, movement_type)` | migration `:107-110` | ✅ |
| One active assertion per reservation | migration `:27-31` | ✅ |
| CQRS `IdempotencyPipe` | `common/cqrs/pipes.ts:67-86` | ❌ **dead** — `extends IdempotentCommand` has **zero** matches across `apps/api/src` |
| HTTP `IdempotencyInterceptor` | `core/interceptors/idempotency.interceptor.ts` | ❌ **never registered** in `app.module.ts` |
| Frontend idempotency key / `X-Request-Id` | `apps/web` | ❌ **zero occurrences** |

### 10.2 Operation identity mapping

**VERIFIED FACT — Reservation operations carry no stable operation identifier.**

* Base `Command` generates `commandId = randomUUID()` **per instance** (`common/cqrs/command.ts:19`) — regenerated on every retry ⇒ useless for dedup.
* No reservation command declares `idempotencyKey`/`operationKey`/`requestId`. `correlationId`/`causationId` appear on only three commands (`link-reservations`, `unlink-reservations`, `execute-auto-cancel-sweep`) and are unused for dedup.
* The port *does* define the mapping: `ReservationAssertionRequest.operationId` (a UUID) + `availabilityOperationKey` (≤200 chars) (`reservation-availability.port.ts:17-18`), validated by `validateOperationId` (`availability-assertion.service.ts:490-494`) and journal `validate` (`reservation-operation-journal.ts:109-117`).

**INFERENCE:** the intended mapping is **`operationKey = <one client- or server-issued key per business operation>`**, distinct from Reservation identity and assertion identity — exactly as the pre-existing domain spec's terminology table defines "Operation identity". **What the key is derived from is NOT specified anywhere** ⇒ **ARCHITECTURAL DECISION REQUIRED — D-15.**

Candidate derivations (none adopted):

| Candidate | Pros | Cons |
|---|---|---|
| Client-supplied `Idempotency-Key` header | survives retries at the source | no client sends it today; requires frontend work |
| `(reservationId, operationType, requestHash)` | no client change | two *legitimate* identical operations (cancel → reinstate → cancel) would collide |
| Server-issued UUID returned to the client | clean | requires API change; retries lose it unless echoed |
| Deterministic hash of `(reservationId, operationType, targetState)` | retry-stable | same collision problem as above |

### 10.3 Retry exposure

**VERIFIED FACT (frontend):** `apps/web/components/Providers.tsx:21-23` sets **`mutations: { retry: 1 }`** globally; the Front Office provider duplicates it (`front-office-provider.tsx:8-19`). No reservation hook overrides it. `services/api.ts:38-44` throws on *any* non-2xx and has **no timeout classification**, so a request that **committed server-side but timed out client-side** is re-fired once.

**Therefore: create, confirm, cancel, batch status, check-out, and extend are all currently double-submission-prone**, with no server-side dedup to absorb it. This is the primary idempotency driver for Phase 3.

### 10.4 Per-operation idempotency need

| Operation | Need | Rationale |
|---|---|---|
| create | **CRITICAL** | non-idempotent POST + `retry:1` + no key ⇒ duplicate reservations ⇒ duplicate assertions |
| confirm / guarantee | HIGH | state-setting, retried |
| cancel | **CRITICAL** | release must never double-fire; locked rule 8 |
| no-show | **CRITICAL** | locked rule: "retries must not double-release" |
| modify dates / room type / extend | **CRITICAL** | replacement must replay, not re-execute |
| check-in / check-out | HIGH | check-in must not create a second consumption (locked rule 6) |
| batch / auto-cancel | **CRITICAL** | N rows × retry |
| reinstate | HIGH | must re-evaluate, not replay a stale assert |
| waitlist join / promote | MEDIUM | promote asserts |
| delete | MEDIUM | hard delete is not replayable |

---

## 11. Concurrency Analysis

### Case A — Two reservations race for the last room

**Today:** `InventoryDomainService.reserve` (`:163-172`) does `INSERT … ON CONFLICT DO UPDATE SET reserved=reserved+$3, available=GREATEST(0, available-$3)` — **`GREATEST(0, …)` floors availability at 0**, so oversell is structurally invisible; and a missing date row is inserted with `allot=0, available=0` (`:164-166`) rather than failing.

**Phase 2:** `assertPreparedInTransaction` inserts/locks every balance row `FOR UPDATE` in sorted date order, re-reads, then applies `updateMany({ where: { asserted_quantity: { lte: capacity - quantity } } })` and requires `count === 1` (`:624-633`), else throws. **VERIFIED FACT: this is correct and fail-closed.** The loser's `updateMany` matches 0 rows ⇒ throw ⇒ whole reservation tx rolls back.

**VERIFIED FACT:** the race is only closed if the Reservation's create/confirm actually calls `assertInTransaction`. It does not.

### Case B — A modifies dates while B books the same dates

**Phase 2 handles this:** replacement locks the union of old+new keys sorted (`:350-354`); a concurrent assert locks the same keys in the same sorted order. Both are `READ COMMITTED` with explicit `FOR UPDATE`, so one blocks. **INFERENCE:** safe provided every Phase 3 path acquires balance locks in the canonical sorted order — which is not yet written down (§9.3).

**Risk:** the legacy `InventoryDomainService` and the new assertion engine **do not share locks**. During any co-existence window, a CRS `reserve` and an assertion `assert` can interleave with no mutual exclusion. ⇒ **ARCHITECTURAL DECISION REQUIRED — D-16 (co-existence locking).**

### Case C — Cancel and modify concurrently

**Today:** both would hit `repo.update`/`repo.cancel` with no row-level reservation lock and non-atomic legacy releases ⇒ interleaved partial state.

**Phase 2 design:** both must first `claim` the same `reservation_availability_operations` row? **No** — the claim is keyed by `operation_key`, and a cancel and a modify have *different* keys ⇒ **they do not serialise against each other.** What serialises them is `reservation_availability_state.current_assertion_id` (validated with `ASSERTION_LINK_MISMATCH` at `:257,:303`) plus the partial unique index on active assertions.

**INFERENCE:** the link check is an optimistic concurrency control, not a lock. Two concurrent operations can both read the same `current_assertion_id` and then serialise on the balance `FOR UPDATE`. The loser fails on the link/status check. **This appears correct but is NOT covered by any existing test.** ⇒ **test gap G-1.**

### Case D — Create request retried

**Today:** duplicate `reservations` rows (no unique constraint on anything a retry would repeat — `confirmation_no` is not unique, `id` is client-generated only if supplied).

**Phase 2:** `assert` replays on `(hotel_id, idempotency_key)` (`:60-61`), with an in-transaction recheck (`:114-117`) and a post-throw durable re-read (`:184-195`). Journal claim replays on `(hotel_id, operation_key)`. **VERIFIED FACT: safe — but only if a stable key reaches it (D-15).**

### Case E — Two modifications target the same reservation

Covered by Case C's link check + the `one_active_reservation_uq` index. **Test gap G-1.**

### Case F — Overlap with GBA/Allotment (Phase 4 boundary)

**VERIFIED FACT:** Phase 1 already folds `gbаRemaining + allotmentRemaining` into `consumption` (`snapshot-calculator.ts:14`), so **assertions are already constrained by Phase 4 data**. `availability-source.adapter.ts:65-76,97-102` currently returns `allotmentStopSaleStatus: 'UNRESOLVED'` whenever any allotment stop-sale exists ⇒ **fail-closed**.

**INFERENCE:** a property with active allotment stop-sales will reject all assertions. That is *correct fail-closed behaviour* per locked rule 11, but it means **Phase 3 cannot be validated end-to-end at such a property until Phase 4 resolves stop-sale semantics**. Must be recorded as a readiness risk (§21), not fixed now.

Also **VERIFIED FACT:** `group-allotment` writes only `inventory_policy` — no availability write; `allotment_pickups` are updated by FO checkout (`check-out.handler.ts:334,421`). No reservation path writes GBA counters.

---

## 12. Outbox / Event Analysis

### 12.1 Architecture

**VERIFIED FACT** — `common/events/event-bus.ts:16-47`:

* `IntegrationEvent` ⇒ `outboxWriter.save(message, tx)` — **in-transaction** (`:47`), so the outbox row commits atomically with the mutation when `tx` is passed.
* `DomainEvent` / `InternalEvent` ⇒ synchronous in-process `EventDispatcher.dispatch` (`:28-30`).
* `outbox-writer.ts:41` inserts with status `PENDING`, `maxRetries=5`.
* `outbox-publisher.ts:46+` polls batches of 50, de-duplicates via `EventIdempotencyService`, enqueues BullMQ job `events` with `jobId = message.id`.
* `modules/shared/events.consumer.ts:21` `@Processor('events')`, dedups on `job.id` ⇒ **at-least-once**.

### 12.2 Reservation events

**VERIFIED FACT** — `reservation-domain.events.ts` defines 34 `IntegrationEvent` classes: `ReservationCreated (:3)`, `ReservationCancelled (:26)`, `ReservationNoShow (:45)`, `ReservationUpdated (:63)`, `ReservationCheckedIn (:81)`, `ReservationDeleted (:112)`, `ReservationConfirmed (:130)`, `ReservationGuaranteed (:148)`, `ReservationReinstated (:167)`, `ReservationExtended (:186)`, `ReservationRoomTypeChanged (:205)`, `ReservationRateChanged (:224)`, plus share/party/penalty/link/auto-cancel/room-move/mass-update events.

**Publish sites (all pass `tx`):** `reservation.repository.ts:501,594,629,655,695`; `cancel-reservation.handler.ts:78,102`; `cancel-scheduled-room-move.handler.ts:40`; `attach-accompanying-guest.handler.ts:51`; `add-accompanying-guest.handler.ts:71`; `create-share.handler.ts:68`.

**Consumer** — `events.consumer.ts` switch: `reservation.created (:47)`, `.updated (:50)`, `.cancelled (:51)`, `.no_show (:52)`, `.deleted (:53)`.

**VERIFIED FACT — no consumer adjusts any availability structure.** Events are pure notification today.

### 12.3 Should Availability be mutated synchronously or asynchronously?

**ARCHITECTURAL DECISION REQUIRED — D-17.**

| Option | Evidence for | Evidence against |
|---|---|---|
| **Synchronous in the Reservation tx (port)** | The port's entire signature is `tx`-first; the migration's FKs and deferred trigger assume same-tx visibility; locked rules 5 (multi-date atomic) and 11 (fail closed) are unsatisfiable asynchronously without sagas; decision sheet row 4 "Material Reservation changes use atomic Availability replacement" | Couples Reservation latency to Availability snapshot reads |
| **Asynchronous via outbox** | Reuses existing, tested outbox; decouples | A cancel that commits while its release is pending is exactly the leak Phase 3 exists to prevent; requires compensation/saga design; **no async assertion consumer exists**; Phase 2 exposes no non-transactional API |

**INFERENCE:** synchronous is the intended design (it is what every existing Phase 3 artifact assumes). The decision to record is whether the outbox is used **only for notification/audit** after the synchronous mutation.

**Also required:** whether a new outbox event type (e.g. `reservation.availability_asserted` / `.released` / `.replaced`) is needed for reconciliation (Phase 6) — **MISSING today**.

---

## 13. Property / Tenant Safety Analysis

### 13.1 How property context is derived

**VERIFIED FACT — four partially overlapping mechanisms:**

1. `PropertyScopeGuard` (`apps/api/src/core/guards/property-scope.guard.ts:28-46`) writes `request.propertyScope` — **never read by the reservations module** (0 hits).
2. `TenantInterceptor` (`apps/api/src/core/interceptors/tenant.interceptor.ts:14-37`, registered `app.module.ts:109`) writes AsyncLocalStorage `RequestContext { hotelId, tenantId, userId }`. **This is what reservations uses** via `getHotelId()` (`core/context/request-context.ts:16,24-30`).
3. `MultiTenantGuard` checks `x-tenant-id` only (`multi-tenant.guard.ts:25-33`) — **`x-property-id` is never checked here**.
4. `PropertyAccessGuard` (`common/authorization/resource-access.guard.ts:25-31`) — **`:29 "Property check disabled for development — x-property-id is used for data scoping only"**.

`PartitionRouterMiddleware` (`core/middleware/partition-router.middleware.ts`) is **dead** — only `CorrelationIdMiddleware` is registered (`common.module.ts:57-58`).

### 13.2 Availability's context is stricter

**VERIFIED FACT:**
* `AuthorizedPropertyContext` = `{ hotelId, actorId, actorType, authorizationBasis: 'PROPERTY_ROLE_MEMBERSHIP', correlationId? }` (`availability.contracts.ts:1-7`).
* Sole production constructor: `AvailabilitySnapshotService.authorize` (`availability-snapshot.service.ts:17-34`) — **rejects empty/`'default'` hotelId (`:18`), loads `platformUserRole` scoped to the requested `propertyId` (`:24-27`), requires an availability/reservations permission (`:28-32`), freezes the context (`:33`)**.
* `AvailabilityController` (`:13-16,:21-24`) resolves `actorId` from the JWT and calls `authorize`; there are **no `@Permissions`/`@PropertyScope` decorators** on it.
* `AvailabilityAssertionService.validateContext` (`:472-476`) re-checks `authorizationBasis === 'PROPERTY_ROLE_MEMBERSHIP'` and rejects `'default'`; `validateReservationInput` (`:478-488`) requires `input.context.hotelId === input.hotelId`.

**INFERENCE:** the Availability side performs a genuine membership check that the Reservation side does not. Wiring the port will therefore **import the stricter check into the reservation path** — a change of effective authorization behaviour that must be called out in Stage A.

### 13.3 Findings — unsafe fallbacks (documented, NOT fixed)

| ID | Location | Behavior | Impact on Availability |
|---|---|---|---|
| **F-1** | `create-reservation.handler.ts:23` — `const hotelId = command.hotelId \|\| command.data.hotel_id \|\| 'default';` | Request body can place a reservation in any tenant, or in `'default'` | `'default'` rows are **invisible** to every real property's consumption scan (`reservation-consumption.adapter.ts:15` filters `hotel_id`), yet Availability **hard-rejects** `'default'` (`availability-snapshot.service.ts:18,37`) ⇒ phantom capacity + unassertable reservation |
| **F-2** | `reservation.repository.ts:424` — `dbData.hotel_id = dbData.hotel_id \|\| hotelId;` with `create-reservation.dto.ts:4 @IsOptional() hotel_id?` | Body wins over context | same as F-1, arbitrary cross-tenant write |
| **F-3** | `mass-update-execute.handler.ts:30-48` | `mass_update_jobs` loaded by id, **`job.hotel_id` never compared to `getHotelId()`** | Executes another tenant's mass update, including its inventory/rate guard |
| **F-4** | `reservation.repository.ts:100,329,521,663,703` — `if (hotelId) where.hotel_id = hotelId` | If ALS is empty (background/cron/public path), reads and `updateMany` become **global** | `batchUpdateStatus` (`:723`) is the highest blast radius |
| **F-5** | `reservation.repository.ts:63` — `const hotelId = getHotelId() \|\| params.propertyId;` | Both may be undefined ⇒ `findAll()` unscoped | cross-tenant read |
| **F-6** | `reservations.service.ts:428,457,464` | `reservation_penalties` read/**settle** raw SQL with `hotelId` fetched and never bound | cross-tenant penalty settlement (write) |
| **F-7** | `prisma-billing.adapter.ts` (12 statements), `prisma-front-office-handoff.adapter.ts:11,31,47`, `prisma-guest-profile.adapter.ts:10` | Keyed only by `reservation_id`/`folio_id`; guest lookup by email | cross-tenant PII / folio access |
| **F-8** | `permission-resolver.service.ts:15-16` + `permission-cache.service.ts:17-19` | Permission cache key computed from ALS, but **interceptors run after guards** ⇒ during `PermissionGuard` ALS is empty ⇒ key is `default:permission:user:<id>` | property-scoped `@Permission` enforcement unreliable for the 300 s TTL |
| **F-9** | `property-scope.guard.ts:35`, `tenant.interceptor.ts:19-23` | A differing `x-property-id` **overrides** the user's own hotel with no membership check | client-controlled tenant for all reservation writes |
| **F-10** | `correlationId` — `availability.controller.ts:13,21` reads `@Headers('x-correlation-id')`; `correlation-id.middleware.ts:8-10` mints a UUID but never propagates it back | If the client omits the header, `context.correlationId` is `undefined` though a UUID exists on `req.correlationId` | audit/reconciliation correlation is client-optional |

**OBSERVED BEHAVIOR — `hotel_id='default'` is explicitly rejected by Availability but produced by Reservation.** This is a direct Reservation↔Availability incompatibility, not merely a security note.

**No finding was fixed during this audit.**

---

## 14. Legacy Availability Interaction

### 14.1 Every Reservation dependency on legacy availability

| # | Dependency | Location | Behavior | Classification |
|---|---|---|---|---|
| L-1 | `availability` table reserve (create path) | `crs-engine.service.ts:390` ← **only from `rates-inventory.controller.ts:107`**, not from `POST /reservations` | `INSERT … ON CONFLICT DO UPDATE SET reserved+1, available=GREATEST(0,available-1)` | **MIGRATE** — becomes an assertion; two booking paths must converge |
| L-2 | `availability` table release (cancel/no-show/delete) | `reservation.repository.ts:609,640,673` → `crs-engine.service.ts:581-586` | release full `[arrival, departure)`, `rooms:1` | **REPLACE** with `port.releaseInTransaction` |
| L-3 | `availability` table diff modify | `reservation.repository.ts:546` → `crs-engine.service.ts:429-537` | reserve new / release old / reserve added | **REPLACE** with `port.replaceInTransaction` |
| L-4 | `InventoryDomainService.isAvailable` read guard | FO `extend-stay.handler.ts:58`, `upgrade-room.handler.ts:61` | read-only TOCTOU guard | **REPLACE** with an assertion-aware eligibility read |
| L-5 | `InventoryDomainService.blockAvailability/consumePickup/releaseUnsold/reserveRooms/releaseRooms` | `inventory.domain-service.ts:208,237,265,189,197` | **zero callers** | **REMOVE** (candidate, Phase 6) |
| L-6 | `CrsEngineService.releaseInventoryTx` | `:588-591` | **zero callers** (the tx-correct variant) | **REMOVE** once L-2 migrates |
| L-7 | `InventoryReservationPort` + `PrismaInventoryReservationAdapter` | `application/ports/inventory-reservation.port.ts`; `infrastructure/adapters/prisma-inventory-reservation.adapter.ts` | `holdInventory/confirmHold/releaseHold` are **no-op stubs**; `checkAvailability` uses `$queryRawUnsafe` counting `rooms WHERE status='AVAILABLE'`; **never injected** | **REMOVE** (candidate, Phase 6) |
| L-8 | `InventoryDomainService` injected into `ReservationRepository` but unused | `reservation.repository.ts:11,57` | referenced only at injection | **REMOVE** (candidate) |
| L-9 | Phantom `SELECT available_count FROM inventory` | `mass-update-execute.handler.ts:191-196` | table/column **do not exist** ⇒ always fails; skippable via `overrideAvailableInventory` `:72-78` | **REMOVE / REPLACE** |
| L-10 | `room_inventory` model (`schema.prisma:10413`) | read at `activities/availability-sales.controller.ts:34,223` | **no writer anywhere** | **KEEP read-only or REMOVE** (Phase 6) |
| L-11 | Channel mirror `channel_availability_log` / `channel_availability` | `channels/channels.service.ts:126,152,175,187` | writes a log; **never pushes into `availability`** | **KEEP** (Phase 8) |
| L-12 | Restrictions/sell-limits writers | `activities/availability-sales.controller.ts:575-781` | writes restriction tables consumed by Phase 1 | **KEEP** — Phase 1 authority |
| L-13 | `group_pickups` / `allotment_pickups` writes at FO checkout | `front-office/…/check-out.handler.ts:334,421` | pickup counters | **KEEP until Phase 4** |
| L-14 | `ReservationConsumptionAdapter` (Phase 1) | `availability/…/reservation-consumption.adapter.ts` | counts `reservations` per date; fail-closed on quantity | **MISSING (population-awareness)** — see B-1/B-2 |
| L-15 | `availability_assertion_balances` | Phase 2 | authoritative assertion balance | **KEEP** — new authority |
| L-16 | **Assertion engine** (`assert/release/replace/journal/state`) | `availability-assertion.service.ts`, `reservation-operation-journal.ts`, Phase 3 migration | built, unwired | **MISSING (wiring)** |
| L-17 | `AvailabilityReconciliationService` | `availability/…/availability-reconciliation.service.ts:17,23` | compares legacy `availability.available` vs canonical snapshot; legacy marked `authoritative:false` | **KEEP** — Phase 6 input |
| L-18 | Outbox consumers touching availability | `modules/shared/events.consumer.ts:47-53` | **none** | **MISSING** if async notification is chosen |

### 14.2 Summary counts

**KEEP 5 · MIGRATE 1 · REPLACE 4 · REMOVE 7 · MISSING 3** (L-14 population-awareness, L-16 wiring, L-18 optional async notification).

**Nothing was removed, migrated, or replaced during this audit.**

---

## 15. Frontend Mutation Analysis

Scope limited to frontend behavior that affects Reservation **lifecycle semantics**. No redesign proposed.

### 15.1 Which system actually mutates

| System | Role | Verdict |
|---|---|---|
| React Query `apps/web/features/reservations/hooks/use-reservation-mutations.ts` | issues every live reservation lifecycle mutation except checkout | **AUTHORITATIVE** |
| Zustand `apps/web/store/reservationStore.ts` | 11 mutation actions; **only `checkOutReservation` has a consumer** (`CheckoutFlowModal.tsx:112,582`) | mostly dead |
| Zustand `features/reservations/store/reservation-ui-store.ts` | selection/filters/dialog-open; `actionLoading` is **never set** | render-only |
| `apps/web/components/reservations/*` modals | `ReservationDetailModal`, `EditReservationModal` **not imported anywhere** | dead for reservations |
| `apps/admin/store/reservationStore.ts` | pure local `set()` on mock rows; only admin network call is a **read** | cannot affect availability |

### 15.2 Mutation entry points

| Lifecycle operation | Route | Trigger | Guard |
|---|---|---|---|
| create | `POST /reservations` | `QuickBookForm.tsx:168,457,1067,1187` | submit guard present |
| modify dates | `PUT /reservations/:id` `{arrival_date, departure_date}` | `ChangeDatesDialog.tsx:121-124` | `isPending` ✅ |
| modify room type | `PUT /reservations/:id` `{room_type}` | `ChangeRoomTypeDialog.tsx:105-108` | `isPending` ✅ |
| modify quantity | — | field collected in `QuickBookForm`, **never sent** | n/a |
| cancel | `POST /:id/cancel` | `CancellationDialog.tsx:106-115` | `isPending` ✅ + confirm checkbox |
| batch cancel | `POST /batch/status` `{ids,status:'cancelled'}` | `MassUpdateDialog.tsx:36-46` | `isPending` ✅ |
| no-show | **`PUT /:id` `{reservation_status:'NO-SHOW'}`** | `ReservationsHomeView.tsx:90-92`, `ArrivalPanel.tsx:94` | ❌ `actionLoading={null}` hard-coded |
| confirm | `POST /:id/confirm` | `ReservationsHomeView.tsx:162-166` | ❌ unguarded |
| reinstate | `POST /front-office/reinstate` | `ReservationDetailWorkspace.tsx:73-78` | ❌ unguarded |
| check-in | deep-link to Front Office (`model/reservation-actions.ts:124-131`) | `ReservationsHomeView.tsx:167-169` | n/a (FO dialog guarded) |
| check-out | `POST /front-office/check-out` | `CheckoutFlowModal.tsx:582` | ✅ `processing` flag |
| extend | `POST /front-office/:id/extend` | `CheckoutFlowModal.tsx:606` | ❌ **unguarded** |
| waitlist join / promote | `POST /:id/waitlist`, `/waitlist/promote` | `WaitlistDialog`, `WaitlistQueue` | mixed |
| delete / guarantee / mass-update execute | exported hooks | **no UI consumer** | n/a |

### 15.3 Findings affecting lifecycle semantics

| ID | Finding | Evidence | Availability relevance |
|---|---|---|---|
| **W-1** | **Global `mutations: { retry: 1 }`** on a non-classified fetch client with no timeout handling | `Providers.tsx:21-23`, `services/api.ts:38-44` | any post-commit failure ⇒ duplicate create/cancel/batch/check-out |
| **W-2** | **No idempotency key anywhere** in `apps/web`/`apps/admin` | grep `idempotency\|X-Request-Id` ⇒ 0 hits | server has nothing to dedup on |
| **W-3** | **Wrong no-show route**: frontend uses `PUT /:id`, backend command is `POST /:id/no-show`; the PUT path is a silent no-op (§2.4) | `reservation.api.ts:186`, `front-office.api.ts:213` | no-show release is **unreachable from the UI** |
| **W-4** | **Action-routing bugs**: detail workspace maps `noShow → openCancellation`; queue `handleAction` has no `noShow`/`reinstate`/`guarantee` case; `guarantee` opens the deposit dialog instead of `POST /:id/guarantee` | `ReservationDetailWorkspace.tsx:108-113`, `ReservationsHomeView.tsx:158-195` | lifecycle transitions fired from the UI don't match the backend commands |
| **W-5** | **Dead submit guards**: `ReservationsHomeView.tsx:133` passes `actionLoading={null}`, so `ReservationDataTable.tsx:114,119` never disable | as cited | double cancel / double no-show |
| **W-6** | **Three field dialects** for the same `PUT /:id`: snake (`ChangeDatesDialog:122`), camel (`reservationStore.ts:487-489`), status-only (`reservation.api.ts:186`) | as cited | only the snake dialect is sent; all are read as camel ⇒ no-ops |
| **W-7** | **Client-side availability display only** — `AvailableRatesMatrix`, `AvailabilityPage`, `getRoomTypes().availableRooms` are read-only; **no availability-derived quantity is ever sent** | grep confirms | frontend is **not** a source of availability logic (correct) |
| **W-8** | **Client-side TOCTOU gate** in dead `use-crs-book.ts:112` (`if (!quote.available) return`) before `POST /rates/engine/book` | as cited | must never become a live pattern |
| **W-9** | Missing `x-property-id` ⇒ header simply omitted ⇒ server falls back to `'default'` | `services/api.ts:30`, `middleware.ts:31` | interacts with F-1 |
| **W-10** | Two independent implementations of checkout (Zustand + React Query FO hooks) | `CheckoutFlowModal.tsx:582` vs `front-office.api.ts:119` | duplicate-submit surface |

**Conclusion:** the frontend must **not** become the source of Availability business logic — and today it is not (W-7). The required frontend work for Phase 3 is limited to **W-1/W-2/W-5 (idempotency and guards)** and **W-3/W-4/W-6 (route and dialect correctness)** so that the real backend lifecycle paths are reachable. That is a prerequisite track, not a redesign.

---

## 16. KEEP / MIGRATE / REPLACE / REMOVE / MISSING

Consolidated classification (detail in §14.1):

### KEEP
1. Phase 1 snapshot pipeline and `SnapshotCalculator` (`snapshot-calculator.ts`) — locked.
2. Phase 2 assertion engine, balances, movements, idempotency, lock ordering — locked.
3. `reservation_availability_state` / `reservation_availability_operations` schema + deferred link trigger + partial unique indexes — Phase 3 foundation, already migrated.
4. `ReservationStatusService` as the single transition authority (`reservation-status.service.ts:20-27`).
5. Outbox publisher/consumer pipeline for notification and audit.
6. `AvailabilityReconciliationService` — Phase 6 input.
7. Restriction/sell-limit writers consumed by Phase 1 (`activities/availability-sales.controller.ts:575-781`).
8. React Query as the single frontend mutation system.

### MIGRATE
1. `CrsEngineService.confirmReservation` reserve path — the *only* real reserve today, and it lives on a **different** route (`rates-inventory.controller.ts:107`). Must converge onto `POST /reservations` + `port.assertInTransaction`.
2. `ReservationConsumptionAdapter` — must become **population-aware** (exclude `ASSERTION_MANAGED`) and status-aligned with `findCurrentInTransaction`.

### REPLACE
1. `crs.releaseInventory` call sites (`reservation.repository.ts:609,640,673`) → `port.releaseInTransaction`.
2. `crs.modifyReservation` call site (`reservation.repository.ts:546`) → `port.replaceInTransaction`.
3. FO `inventoryDomain.isAvailable` guards (`extend-stay.handler.ts:58`, `upgrade-room.handler.ts:61`) → assertion-aware eligibility.
4. Frontend no-show route and dialect (W-3/W-6) → the real `POST /:id/no-show` command.

### REMOVE *(candidates only — Phase 6, not now)*
`inventory.domain-service.ts` dead methods (`:189,:197,:208,:237,:265`) · `releaseInventoryTx` (`crs-engine.service.ts:588`) · `InventoryReservationPort` + adapter · unused `InventoryDomainService` injection (`reservation.repository.ts:11,57`) · phantom `SELECT available_count FROM inventory` (`mass-update-execute.handler.ts:191-196`) · `PartitionRouterMiddleware` · HTTP `IdempotencyInterceptor` *or* register it · CQRS `IdempotencyPipe` *or* delete it.

### MISSING *(the Phase 3 gap list)*
| ID | Missing piece | Blocking? |
|---|---|---|
| M-1 | Every Reservation handler injecting and calling `RESERVATION_AVAILABILITY_PORT` | **yes** |
| M-2 | `ReservationOperationJournal.claim/complete/reject` invocation in each handler | **yes** |
| M-3 | `setPopulationInTransaction` on create (and a backfill/cutover story for existing rows) | **yes** |
| M-4 | Population-aware, status-aligned `ReservationConsumptionAdapter` | **yes** (B-1) |
| M-5 | Resolvable quantity (D-0) | **yes** |
| M-6 | A stable operation key reaching the port (D-15) | **yes** |
| M-7 | Transaction promotion for `batchUpdateStatus` and auto-cancel sweep | **yes** |
| M-8 | Fix or route around the six silent no-op handlers (§2.4) | **yes** for confirm/guarantee/reinstate/extend/room-type |
| M-9 | Frontend idempotency key + retry policy (W-1/W-2) | high |
| M-10 | New outbox event(s) for assertion lifecycle (reconciliation/Phase 6) | medium |
| M-11 | Availability write endpoint or an explicitly internal-only port (currently neither is exercised) | medium |
| M-12 | `IN_PROGRESS` journal reconciliation policy | medium |
| M-13 | `CURRENT_ASSERTION_MISSING` / `RESERVATION_POPULATION_MISMATCH` handling in Reservation error mapping | medium |

---

## 17. Open Business Decisions

### 17.1 Conflicts between the brief and the repository (resolve first)

| ID | Conflict | Both sides | Proposed Stage A handling |
|---|---|---|---|
| **C-0** | **Phase 3 stage position.** Brief says *forensic audit ahead of the decision sheet*; repo contains a decision sheet marked "All decisions resolved" + a draft domain spec + a Phase 3 DB migration, all untracked. | Brief §1–§4 vs `availability-phase3-business-rules-decision-sheet.md:3,121`, `availability-phase3-domain-specification.md:8`, `packages/db/migrations/20260928000000_…` | Do **not** silently adopt or discard. Establish provenance, then either *ratify as Stage B input* or *quarantine as draft*. |
| **C-1** | **Room quantity.** Brief: unresolved, must not be guessed. Decision sheet `:31`: "quantity is always 1". Reservation spec `03_PROCEDURES.md:252,338,430`: "number of rooms"/"room quantity" is a required input and an inventory-impacting change. `07_DECISIONS.md:1269`: "One reservation maps to one room." | three sources, three readings | D-0, blocking — see §5.6 |
| **C-2** | **No-show release range.** Locked rule: release *remaining future nights, not the elapsed arrival night*. Current code: releases `[arrival, departure)` wholesale. | `crs-engine.service.ts:581-586,593-603` vs decision sheet `:37` | D-3, blocking for no-show |
| **C-3** | **Overstay ordering.** Decision 5: secure the night *before* committing extended dates. Current code: commits `departure_date` first, swallows the failure. | `check-out.handler.ts:94-111` vs decision sheet `:77-88` | D-6, blocking for extension |
| **C-4** | **Hard delete of active reservations.** Decision sheet row 12: "Active Reservations are not hard-deleted." Current code: `repo.delete` hard-deletes any non-`CHECKED_IN` reservation. | `reservation.repository.ts:660-697` vs decision sheet `:43` | D-11 |
| **C-5** | **Consuming-status set.** Three different sets (§6.3). | adapter `:5` vs foundation `:435` vs decision sheet `:35` | D-1, blocking |
| **C-6** | **Documentation governance.** Project instructions call the Reservations specs "locked"; `docs/audit/Reservations/XYLO MASTER SPECIFICATION.md` labels itself `DRAFT — NOT LOCKED` and `03_PROCEDURES.md` `DRAFT FOR REVIEW` (already noted by the pre-existing decision sheet `:115`). | as cited | record; do not resolve in this step |
| **C-7** | **Phase 1/2 documentation absence.** No Phase 1 or Phase 2 specification document exists anywhere under `docs/`. Only code + tests. | grep across `docs/` | record; Phase 3's authority hierarchy depends on them |

### 17.2 Business decisions required

| ID | Decision | Why code cannot decide it |
|---|---|---|
| **D-0** | Reservation room quantity model (blocking) | §5.6 |
| **D-1** | Consuming-status set + assertion creation point (incl. PENDING, PROSPECT, status spelling) | §6.3 |
| **D-2** | Exact release date set on cancel | partial-release semantics |
| **D-3** | No-show release range (future nights only?) | conflicts with current code (C-2) |
| **D-4** | Batch operation atomicity: per-reservation atomic with explicit partial results vs all-or-nothing | brief requires an explicit choice |
| **D-5** | Early-departure release timing and its atomicity with the FO checkout transaction | §4 row 9 |
| **D-6** | Overstay/extension ordering (assert-then-commit) | conflicts with current code (C-3) |
| **D-7** | Date-change transfer/overlap policy | already implemented in Phase 2; needs ratification |
| **D-8** | Room-type upgrade: assertion replacement for *operational/complimentary* upgrades too? | decision sheet row 13 |
| **D-10** | Reinstatement: re-evaluate then restore, atomically | no `unrelease` exists |
| **D-11** | Delete policy for assertion-managed reservations | FK is `RESTRICT` (C-4) |
| **D-12** | Waitlist promotion capacity gate | currently none |
| **D-13** | Mass-update inventory gate | phantom query (L-9) |
| **D-14** | Group/block assertion boundary → Phase 4 confirmation | §4 row 28 |
| **D-19** | Which reservation events must emit new outbox messages | §12.3 |
| **D-20** | Legacy `availability` counter: authoritative, projection, or retired at cutover | reconciliation decides Phase 6 |

### 17.3 Architectural decisions required

| ID | Decision |
|---|---|
| **D-9** | Transaction ownership: Reservation tx owns the Availability mutation (A) vs outbox saga (B) — see §9.2 |
| **D-15** | Derivation of the stable operation key — see §10.2 |
| **D-16** | Co-existence locking between the legacy `availability` counter and assertion balances — see §11 Case B |
| **D-17** | Synchronous mutation + outbox-for-notification vs asynchronous consumption — see §12.3 |
| **D-18** | Command-name namespace strategy / merge of `ExtendStayCommand` and `ReinstateReservationCommand` — see §2.3 |
| **D-21** | Population assignment strategy for existing reservations: per-operation lazy assignment vs a controlled backfill — see B-2 |
| **D-22** | Whether Availability exposes any write endpoint or remains an internal port only |

### 17.4 Prerequisite (non-Availability) defects triaged as Phase 3 inputs

Not fixes — **triage decisions only** (prerequisite / parallel track / ignore):

* **P-1** Six silent no-op handlers + snake/camel DTO-repo mismatch (§2.4).
* **P-2** Frontend wrong no-show route and action-routing bugs (W-3, W-4).
* **P-3** F-1/F-2 `'default'` and body-controlled `hotel_id` on create.
* **P-4** F-3 mass-update cross-tenant job execution.
* **P-5** `batchUpdateStatus` / auto-cancel sweep with no transaction and no release.
* **P-6** `reinstate-reservation.handler.ts:27` gating on `'NO_SHOW'` vs stored `'NO-SHOW'`.
* **P-7** CQRS `IdempotencyPipe` dead + HTTP `IdempotencyInterceptor` unregistered.
* **P-8** Non-atomic legacy release (`releaseInventoryTx` unused).

---

## 18. Proposed Phase 3 Business-Rules Decision Sheet

> **Status: PROPOSAL ONLY — not a decision record.** To be reconciled with the pre-existing sheet in Stage B (C-0).

Suggested structure, mirroring the existing sheet's format:

| # | Decision | Proposed status | Source of truth | Blocking? |
|---:|---|---|---|---|
| 1 | PENDING consumption | **OPEN (D-1a)** — three-way conflict C-5 | code vs code vs sheet | **yes** |
| 2 | No-show timing / release range | **OPEN (D-3)** — conflict C-2 | sheet `:37` vs `crs-engine.service.ts:581` | **yes** |
| 3 | Early departure | proposed RESOLVED as sheet Decision 3 | sheet `:45-59` | no (needs C-3 check) |
| 4 | Checkout release | proposed RESOLVED as sheet Decision 4 | sheet `:61-73` | no |
| 5 | Overstay ordering | **OPEN (D-6)** — conflict C-3 | sheet `:75-88` vs `check-out.handler.ts:94-111` | **yes** |
| 6 | Reinstatement | proposed RESOLVED as sheet row 6 | sheet `:41` | no |
| 7 | Multi-room grouping | **= D-0, OPEN** — conflict C-1 | §5.6 | **yes** |
| 8 | Assertion linkage | proposed RESOLVED — encoded in migration + port | migration `:27-31`, `availability-assertion.service.ts:496-507` | no |
| 9 | Idempotency ownership | **OPEN (D-15)** — key derivation unspecified | §10.2 | **yes** |
| 10 | Legacy counter cutover | deferred to Phase 6 (D-20) | §14 | no (Phase 3) |
| 11 | Batch atomicity | proposed: atomic per Reservation with explicit partial results (per brief §12) — needs ratification | brief | no |
| 12 | Active Reservation delete | **OPEN (D-11)** — conflict C-4 | sheet `:43` vs `repo.delete:660-697` | no |
| 13 | Complimentary/operational upgrade | **OPEN (D-8)** | sheet `:42` | no |
| **14 (new)** | Consuming-status set + status-spelling normalisation | **OPEN (D-1b, D-1c, D-1d)** | §6.3 | **yes** |
| **15 (new)** | Waitlist promotion capacity gate | **OPEN (D-12)** | §4 row 20 | no |
| **16 (new)** | Population assignment for pre-existing reservations | **OPEN (D-21)** | §3.3 | **yes** |
| **17 (new)** | Group/block assertion boundary belongs to Phase 4 | proposed CONFIRMED | brief §3 roadmap | no |

**Fixed contract (unchanged, carried forward):** departure exclusive; one active assertion per Reservation linked by `(hotelId, RESERVATION, Reservation.id)`; release/replacement targets the exact `assertionId`; Availability authoritative for room-type inventory; material changes use atomic replacement; released assertions are not reusable; multi-date atomicity; deterministic lock ordering; durable idempotency; tenant isolation mandatory; unresolved capacity fails closed.

---

## 19. Proposed Locked Domain Specification Outline

> **Status: PROPOSAL OUTLINE.** A draft v0.1 already exists (`availability-phase3-domain-specification.md`, 50 662 bytes). The outline below is what a *locked* version must additionally pin down, and is offered for reconciliation in Stage C.

1. **Document control & authority hierarchy** — must name Phase 1/2 specifications, which **do not exist in the repo** (C-7).
2. **Scope / non-scope** — unchanged from the draft; explicitly excludes GBA/Allotment (Phase 4), CRS authority (Phase 8), physical room state.
3. **Domain ownership table** — Reservations owns lifecycle; Availability owns commitment; Front Office owns operational transitions; Rooms/Housekeeping physical; Billing financial; Night Audit no-show.
4. **Reservation inventory model** — **must resolve D-0**; state whether quantity is fixed at 1 or modelled.
5. **Availability commitment model** — one active assertion per Reservation; assertion = commitment, not physical room identity.
6. **Reservation ↔ Availability contract** — linkage, `findCurrent` validation, population states.
7. **Consuming & non-consuming contexts** — **must resolve D-1** with an explicit status-set table including every storage spelling (`RESERVED`, `PROSPECT`, `IN_HOUSE`, `NO-SHOW`, `CHECKED_OUT`).
8. **Per-operation effects** — one subsection per row of the matrix in §4, each stating: trigger, required port call, date set, failure outcome.
9. **Modification semantics** — dates / room type / quantity / combined, with the worked examples from §8.
10. **Check-in / check-out semantics** — no second consumption; no release at normal checkout; early departure and overstay as distinct operations.
11. **Cancel / no-show / reinstatement semantics** — **must resolve D-3 and D-10**.
12. **Atomicity & transaction ownership** — **must resolve D-9**; must state the global lock acquisition order (§9.3).
13. **Idempotency** — **must resolve D-15**; state claim/complete share one tx (§9.4) and the `IN_PROGRESS` policy (M-12).
14. **Concurrency** — the six cases in §11 as normative scenarios; **must resolve D-16**.
15. **Failure & recovery** — fail-closed matrix; `CURRENT_ASSERTION_MISSING`, `RESERVATION_POPULATION_MISMATCH`, `ASSERTION_LINK_MISMATCH`, `REPLACEMENT_REJECTED` handling (M-13).
16. **Audit & events** — **must resolve D-17/D-19**.
17. **Tenancy** — mandatory property scoping; explicit prohibition of `'default'` (F-1/F-2 must be named as violations of this rule).
18. **Legacy boundaries** — KEEP/MIGRATE/REPLACE/REMOVE/MISSING from §14 as normative.
19. **Out of scope / deferred** — Phase 4/5/6 items.
20. **Traceability** — every normative statement tagged to a decision-sheet row and an evidence citation.

---

## 20. Proposed Phase 3 Implementation Plan Outline

> **Status: PROPOSAL OUTLINE only.** No implementation is authorised in this step.

**Stage 0 — Blocker clearance (no code until these close)**
* B-1 Resolve D-0 (quantity) and align `ReservationConsumptionAdapter` (M-4) — *Availability-side change, must be re-validated against Phase 1/2 lock.*
* B-2 Population assignment strategy (D-21) + `setPopulationInTransaction` on create (M-3).
* B-3 Status-set alignment (D-1 / C-5).
* B-4 Operation key derivation (D-15 / M-6).
* B-5 Prerequisite triage of P-1 … P-8.

**Stage 1 — Foundation wiring (vertical slice, create)**
1. Inject `RESERVATION_AVAILABILITY_PORT` + `ReservationOperationJournal` into `CreateReservationHandler` path.
2. `journal.claim(ASSERT)` → `setPopulation(ASSERTION_MANAGED)` → `assertInTransaction` → `journal.complete`, all inside the existing `repo.create` transaction (`reservation.repository.ts:433-514`).
3. Map `REJECTED` to a domain error so a failed assertion rolls back the reservation.
4. Tests: idempotent replay, conflict, fail-closed rejection, cross-property rejection, rollback equivalence.

**Stage 2 — Release paths**
1. cancel → `releaseInTransaction(full stay)`.
2. no-show → `releaseInTransaction(future nights)` per D-3.
3. delete → disallow or release-first per D-11.
4. Promote `batchUpdateStatus` and the auto-cancel sweep to per-reservation interactive transactions (M-7).
5. Fix P-8 (`releaseInventoryTx` / drop legacy release) as part of the migration, not as a separate refactor.

**Stage 3 — Replacement paths**
1. Date change, room-type change, extend, shorten via `replaceInTransaction`.
2. Requires P-1 (the six no-op handlers) to be resolved so the commands actually persist.
3. FO `extend-stay` / `upgrade-room` / early-departure checkout routed through the same port with assert-before-commit ordering (D-6).
4. `ExecuteScheduledRoomMoveCommand` — same-type ⇒ no assertion change; type change ⇒ replace.

**Stage 4 — Cross-cutting**
1. Frontend idempotency key + retry policy (M-9, W-1/W-2).
2. Frontend route/dialect fixes (W-3/W-4/W-6).
3. Command-name namespace (D-18).
4. Property-safety findings F-1…F-10 triaged (some are hard prerequisites for safe cutover).
5. New outbox events (M-10) if D-17 chooses notification.

**Stage 5 — Validation**
* Availability test suite extended; unit tests; PostgreSQL integration tests (pattern already exists: `reservation-availability-foundation.postgres.spec.ts`); typecheck; build; lint.
* Concurrency tests for Cases A–E (G-1).
* Property-scope regression for every Reservation → Availability path.

**Exit gates (proposed):** all of M-1…M-13 closed; all blocking decisions (D-0, D-1, D-3, D-6, D-9, D-15, D-21) resolved and recorded; C-0 reconciled; no new fail-closed regression at a clean test property.

---

## 21. Readiness Risks / Blockers

### Blockers (must close before any Phase 3 implementation)

| ID | Blocker | Evidence | Consequence if ignored |
|---|---|---|---|
| **B-0** | **State conflict C-0** — unclear whether Stage B/C already happened | §1.3 | Duplicate or contradictory decision records |
| **B-1** | **Phase 1 fail-closed on any consuming reservation** + non-population-aware adapter + default `RESERVED` status | `reservation-consumption.adapter.ts:20-27`, `availability-snapshot.service.ts:80,100-102`, `schema.prisma:9579` | every assertion rejected; Phase 3 untestable in any realistic property |
| **B-2** | **No population assignment exists** | `assertInTransaction` `:204-206`; no production caller of `setPopulationInTransaction` | `RESERVATION_POPULATION_MISMATCH` on every operation |
| **B-3** | **Consuming-status set contradiction (C-5)** | §6.3 | assertions created/required at the wrong lifecycle point |
| **B-4** | **No operation key reaches the port (D-15)** | §10.2 | retries double-assert or conflict |

### High risks

| ID | Risk | Evidence |
|---|---|---|
| **R-1** | Allotment stop-sales force `UNRESOLVED` ⇒ assertions rejected at any property using allotments | `availability-source.adapter.ts:97-102` — correct fail-closed, but blocks Phase 3 validation until Phase 4 |
| **R-2** | Legacy counter and assertion balances do not share locks during co-existence | D-16, §11 Case B |
| **R-3** | Six silent no-op handlers ⇒ the operations Phase 3 must instrument do not persist | §2.4 |
| **R-4** | Frontend `retry:1` + no idempotency key ⇒ duplicate operations | W-1, W-2 |
| **R-5** | Property fallbacks F-1/F-2 write reservations into `'default'`, which Availability rejects | `create-reservation.handler.ts:23` vs `availability-snapshot.service.ts:18` |
| **R-6** | `batchUpdateStatus` and auto-cancel sweep: no transaction, no release, conditional tenant scoping | `reservation.repository.ts:699-725`, `execute-auto-cancel-sweep.handler.ts:167-171`, `:703` |
| **R-7** | Command-name collisions make it ambiguous which handler owns extension/reinstate | §2.3 |
| **R-8** | Non-atomic legacy release already exists (independent of Phase 3) — `releaseInventoryTx` unused | `crs-engine.service.ts:581-591` |
| **R-9** | No Phase 1/2 specification documents exist to anchor the authority hierarchy | C-7 |
| **R-10** | Concurrent modification vs cancel is untested at the assertion-link level | G-1, §11 Case C/E |

### Test gaps to close in Phase 3

**G-1** concurrent replace vs release on the same assertion · **G-2** population assignment + rollback · **G-3** journal `IN_PROGRESS` recovery · **G-4** batch partial results · **G-5** cross-property assertion rejection through a real Reservation command · **G-6** no-show future-nights-only release.

---

## 22. Final Recommendation

**The project is ready to move to Stage A — Review the audit — but it is NOT ready to move directly to the Business-Rules Decision stage.**

Reasons:

1. **C-0 must be settled first.** The repository already contains a Phase 3 business-rules decision sheet marked fully resolved, a Phase 3 domain specification draft, and a Phase 3 database migration. Either these are ratified as the Stage B input (in which case the work is *review and ratify*, not *author*) or they are quarantined as drafts. This cannot be assumed, and it changes what Stage B means.

2. **Four blocking decisions are genuinely open and cannot be inferred from code:** D-0 (room quantity — explicitly forbidden to resolve here), D-1 (consuming-status set — three conflicting sources), D-3 (no-show release range — code contradicts the locked rule), D-6 (overstay ordering — code contradicts Decision 5). Each needs a business ruling, and D-0 additionally gates blocker B-1.

3. **Two architectural decisions are open:** D-9 (transaction ownership — strongly indicated as "Reservation owns it" by every existing artifact, but never formally recorded) and D-15 (operation-key derivation — completely unspecified).

4. **Blockers B-1 and B-2 are structural, not procedural.** Even with all decisions resolved, the current Phase 1 consumption adapter will reject assertions on any date that has reservations, and no code path assigns `ASSERTION_MANAGED` population. Both are *Availability-side* work that must be re-validated against the Phase 1/2 lock.

**Recommended immediate sequence:**

* **Stage A (now):** review this report; resolve **C-0**; triage **P-1…P-8** as prerequisite/parallel/ignore; confirm the §2.3 command-collision runtime behaviour; confirm the §11 Case C/E behaviour with a targeted test.
* **Stage B:** resolve D-0, D-1, D-3, D-6, D-9, D-15, D-21 (blocking set); close out the non-blocking decisions.
* **Stage C:** lock the Phase 3 domain specification per §19, reconciling the existing draft.
* **Stage D:** produce the detailed implementation plan per §20.
* **Stage E:** readiness review against §21 exit gates.
* **Stage F:** implement.

**No code, schema, migration, API, DTO, or frontend change was made in this step.** Phase 1 and Phase 2 remain closed and were not reopened; the two places where Phase 1 would need to change for Phase 3 to function (population-aware consumption, status-set alignment) are raised as blockers **B-1/B-3** for an explicit ratification decision rather than assumed.

---

*End of report.*

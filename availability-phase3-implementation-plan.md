# XYLO Availability Phase 3 — Implementation Plan

## 1. Document Status and Authority

| Field | Value |
|---|---|
| Document | XYLO Availability Phase 3 Implementation Plan |
| Stage | D — Implementation Plan (planning only) |
| Version | 0.1 |
| Status | **DRAFT — AWAITING STAGE D READINESS REVIEW** |
| Authoring basis | Locked Phase 3 Domain Specification + repository audit |
| Execution authority | None. No task in this document may be implemented until the Readiness Review approves it. |

**Source hierarchy used (this order, no deviations):**

1. `availability-phase3-domain-specification.md` — **LOCKED (2026-09-28)**
2. `availability-phase3-stage-b-ratification.md`
3. `availability-phase3-business-rules-decision-sheet.md`
4. `availability-phase1-2-contract-extract.md` — **DERIVED — NOT AN AUTHORED SPECIFICATION**
5. `availability-phase3-stage-c-lock-review.md`
6. `availability-phase3-reservation-lifecycle-forensic-audit.md`
7. Existing code / tests / migrations — **implementation evidence only** (specification §4 item 7)

Where implementation evidence conflicts with the locked specification, the conflict is recorded in **§4.4 Conflict Register** and is *not* silently adapted.

**Classification key used throughout:**

| Tag | Meaning |
|---|---|
| **MUST IMPLEMENT** | Required to satisfy a locked clause; Phase 3 work. |
| **MUST NOT CHANGE** | Locked rule; the plan is forbidden from altering it. |
| **EXISTING AND COMPLIANT** | Already satisfies the locked clause; not a task. |
| **DEFER TO PHASE 6** | Explicitly out of Phase 3 (cutover/retirement/reconciliation). |

---

## 2. Scope

Phase 3 = **Reservation Lifecycle Integration**: every Reservation lifecycle path that can affect room-type inventory must execute against the Phase 2 Availability Assertion Engine, synchronously, atomically, in the ratified lock order, with ratified idempotency and population classification.

In scope:

- Reservation lifecycle write paths: create, modify dates, room-type change, cancel, no-show, check-in, early departure, normal checkout, extend/overstay, reinstate, batch status update, auto-cancel sweep, delete prohibition, waitlist join/promote, room-type upgrade.
- Availability integration: port wiring, journal claim, lock order, population classification, exact-assertion linkage.
- Transaction ownership: one business transaction per mutation path (D-9).
- Idempotency: HTTP `Idempotency-Key`, command-level, upstream identity, durable scheduled identity (D-15).
- Command ownership cleanup (D-23).
- Migration **deployment** (not authoring), testing, rollout sequencing.

## 3. Non-Goals

- No production code, schema, migration, test, route or frontend change **in this stage**.
- No redesign of the Availability domain; no new inventory calculator; no new authority.
- No quantity field, no quantity inference, no Reservation schema change (D-0).
- No second status taxonomy; no new reference-table status (spec §10.9, §34.5).
- No cutover, reconciliation, or retirement of legacy counters (Phase 6).
- No GBA/Allotment integration (Phase 4), no CRS/channel authority, no financial policy.
- No reopening of any Stage B/C decision, and no edit to the locked specification.
- No authored Phase 1/2 specification (C-7 remains closed by the Contract Extract).

---

## 4. Current Implementation State

### 4.1 Verified structural facts

| # | Fact | Evidence | Label |
|---|---|---|---|
| F-1 | `RESERVATION_AVAILABILITY_PORT` is exported by `AvailabilityModule` and imported by `ReservationsModule`, but has **zero production consumers** (4 hits: definition + 3 module lines). | `reservation-availability.port.ts:5`; `availability.module.ts:12,23,26` | VERIFIED FACT |
| F-2 | `ReservationOperationJournal` is registered (`reservations.module.ts:95,114`) but its only caller outside itself is its own spec. **No production path ever claims the journal.** | `reservation-operation-journal.ts` | VERIFIED FACT |
| F-3 | **Zero** commands implement `IdempotentCommand`; `IdempotencyPipe` short-circuits on missing key for every command. | `pipes.ts:63-65,76`; grep `extends IdempotentCommand` = 0 | VERIFIED FACT |
| F-4 | HTTP `IdempotencyInterceptor` exists but is **never registered** in `app.module.ts` or `main.ts`. | `core/interceptors/idempotency.interceptor.ts:13` | VERIFIED FACT |
| F-5 | `apps/web` contains **zero** idempotency references while `Providers.tsx:21-23` sets `mutations: { retry: 1 }`. | grep `idempotenc` in `apps/web` = 0 | VERIFIED FACT |
| F-6 | Only row locks in the Availability module are `FOR UPDATE` on **balances** (`availability-assertion.service.ts:106,361,561,604`). `reservation_availability_state` is only ever plain-read; `reservations` is never locked by the engine. | grep `FOR UPDATE` | VERIFIED FACT |
| F-7 | Engine order today = `balance locks → assertion insert/update → state UPDATE (last)`; no journal claim; no `reservations` lock. **Inverts specification §23 / D-24.** | service `:201,222,597,243` etc. | VERIFIED FACT |
| F-8 | `releaseInventoryTx` (transaction-aware) has **zero callers**; `crs.releaseInventory` opens its own root `prisma.$transaction` and is called from *inside* repository transactions at `reservation.repository.ts:609,640,673`. | grep `releaseInventoryTx` = 1 (definition) | VERIFIED FACT |
| F-9 | `TransactionPipe` wraps every command in `transactionManager.execute(...)` but **passes no `tx` to the handler** (`pipes.ts:88-104`), so each command opens a second transaction. | `cqrs.module.ts:37-41` | VERIFIED FACT |
| F-10 | `reservation_availability_state` is **never written by production code**; `setPopulationInTransaction` has no production caller. With 18 pending migrations, `assertInTransaction` would throw `RESERVATION_POPULATION_MISMATCH` for every one of the 1124 live Reservations. | service `:450-470`; `prisma migrate status` | VERIFIED FACT |
| F-11 | `reservation-consumption.adapter` returns `UNRESOLVED` whenever any candidate exists (`:23`) and classifies raw persisted strings (`:5,24-27`). | adapter `:12-36` | VERIFIED FACT |
| F-12 | **No availability view reads `availability_assertion_balances`** — all 22 non-test hits are inside `availability-assertion.service.ts`. The snapshot's `consumption` comes only from the legacy reservations source. | grep `availability_assertion_balances` | VERIFIED FACT |
| F-13 | D-23 collision is live: `ExtendStayCommand` and `ReinstateReservationCommand` are each registered **twice**; `CqrsModule` is `@Global()` and `CommandBus.register` is last-wins, so **Front Office overwrites Reservations**. | `command-bus.ts:20-27`; `front-office.module.ts:210,214`; `command-bus-collision.spec.ts:88-104` | VERIFIED FACT |
| F-14 | No scheduled Reservation job exists. No-show (`reservations.controller.ts:98`) and auto-cancel (`:503`) are HTTP-only; Night Audit is a stub (`finance-night-audit.service.ts:51-68`). | grep `@Cron` → only outbox plumbing | VERIFIED FACT |
| F-15 | `AuthorizationPipe` hard-fails any command with no `userId` (`pipes.ts:56-60`) and `requireHotelId()` needs the AsyncLocalStorage request store, so a system/scheduled identity **cannot run today**; the `?? 'SYSTEM'` fallback at `execute-auto-cancel-sweep.handler.ts:38` is unreachable. | — | VERIFIED FACT |
| F-16 | Migration state: **18 pending, 4 drift**; last common = `20260819000000_add_inv_audit_log`. Both `20260927000000_availability_assertion_engine` and `20260928000000_availability_phase3_reservation_foundation` are **unapplied**. | `prisma migrate status` | OBSERVED BEHAVIOR |

### 4.2 Lifecycle path state (condensed)

| Path | Route | Command | Tx today | Inventory effect today | State |
|---|---|---|---|---|---|
| Create | `POST /reservations` | `CreateReservationCommand` | repo-only | **none** | **MUST IMPLEMENT** |
| Modify dates | `PUT /reservations/:id` | `UpdateReservationCommand` | split (2 tx) | legacy CRS, non-atomic | **MUST IMPLEMENT** |
| Change room type | `PUT /reservations/:id/room-type` | `ChangeRoomTypeCommand` | **none** | none (silent no-op) | **MUST IMPLEMENT** |
| Change rate | `PUT /reservations/:id/rate` | `ChangeRateCommand` | none | none | **DEFER** (no inventory effect; spec §3 out-of-scope) |
| Cancel | `POST /reservations/:id/cancel` | `CancelReservationCommand` | repo tx | legacy release, **separate tx** | **MUST IMPLEMENT** |
| No-show | `POST /reservations/:id/no-show` | `ProcessNoShowCommand` | repo tx | legacy release, separate tx | **MUST IMPLEMENT** |
| Check-in | `POST /check-in/finalize` | `FinalizeCheckInCommand` | SERIALIZABLE | none (correct) | **EXISTING AND COMPLIANT** |
| Early departure | `POST /front-office/check-out` | `CheckOutCommand` | partial | **none released** | **MUST IMPLEMENT** |
| Normal checkout | `POST /front-office/check-out` | `CheckOutCommand` | partial | none (correct) | **EXISTING AND COMPLIANT** |
| Extend | `POST /reservations/:id/extend` + `POST /front-office/:id/extend` | **duplicate** `ExtendStayCommand` | none / partial | check-only, never asserts | **MUST IMPLEMENT** |
| Reinstate | `POST /reservations/:id/reinstate` + `POST /front-office/reinstate` | **duplicate** `ReinstateReservationCommand` | none / partial | none | **MUST IMPLEMENT** |
| Batch status | `POST /reservations/batch/status` | `BatchUpdateStatusCommand` | **none** | **none** | **MUST IMPLEMENT** |
| Auto-cancel | `POST /reservations/auto-cancel/sweep` | `ExecuteAutoCancelSweepCommand` | **none** | **none**, bypasses `repo.cancel` | **MUST IMPLEMENT** |
| Delete | `DELETE /reservations/:id` | `DeleteReservationCommand` | repo tx | legacy release | **MUST IMPLEMENT** (prohibition enforcement) |
| Waitlist join | `POST /reservations/:id/waitlist` | `JoinWaitlistCommand` | repo tx | **no release** | **MUST IMPLEMENT** |
| Waitlist promote | `POST /reservations/:id/waitlist/promote` | `PromoteFromWaitlistCommand` | repo tx | **no assert** | **MUST IMPLEMENT** |
| Room-type upgrade | `POST /front-office/…` (`upgrade-room.handler.ts`) | `UpgradeRoomCommand` | none observed | check-only | **MUST IMPLEMENT** |
| Mass update | `POST /reservations/mass-update/execute` | `MassUpdateExecuteCommand` | none | check-only, no reserve/release | **MUST IMPLEMENT** |

### 4.3 Existing test surface

39 spec files across `reservations`, `availability`, `front-office`, `common/cqrs`. Known pre-existing failures (Stage A, reproduced with new files removed): `reservation-integration.spec.ts` (3/17), `mass-update-execute.handler.spec.ts` (2/5), `check-in/__tests__/queries.handler.spec.ts` (3/10). **These are not Phase 3 regressions** and are tracked as baseline debt (R-6).

### 4.4 Conflict Register — implementation vs locked specification

| ID | Conflict | Spec ref | Handling |
|---|---|---|---|
| **X-1** | Create consumes no inventory; a CONFIRMED Reservation commits with no assertion. | §10.1, §11 row 199, §32.8 | **MUST IMPLEMENT** (T-12) |
| **X-2** | Six handlers are silent no-ops: `repo.update` reads camelCase while callers pass snake_case — `confirm`, `guarantee`, `reinstate`, `extend`, `change-room-type`, `change-rate`. | §11, §21 | **MUST IMPLEMENT** where inventory-affecting (T-13/14/20/21); change-rate → **DEFER** |
| **X-3** | `PUT /reservations/:id` does not persist date changes (DTO snake_case vs `repo.update` camelCase). | §11, §12.2 | **MUST IMPLEMENT** (T-13) |
| **X-4** | Inventory release runs in a *different* transaction from the status write (`crs.releaseInventory` opens a root tx; `releaseInventoryTx` unused). | §21 Success/Failure | **MUST IMPLEMENT** (T-15/16/24) |
| **X-5** | `batchUpdateStatus` has no transaction and performs no release. | §32.23, §21 | **MUST IMPLEMENT** (T-22) |
| **X-6** | Auto-cancel bypasses `repo.cancel`: no release, no status history, no status guard, no transaction. | §11 auto-cancel row, D-9 | **MUST IMPLEMENT** (T-23) |
| **X-7** | Early departure rewrites `departure_date` and releases nothing. | §10.4, §32.27 | **MUST IMPLEMENT** (T-18) |
| **X-8** | Front Office extend is `isAvailable` check-only — never asserts before committing dates. | §10.5, §32.28 | **MUST IMPLEMENT** (T-20) |
| **X-9** | `joinWaitlist` moves to WAITLIST without releasing; `promoteFromWaitlist` re-confirms without asserting. | §10.1, §11 rows 205/212 | **MUST IMPLEMENT** (T-25) |
| **X-10** | Lock order inverted/absent: no journal claim, state locked *after* balances, `reservations` never locked. | §23 (D-24) | **MUST IMPLEMENT** (T-05/06/07) |
| **X-11** | Availability port is orphaned — no lifecycle path calls it. | §5, §8 | **MUST IMPLEMENT** (T-02) |
| **X-12** | Population never assigned → `RESERVATION_POPULATION_MISMATCH` for all live rows. | §27 (D-21) | **MUST IMPLEMENT** (T-09) |
| **X-13** | Snapshot `consumption` reads *all* Reservations regardless of population, while assertion balances read only ASSERTION_MANAGED → a dual-count risk the moment T-12 lands. | §27, §32.25 | **MUST IMPLEMENT** (T-10) |
| **X-14** | Consumption adapter classifies raw persisted strings and never resolves. | §10.9 (D-0/D-1), extract §4.5/§7 | **MUST IMPLEMENT** (T-11) |
| **X-15** | No idempotency adopters; interceptor unregistered; header name mismatch (`x-idempotency-key` vs `Idempotency-Key`); frontend sends none. | §22 (D-15) | **MUST IMPLEMENT** (T-28/29/30/32) |
| **X-16** | D-23 duplicates live; Front Office handler wins; both Reservations routes dispatch an incompatible payload. | §28, D-23 | **MUST IMPLEMENT** (T-33/34) |
| **X-17** | No scheduled identity is constructible: `AuthorizationPipe` rejects commands without `userId`. | §22 scheduled/Night Audit rows | **MUST IMPLEMENT** (T-31) |
| **X-18** | `reinstate` gate tests `'NO_SHOW'` but storage is `'NO-SHOW'` → no-shows can never be reinstated. | §10.7, §11 row 220, §10.9 rule 1 | **MUST IMPLEMENT** (T-21) |
| **X-19** | `DELETE /reservations/:id` exists as a lifecycle operation on active Reservations. | §10.2 (Decision 12) | **MUST IMPLEMENT** (T-24) |
| **X-20** | `TransactionPipe`'s transaction never contains handler writes; two transactions per command. | §21, D-9 | **MUST IMPLEMENT** (T-03) |
| **X-21** | `normalizeStatus` throws on exported `IN_HOUSE`; `PROSPECT` has no reference row. | §34.5.2, §34.5.1 | **MUST IMPLEMENT** for `IN_HOUSE` (T-11b); `PROSPECT` → **DEFER** (no Phase 3 path persists it) |
| **X-22** | `changeRate` is a silent no-op, but rate does not affect room-type inventory. | §3 out-of-scope | **DEFER** — recorded, not a Phase 3 task |

---

## 5. Implementation Architecture

```
HTTP / scheduled trigger
   │  ① Idempotency-Key acquired  (T-29/T-30/T-31)
   ▼
Command (extends IdempotentCommand, carries operationKey)  (T-32)
   │  ② TransactionPipe opens ONE tx and passes it down   (T-03)
   ▼
Reservation handler  ── owns lifecycle state change
   │  ③ journal claim  (operation-journal claim)          (T-05)
   │  ④ reservation_availability_state lock  FOR UPDATE   (T-06)
   │  ⑤ sorted assertion-balance locks                    (existing, keep)
   │  ⑥ reservations row lock                             (T-07)
   ▼
ReservationAvailabilityPort (assert | release | replace | findCurrent | setPopulation)
   │  population assigned on first touch → ASSERTION_MANAGED   (T-09)
   ▼
AvailabilityAssertionService  (Phase 2 engine — MUST NOT CHANGE its balance/movement rules)
   │  capacity adjudication, exact-assertion linkage, append-only movements
   ▼
ONE COMMIT — Reservation state + assertion state + journal completion
```

**Principles carried from the locked specification:**

- Availability is authoritative; no caller computes a balance (§5).
- Every Availability-affecting path is in the same business transaction (§21 Scope and synchronicity, D-9).
- Classification runs on canonical status (§10.9).
- Quantity is constant `1` (§7) — never inferred, never a new field.
- Released assertions are never reused (§32.13).

---

## 6. Exact Change Inventory

| ID | Change | Class | Primary file(s) |
|---|---|---|---|
| C-01 | Deploy the 18 pending migrations (authoring: **none**) | MUST IMPLEMENT | `packages/db/migrations/*` (apply only) |
| C-02 | Single unit-of-work: `TransactionPipe` supplies `tx` to handlers | MUST IMPLEMENT | `common/cqrs/pipes.ts`, `cqrs.module.ts` |
| C-03 | Journal claim as step ① of every Availability mutation | MUST IMPLEMENT | `reservation-operation-journal.ts`, port impl |
| C-04 | Lock `reservation_availability_state FOR UPDATE` before balance locks | MUST IMPLEMENT | `availability-assertion.service.ts` |
| C-05 | Lock `reservations` row last | MUST IMPLEMENT | `availability-assertion.service.ts` / handler |
| C-06 | First-touch population assignment (D-21) | MUST IMPLEMENT | `availability-assertion.service.ts:450-470`, port |
| C-07 | Partition legacy consumption source by population (§27) | MUST IMPLEMENT | `reservation-consumption.adapter.ts` |
| C-08 | Canonical classification + `quantity = candidates.length` in adapter (D-0/D-1) | MUST IMPLEMENT | `reservation-consumption.adapter.ts` |
| C-09 | `normalizeStatus` accepts exported `IN_HOUSE` | MUST IMPLEMENT | `packages/shared/src/reservation-state-machine.ts` |
| C-10 | Wire `RESERVATION_AVAILABILITY_PORT` into every inventory-affecting handler | MUST IMPLEMENT | `reservations.module.ts`, handlers |
| C-11 | Align `repo.update` payload key contract with callers | MUST IMPLEMENT | `reservation.repository.ts`, DTOs |
| C-12 | Integrate port into create / modify / room-type / cancel / no-show / extend / reinstate / batch / auto-cancel / delete / waitlist / upgrade / mass-update | MUST IMPLEMENT | see §21 |
| C-13 | Early-departure release integration | MUST IMPLEMENT | `check-out.handler.ts` |
| C-14 | Enforce active-Reservation delete prohibition | MUST IMPLEMENT | `delete-reservation.handler.ts`, `reservation.repository.ts` |
| C-15 | Fix reinstate `NO_SHOW`/`NO-SHOW` gate | MUST IMPLEMENT | `reinstate-reservation.handler.ts` |
| C-16 | Delete Front Office duplicate command classes; FO dispatches Reservations commands | MUST IMPLEMENT | `front-office/application/commands/{extend-stay,reinstate}*` |
| C-17 | Fix `IdempotencyPipe` to mark after execution with success/failure states | MUST IMPLEMENT | `common/cqrs/pipes.ts` |
| C-18 | Register HTTP `IdempotencyInterceptor`; normalise header name | MUST IMPLEMENT | `app.module.ts`, `core/interceptors/idempotency.interceptor.ts` |
| C-19 | Frontend generates and reuses `Idempotency-Key` on mutations | MUST IMPLEMENT | `apps/web/lib/api/client.ts`, `Providers.tsx` |
| C-20 | System/scheduled operation identity + `AuthorizationPipe` service principal | MUST IMPLEMENT | `common/cqrs/pipes.ts`, `request-context.ts` |
| C-21 | Adopt `IdempotentCommand` on inventory-affecting commands | MUST IMPLEMENT | see §21 |
| C-22 | Regression + concurrency + isolation + fail-closed test suites | MUST IMPLEMENT | see §16 |
| — | Quantity model, status taxonomy, balance/movement rules, exact-assertion linkage, snapshot arithmetic | **MUST NOT CHANGE** | §7 reference |
| — | Legacy `availability`/`inventory` counters, GBA/allotment, snapshot read path for LEGACY population, cutover, reconciliation | **DEFER TO PHASE 6 / 4** | §13 |
| — | Check-in double-consumption, balance sort order, tenancy guards, fail-closed unresolved handling, movement append-only trigger | **EXISTING AND COMPLIANT** | §7 reference |

---

## 7. Reservation Lifecycle Integration Map

Per path: locked clause → current code → required integration.

| # | Path | Locked clause | Current code | Required integration | Class |
|---|---|---|---|---|---|
| 1 | Create | §10.1, §11 r199/200, §32.8 | `repo.create` — no port call | Assert full `[arrival, departure)` before the consuming state commits; for PENDING hold assert hold set; for no-availability PENDING assert nothing | MUST IMPLEMENT |
| 2 | Modify dates | §11, §12.2, §21 Replacement | two transactions, key mismatch | Single tx; `replaceInTransaction` for the date delta; old commitment retained until replacement succeeds | MUST IMPLEMENT |
| 3 | Room-type change | §10.8, §11 r218, §19 | silent no-op | `replaceInTransaction` across room-type balance keys | MUST IMPLEMENT |
| 4 | Cancel | §10.2, §16, §12.2 | release in separate tx | `releaseInTransaction` on exact assertion, same tx as `CANCELLED` | MUST IMPLEMENT |
| 5 | No-show | §10.6, §17, §12.2 | releases full range via legacy path | Release **remaining unelapsed nights only**; never the elapsed arrival night | MUST IMPLEMENT |
| 6 | Check-in | §10.3, §32.11 | SERIALIZABLE commit, no inventory | **No change.** Verify only | EXISTING AND COMPLIANT |
| 7 | Early departure | §10.4, §13, §32.27 | rewrites date, releases nothing | `replaceInTransaction` removing `[E, scheduled departure)` atomically with shortened dates | MUST IMPLEMENT |
| 8 | Normal checkout | §10.3, §14, §32.26 | no release (correct) | **No release.** Assert absence of any release call | EXISTING AND COMPLIANT |
| 9 | Extend/overstay | §10.5, §15, §32.28/29 | check-only / no-op | Assert added nights **before** committing extended dates; failure leaves dates unchanged | MUST IMPLEMENT |
| 10 | Reinstate | §10.7, §18 | no-op + gate bug | Fresh evaluation + **new assertion identity** before restoring consuming state; fix `NO-SHOW` gate | MUST IMPLEMENT |
| 11 | Batch status | §32.23, §20 | no tx, no release | Per-Reservation interactive tx; explicit per-item results; each item fully committed or not | MUST IMPLEMENT |
| 12 | Auto-cancel | §11 auto-cancel row, §10.2, D-9 | bypasses cancel entirely | Route through the cancellation contract per item: exact-assertion release + `CANCELLED` in one tx | MUST IMPLEMENT |
| 13 | Delete | §10.2 (Decision 12) | deletes with legacy release | Refuse hard delete of an active Reservation; non-terminal states must be cancelled first | MUST IMPLEMENT |
| 14 | Waitlist join | §11 r212, §10.1 | moves state, no release | Release the hold assertion when leaving a consuming context | MUST IMPLEMENT |
| 15 | Waitlist promote | §11 r205 | re-confirms, no assert | Assert before the consuming state commits | MUST IMPLEMENT |
| 16 | Room-type upgrade | §10.8, §11 r218, §19 | check-only | Atomic old-to-new replacement; old commitment retained on failure | MUST IMPLEMENT |
| 17 | Mass update | §12.2, §32.23 | check-only, no reserve, 1000-row truncation | Treat as batch: per-item transaction + inventory effect + explicit partial results | MUST IMPLEMENT |
| 18 | Rejection path | — | no rejection concept exists in the state machine | None required | NOT APPLICABLE |

---

## 8. Availability Integration Map

| Component | File | Role in Phase 3 | Change |
|---|---|---|---|
| Port interface | `reservations/application/ports/reservation-availability.port.ts:48-75` | `assert / release / replace / findCurrent / setPopulation`, each `tx`-first | **MUST NOT CHANGE** signature; consumers must be added |
| Port binding | `availability/availability.module.ts:23` | `useExisting: AvailabilityAssertionService` | **EXISTING AND COMPLIANT** — wire consumers |
| Assertion service | `availability/application/services/availability-assertion.service.ts` | engine implementation | Balance/movement/linkage rules **MUST NOT CHANGE**; lock order + population **MUST IMPLEMENT** |
| Operation journal | `reservations/infrastructure/repositories/reservation-operation-journal.ts` | `claim/complete/reject/find`, `skipDuplicates` | **MUST IMPLEMENT** — must be invoked from every mutation path |
| Consumption adapter | `availability/infrastructure/adapters/reservation-consumption.adapter.ts` | feeds snapshot `reservations` source | **MUST IMPLEMENT** C-07 + C-08 |
| Source adapter | `availability/infrastructure/adapters/availability-source.adapter.ts` | aggregates sources incl. reservations | **MUST IMPLEMENT** (population partition read) |
| Snapshot service | `availability/application/services/availability-snapshot.service.ts` | read-only fail-closed view | Arithmetic **MUST NOT CHANGE** (extract §4.4) |
| Snapshot calculator | `availability/domain/policies/snapshot-calculator.ts` | pure policy | **MUST NOT CHANGE** |
| Transaction manager | `common/database/unit-of-work/transaction-manager.ts` | isolation + deadlock retry | **EXISTING AND COMPLIANT** |
| Unit of work | `common/database/unit-of-work/unit-of-work.ts` | ambient tx holder, no consumers | **EXISTING AND COMPLIANT** (dormant) — revisit only via C-02 |
| Legacy CRS engine | `rates-inventory/crs-engine.service.ts` | legacy counters | **DEFER TO PHASE 6** except where a path must stop calling it (§13) |
| Schema | `packages/db/schema.prisma:17310-17429` | 5 Phase 2/3 tables | **MUST NOT CHANGE** |

**Assertion linkage (§9.1) — MUST NOT CHANGE:** `hotelId` + `referenceType = RESERVATION` + `referenceId = Reservation.id`; `assertionId` exact-verified before replace/release. Enforced today by `findCurrentInTransaction` (`:419-448`) and the partial unique index — **EXISTING AND COMPLIANT**.

---

## 9. Transaction Strategy

**Rule (spec §21 Scope and synchronicity, D-9):** the Availability mutation and the Reservation mutation are one business operation — committed together or not at all. No outbox, saga or event bus may substitute.

| Item | Plan |
|---|---|
| Unit of work | One interactive transaction per mutation path (C-02). `TransactionPipe` must pass `tx` to the handler; handlers must stop opening a second `$transaction`. |
| Isolation | Keep `READ_COMMITTED` as the engine default (`transaction-manager.ts:30`); deadlock retry already present (exponential backoff, 3 attempts). SERIALIZABLE remains for check-in only. |
| Ordering inside the tx | journal claim → state lock → sorted balance locks → `reservations` row (§10). |
| Commit boundary | Reservation state + assertion state + movements + journal `complete()` all inside the same tx. |
| Failure | Any throw rolls back everything; the journal row is unreachable in `IN_PROGRESS` (extract §5.3 invariant 3). |
| Events | Domain events must be published **after** commit. `EventBus.dispatchInternal` is fire-and-forget in-process — **MUST NOT** be used to carry the Availability mutation. Integration events may continue to use the outbox (unrelated to inventory). |
| Paths to convert | create, modify, room-type, cancel, no-show, extend, reinstate, batch, auto-cancel, delete, waitlist join/promote, upgrade, mass-update, early departure (check-out). |

**Explicitly out:** normal checkout (no inventory mutation) and check-in (already SERIALIZABLE, compliant).

---

## 10. Locking Strategy

**Required order (spec §23, D-24) — MUST IMPLEMENT:**

```
operation-journal claim → reservation_availability_state lock → sorted assertion-balance locks → reservations row
```

| Step | Current state | Required change | Task |
|---|---|---|---|
| ① journal claim | **absent** in every path (F-2) | `journal.claim(tx, …)` as the first statement of every Availability mutation; `CLAIMED` proceeds, `REPLAY` returns recorded result, `IDEMPOTENCY_CONFLICT` aborts | T-05 |
| ② state lock | plain `findUnique` at `:201,256,302,423`; UPDATE at `:243,292,412` **after** balance locks | `SELECT … FOR UPDATE` on `reservation_availability_state` immediately after the claim, before any balance lock | T-06 |
| ③ sorted balance locks | **compliant** — `lockBalanceKeys` sorts `hotelId → roomType → stayDate` (`:546-547`) | keep; no change | EXISTING AND COMPLIANT |
| ④ `reservations` row | **never locked** by the engine; repository locks it *first* in its own tx | lock after balance locks, before writing lifecycle state | T-07 |

**Known violations to remove:**

| Location | Violation | Fix |
|---|---|---|
| `availability-assertion.service.ts:50-196` (`assert()`) | no claim, no state, no `reservations` lock; has **no production caller** | T-08: delete or gate behind an explicit non-lifecycle entry point that cannot be reached by Reservation paths |
| `reservation.repository.ts:433,557,605,636` | locks `reservations` **first** | reorder under C-02/C-05 |
| `crs-engine.service.ts:450-457` | locks `reservations` in its own tx before inventory work in another | legacy path — **DEFER TO PHASE 6** for untouched paths; must stop being used by Phase 3 paths (§13) |
| Test `availability-assertion-concurrency.postgres.spec.ts:120-129` | claims outside the tx | T-37: claim inside the transaction under test |

---

## 11. Idempotency Strategy

**Rule (spec §22, D-15):** operation identity is owned by the initiating business operation; Reservation ID and assertion ID never substitute for it.

| Leg | Current state | Required change | Task |
|---|---|---|---|
| HTTP `Idempotency-Key` | interceptor exists, **unregistered**; reads `x-idempotency-key` while errors say `Idempotency-Key`; `idempotency_keys` table exists | register globally for mutating verbs; accept **both** header spellings; hash-bind to method+url+body (already implemented) | T-29 |
| Retry reuse | frontend retries mutations once with **no key** | client generates a key per logical mutation and reuses it on retry | T-30 |
| Command level | `IdempotencyPipe` registered but inert (0 adopters); marks **before** execute with no failure release | mark **after** execution with success/failure states; replay returns recorded outcome | T-17 |
| Adopters | none | `extends IdempotentCommand` on: create, update, change-room-type, cancel, process-no-show, extend-stay, reinstate-reservation, batch-update-status, execute-auto-cancel-sweep, delete, join-waitlist, promote-from-waitlist, upgrade-room, check-out, mass-update-execute | T-32 |
| Upstream identity | CRS/channel must preserve upstream key separately from XYLO Reservation ID | carry `upstreamOperationId` through the command into the journal record | T-04 |
| Scheduled / Night Audit | **no durable identity exists**; `Command.commandId` is a fresh `randomUUID()` per dispatch; `AuthorizationPipe` blocks service calls | deterministic key `hotelId:businessDate:jobType:itemId`; service principal that satisfies `AuthorizationPipe` and `requireHotelId()` | T-31 |
| Journal ↔ port | port requires `operationId` + `availabilityOperationKey` (`port:16-17`) and movements FK to the journal | derive both from the same operation key so claim and completion share one tx | T-04 |

**Replay semantics (must hold):** same key + same request hash ⇒ recorded result; same key + different hash ⇒ `IDEMPOTENCY_CONFLICT` (409); unknown commit ⇒ read-back, never a fresh mutation (spec §21 Unknown outcome).

---

## 12. State Population Strategy (D-21)

| Rule | Plan |
|---|---|
| Create on first touch | `setPopulationInTransaction` becomes reachable: the port assigns population on the first Availability-managed operation |
| `ASSERTION_MANAGED` | when the operation is a Phase 3 port call |
| `LEGACY` | otherwise; **untouched Reservations remain `LEGACY`** |
| Sticky | assigned once, never re-assigned |
| No bulk backfill | explicitly forbidden; the migration's "no backfill" note stays honest |
| Failure mode | an unclassified Reservation must never be guessed (spec §27) |

**Interaction with live data (1124 rows):** every live Reservation starts with **no state row**. The first Phase 3 operation on it creates the row with `ASSERTION_MANAGED` (because that operation is a port call). Until then the row is treated as `LEGACY` and must not be asserted through the port — `assertInTransaction` must not throw `RESERVATION_POPULATION_MISMATCH` for "row absent", it must **assign** first. This is the single most important behavioural change in T-09.

**Dual-count guard (X-13):** because the snapshot counts *all* Reservations while assertion balances count only ASSERTION_MANAGED, C-07 must partition the consumption source by population before C-12 lands. Ordering is mandatory: **C-07 before C-12**.

---

## 13. Legacy Coexistence Strategy

### 13.1 Mapping `LEGACY → ASSERTION_MANAGED`

A Reservation moves population **only** at its first Phase 3 port call, and only for that operation's transaction. There is no batch migration step.

### 13.2 Old counters — what remains, what stops

| Old counter / path | Where | Phase 3 treatment | Class |
|---|---|---|---|
| `availability` table rows (legacy per-date counters) | `crs-engine.service.ts:344-390,523-537,581` | Used for **LEGACY-population** Reservations only. Phase 3 paths for ASSERTION_MANAGED Reservations **must stop calling it**. | MUST IMPLEMENT (stop-use) / DEFER (retirement) |
| `inventory` raw SELECT in mass update | `mass-update-execute.handler.ts:191-199` | Replaced by port assertion for consuming items | MUST IMPLEMENT |
| `inventoryDomain.isAvailable` | `front-office/…/extend-stay.handler.ts:58`, `upgrade-room.handler.ts:61` | Check-only; must be superseded by port assert (check-only is never sufficient for §10.5/§10.8) | MUST IMPLEMENT |
| `crs.releaseInventory` from repository | `reservation.repository.ts:609,640,673` | Replaced by `releaseInTransaction` on those paths | MUST IMPLEMENT |
| `crs.modifyReservation` | `reservation.repository.ts:546` | Replaced by `replaceInTransaction` on those paths | MUST IMPLEMENT |
| Snapshot `reservations` source | `reservation-consumption.adapter.ts` | **Remains**, but partitioned by population and reclassified canonically | MUST IMPLEMENT |
| Legacy `availability` table + reconciliation view | `availability-reconciliation.service.ts` | **Not retired**; continues for LEGACY population | DEFER TO PHASE 6 |
| GBA / allotment sources | `availability-source.adapter.ts:69-102` | Untouched | DEFER TO PHASE 4 |

### 13.3 What Phase 3 does **NOT** retire

The legacy `availability` and `inventory` tables, the reconciliation read model, the snapshot endpoint, and every LEGACY-population path remain in service. **No table is dropped, no column is removed, no legacy counter is deleted in Phase 3.**

### 13.4 Reserved for Phase 6

Cutover of a Reservation population to assertion authority, deterministic population migration at scale, legacy counter retirement, dual-source reconciliation closure, and removal of `crs-engine` inventory calls.

---

## 14. Migration / Data Strategy

| Question | Answer |
|---|---|
| Which schema/migrations already exist? | `20260927000000_availability_assertion_engine` (balances, assertions, movements, append-only trigger) and `20260928000000_availability_phase3_reservation_foundation` (`reservation_availability_operations`, `reservation_availability_state` with `population CHECK ('LEGACY','ASSERTION_MANAGED')`, movement `operation_id` + partial unique index, deferred link trigger). Also `idempotency_keys` (`schema.prisma:13173`). |
| Which are required but not deployed? | **Both** Phase 2 and Phase 3 migrations, plus 16 others — **18 pending** (F-16). |
| Is any migration missing? | **No.** Phase 3 requires no new schema. One **conditional data** item: a `reservation_status` row for `PROSPECT` — but **no Phase 3 path persists `PROSPECT`**, so it is not required now (spec §34.5.1: compatibility must be established before any such path ships). |
| Is any backfill required? | **No.** D-21 forbids bulk backfill. |
| First-touch effect on live data | 1124 Reservations have no `reservation_availability_state` row; T-09 makes first touch **assign** rather than throw. |
| Drift | **4 migrations exist in the database but not locally** (`remove_uuid_from_user_refs`, `add_warehouse_id_to_supply_request`, `add_goods_issue_id_to_receipt`, `make_po_line_id_optional_on_receipt_line`). These must be reconciled **before** applying the 18, or the apply will fail. |
| Deferred to Phase 6 | population cutover, legacy counter retirement, dual-source reconciliation. |

**MUST NOT CHANGE:** no schema edit, no new migration authoring, no `quantity` column, no status-table write, no movement/backfill script.

**Deployment order (see §17):** resolve drift → apply 18 → verify assertion tables exist in `public` → then any code that touches them.

---

## 15. Command Ownership Cleanup (D-23)

**Ratified:** Reservations owns `ExtendStayCommand` and `ReinstateReservationCommand`; Front Office *initiates* but must not maintain competing command payloads or handlers.

| Current | Problem | Required change |
|---|---|---|
| `reservations/application/commands/extend-stay/extend-stay.command.ts:3` + handler | registered at `reservations.module.ts:263` | **KEEP** (owner) |
| `front-office/application/commands/extend-stay/extend-stay.command.ts:3` + handler | registered at `front-office.module.ts:210`, **wins** last-wins | **DELETE** class + provider + registration |
| `reservations/application/commands/reinstate-reservation/reinstate-reservation.command.ts:3` + handler | registered at `reservations.module.ts:262` | **KEEP** (owner) |
| `front-office/application/commands/reinstate/reinstate.command.ts:3` + handler | registered at `front-office.module.ts:214`, **wins** | **DELETE** class + provider + registration |
| Payload divergence | Reservations: `{id, hotelId, newDepartureDate}` / `{id, hotelId}`. FO: `{reservationId, extraNights, reason}` / `{reservationId, newRoomNumber, reason}` | One payload per operation: Reservations'. FO service maps its DTO onto the Reservations payload. |
| FO routes | `POST /front-office/:id/extend`, `POST /front-office/reinstate` have **no `@Permission` decorator** | Add the same permissions the Reservations routes use (`RESERVATION_EXTEND`, `RESERVATION_REINSTATE`) |

**Post-change invariant:** exactly one `commandBus.register` per command name anywhere in the graph; the D-23 collision spec (`command-bus-collision.spec.ts`) must be inverted to assert a single owner and a warning-free registry.

Also verify for other collisions: audit the registry for any other duplicated `commandName` across modules (the collision mechanism is general, not specific to these two).

---

## 16. Testing Strategy

Runner: `cd apps/api && pnpm test` with `--maxWorkers=1 --max-old-space-size=8192`; Postgres-gated suites require `AVAILABILITY_TEST_DATABASE_URL`.

### 16.1 Functional matrix

| # | Case | Assertion points |
|---|---|---|
| F-01 | Create consuming (CONFIRMED/GUARANTEED) | assertion exists for every date in `[arrival, departure)`; state = ASSERTION_MANAGED; journal CLAIMED→COMPLETE |
| F-02 | Create non-consuming (no-availability PENDING) | **no** assertion row; no balance change |
| F-03 | Create active hold | hold assertion exists; expiry releases it |
| F-04 | Modify dates | old assertion superseded, new ACTIVE; no gap where old released and new unssecured; unchanged dates preserved |
| F-05 | Room-type change | cross-room-type replacement; old room type released only after new secured |
| F-06 | Cancel | exact-assertion release + `CANCELLED` commit together; non-consuming cancel has no release |
| F-07 | No-show | releases remaining unelapsed nights **only**; elapsed arrival night still committed |
| F-08 | Early departure | `[E, scheduled departure)` released immediately; `[arrival, E)` retained; dates shortened atomically |
| F-09 | Normal checkout | **zero** release operations issued |
| F-10 | Overstay success | added night asserted **before** `departure_date` commit |
| F-11 | Overstay failure | original dates + original assertion unchanged; no partial commit |
| F-12 | Reinstate success | fresh assertion identity; released assertion not reused |
| F-13 | Reinstate unavailable | source state unchanged; no false commitment |
| F-14 | Check-in | no second consumption; assertion identity unchanged |
| F-15 | Same-type room transfer | **zero** Availability delta |
| F-16 | Room-type upgrade | atomic replacement; old retained on failure |
| F-17 | Auto-cancel | per-item tx; release + CANCELLED; failure of one item does not affect another; re-run is idempotent |
| F-18 | Batch status | per-item atomicity; explicit partial results; no item half-mutated |
| F-19 | Waitlist join/promote | join releases hold; promote asserts **before** CONFIRMED |
| F-20 | Delete on active Reservation | refused (§10.2) |
| F-21 | Mass update | per-item inventory effect + explicit partial results |

### 16.2 Idempotency matrix

| Case | Expectation |
|---|---|
| Same `Idempotency-Key` repeated | recorded result returned; no second movement |
| Retry after success | replay, no mutation |
| Retry after failure | original durable failure returned (not re-executed) |
| Same key, different payload | 409 `IDEMPOTENCY_CONFLICT` |
| Duplicate scheduled operation | deterministic key ⇒ one execution per `hotelId:businessDate:jobType:itemId` |
| Duplicate command dispatch | `IdempotencyPipe` replay; journal `REPLAY` |

### 16.3 Concurrency matrix (Postgres-gated)

| Case | Expectation |
|---|---|
| **Case C regression (mandatory)** | **concurrent Cancel + Modify must not allow `RELEASED` to be overwritten while the Reservation still holds inventory** |
| Concurrent cancel + extend | one wins; loser re-evaluates or fails; no double release |
| Concurrent extend + modify | at most one change from the same prior state succeeds |
| Concurrent reinstatement | Availability arbitrates; loser keeps source state |
| Competing assertion operations | single active assertion invariant holds |
| **Lock-order correctness** | probe asserts claim → state → balances → reservations observed in that order |
| Uniqueness conflict / loser | `one_active_reservation_uq` violated ⇒ clean conflict, no orphaned assertion |

Existing regression target: `availability-assertion-concurrency.postgres.spec.ts:253-303` currently documents the failing behaviour — it must be flipped to assert the locked behaviour.

### 16.4 Tenant / property isolation

Wrong hotel; wrong property; cross-hotel assertion lookup; cross-property capacity; no fallback hotel id (spec §26).

### 16.5 Failure / fail-closed

Unresolved room type; insufficient capacity; assertion failure ⇒ transaction rollback; idempotency journal failure; stale/superseded assertion; **missing `reservation_availability_state`** (must assign, not throw); unknown commit outcome ⇒ read-back.

### 16.6 Baseline discipline

The 8 pre-existing failing tests (F-3 suites) must be recorded as a **baseline** before Phase 3 work starts, so new failures are attributable.

---

## 17. Rollout Sequence

| Phase | Content | Gate |
|---|---|---|
| **R0 — Baseline** | Record the 8 pre-existing test failures; run full suite; snapshot `prisma migrate status` | Baseline captured |
| **R1 — Schema deploy** | Reconcile 4 drift migrations → apply 18 pending → verify the 5 assertion tables exist in `public` | DB ready; assertion tables present |
| **R2 — Foundation** | C-02 (single tx), C-03 (journal claim), C-04/C-05 (lock order), C-06 (population first-touch), C-09 (`IN_HOUSE`) | Lock-order + population tests green |
| **R3 — Read path** | C-07 (population partition) **then** C-08 (canonical classification + quantity) | Snapshot resolves on live-shaped data |
| **R4 — Wiring** | C-10 (port consumers), C-11 (payload key contract) | Port reachable from handlers |
| **R5 — Lifecycle** | C-12/C-13/C-14/C-15 in dependency order: create → cancel/no-show → modify/room-type → extend → reinstate → waitlist/upgrade → batch/auto-cancel/mass-update/delete | Functional matrix F-01…F-21 green |
| **R6 — Idempotency** | C-17, C-18, C-19, C-20, C-21 | Idempotency matrix green |
| **R7 — Command ownership** | C-16 | Single-owner registry; collision spec inverted |
| **R8 — Hardening** | C-22 full test matrix incl. Case C regression, isolation, fail-closed | All gates §18 |

**Ordering constraints (violating these breaks the specification):**

1. **R1 before R2** — no code may touch tables that do not exist.
2. **C-07 before C-12** — partition the consumption source before any Reservation asserts, or §32.25 dual-count occurs.
3. **C-02 before C-05** — a single tx must exist before ordering locks inside it.
4. **C-06 before C-12** — population must assign on first touch or every assert fails.
5. **C-16 after the Reservations handlers are correct** — deleting the FO handlers before the owner handler works would strand the FO routes.

---

## 18. Validation Gates

| Gate | Criterion |
|---|---|
| **G1 Schema** | `prisma migrate status` clean; 5 tables present; drift resolved |
| **G2 Order** | Lock-order test proves claim → state → balances → reservations |
| **G3 Population** | First touch assigns; no `RESERVATION_POPULATION_MISMATCH` on live-shaped data; no backfill script exists |
| **G4 Atomicity** | No path commits Reservation state without its assertion (or its deliberate non-consumption) |
| **G5 No dual count** | Each Reservation counted by exactly one of {legacy source, assertion balances} |
| **G6 Idempotency** | Every inventory-affecting command is `IdempotentCommand`; HTTP key honoured end-to-end incl. frontend |
| **G7 Ownership** | Exactly one handler per command name; FO routes resolve to Reservations handlers |
| **G8 Test matrix** | §16 fully green; Case C regression green; baseline failures unchanged in count |
| **G9 Non-regression** | Check-in, normal checkout, quantity model, status taxonomy, balance rules byte-unchanged |
| **G10 Scope** | Zero schema authoring; zero legacy table retirement; zero Phase 1/2 spec authoring |

---

## 19. Risks and Mitigations

| ID | Risk | L | I | Mitigation |
|---|---|---|---|---|
| R-1 | Applying 18 migrations against a drifted DB corrupts state | H | H | Reconcile the 4 drift migrations first, in a maintenance window, with a DB snapshot |
| R-2 | Ordering error (C-12 before C-07) causes dual counting | M | H | Enforced by the §17 ordering constraint; gate G5 test written in R3 |
| R-3 | Lock-order change introduces deadlocks under load | M | M | Balanced locks already sorted; keep READ_COMMITTED; existing `TransactionManager` deadlock retry; concurrency suite in R8 |
| R-4 | First-touch population misclassifies a Reservation | M | H | Sticky assignment + fail-closed unknown population (§27); no reclassification path |
| R-5 | Deleting FO handlers strands FO routes mid-sequence | M | M | Sequence C-16 last within R7; FO service maps onto Reservations payload before deletion |
| R-6 | 8 pre-existing failures mask new regressions | H | M | Capture baseline in R0; compare counts, not absolutes |
| R-7 | Frontend key adoption lags backend | M | M | Backend accepts requests without a key (passthrough) so R6 is not blocked; frontend tracked as its own task C-19 |
| R-8 | Scheduled identity work reopens authorization design | M | M | Scope C-20 strictly to a service principal satisfying `AuthorizationPipe` + `requireHotelId()`; no new auth model |
| R-9 | Silent no-op handlers hide incomplete integration (X-2) | H | M | C-11 makes the payload contract explicit and adds a "modified" assertion per handler test |

---

## 20. Deferred Items

| Item | Destination |
|---|---|
| Legacy `availability` / `inventory` counter retirement | Phase 6 |
| Population cutover at scale; dual-source reconciliation closure | Phase 6 |
| Removing `crs-engine` inventory calls from untouched legacy paths | Phase 6 |
| GBA / Allotment integration | Phase 4 |
| `PROSPECT` reference-row compatibility (no Phase 3 path persists it) | Deferred until a path exists |
| `changeRate` silent no-op (no inventory effect) | Deferred — outside Availability scope (spec §3) |
| Night Audit scheduled orchestration beyond durable identity | Operational SOP review (spec §34.2) |
| Direct `CONFIRMED → CHECKED_OUT` lifecycle discrepancy | Reservations/FO owners (spec §34.3) |
| Hotel-local cutoff / effective-time policy | Owning business policy (spec §34.4) |
| Reservations spec lock metadata (C-6) | Reservations rebuild track |
| Cross-property transfer (ADR-072) | Separate authorization |
| Frontend amendment-impact dialogs / Quick Book | Phase 9 frontend roadmap |

---

## 21. File-by-File Implementation Task List

Format per task: **File · State · Change · Why · Spec ref · Deps · Tx boundary · Locking · Idempotency · Tests · Migration · Risk.**

### Wave R2 — Foundation

---

**T-01 — Deploy pending migrations**
- **File:** `packages/db/migrations/` (apply only — `pnpm db:push`/`migrate deploy`); reconcile `20260714183730_remove_uuid_from_user_refs`, `20260716120000_add_warehouse_id_to_supply_request`, `20260716140000_add_goods_issue_id_to_receipt`, `20260716150000_make_po_line_id_optional_on_receipt_line`
- **State:** 18 pending, 4 drift; Phase 2/3 tables absent from `public`
- **Change:** resolve drift, then apply all pending
- **Why:** no Phase 3 code can run without the tables (F-16)
- **Spec ref:** §3, §27 (infrastructure prerequisite)
- **Deps:** none
- **Tx:** migration tooling
- **Locking:** n/a
- **Idempotency:** n/a
- **Tests:** `prisma migrate status` clean; G1
- **Migration:** **apply, do not author**
- **Risk:** **HIGH** (drift) — snapshot first

---

**T-02 — Single unit-of-work (`TransactionPipe` supplies `tx`)**
- **File:** `apps/api/src/common/cqrs/pipes.ts:88-104`, `apps/api/src/common/cqrs/cqrs.module.ts:37-41`
- **State:** pipe opens a tx the handler cannot see; handlers open a second tx (F-9)
- **Change:** pass the ambient `tx` into handler execution; handlers accept it and stop opening nested `$transaction` where they do so today
- **Why:** one business operation must span Reservation + Availability (§21, D-9)
- **Spec ref:** §21 Scope and synchronicity; D-9
- **Deps:** R1
- **Tx:** single interactive tx per command
- **Locking:** prerequisite for ordering locks inside one tx
- **Idempotency:** unchanged
- **Tests:** F-01/F-06 prove one commit; no double-open
- **Migration:** none
- **Risk:** HIGH — touches every command

---

**T-03 — Journal claim as step ①**
- **File:** `apps/api/src/modules/reservations/infrastructure/repositories/reservation-operation-journal.ts:30-107`; invocation site inside the port impl
- **State:** registered, never called in production (F-2)
- **Change:** `claim(tx, …)` first statement of every Availability mutation; `REPLAY` returns recorded result; `IDEMPOTENCY_CONFLICT` aborts; `complete()`/`reject()` in the same tx
- **Why:** D-24 step 1; extract §5.3 invariant 3
- **Spec ref:** §23, §22, extract §5.3/§5.4
- **Deps:** T-02
- **Tx:** claim + completion share the tx
- **Locking:** claim acquires the journal row first
- **Idempotency:** primary durable claim
- **Tests:** idempotency matrix; F-06 replay
- **Migration:** table exists (Phase 3 migration) — no authoring
- **Risk:** MEDIUM

---

**T-04 — Operation identity derivation (D-15)**
- **File:** new helper under `reservations/application/` or `common/cqrs/`; consumed by handlers
- **State:** no derivation rule exists; port demands `operationId` + `availabilityOperationKey` (extract §5.4)
- **Change:** derive per initiator — HTTP → header key; channel → upstream identity preserved separately; scheduled → `hotelId:businessDate:jobType:itemId`
- **Why:** §22 requires it; port requires it
- **Spec ref:** §22, D-15; extract §5.4
- **Deps:** T-03, T-31 (scheduled leg)
- **Tx:** key created before claim
- **Locking:** none
- **Idempotency:** the identity itself
- **Tests:** F-05 idempotency rows; upstream identity preserved
- **Migration:** none
- **Risk:** MEDIUM

---

**T-05 — Lock `reservation_availability_state` (step ②)**
- **File:** `apps/api/src/modules/availability/application/services/availability-assertion.service.ts:199-249, 251-297, 299-417, 419-448`
- **State:** plain `findUnique` at `:201,256,302,423`; state `UPDATE` occurs **after** balance locks (F-7)
- **Change:** `SELECT … FOR UPDATE` on the state row immediately after the journal claim, before any balance lock; retain read-consistency checks
- **Why:** D-24 step 2; closes the documented read-before-lock race
- **Spec ref:** §23 (D-24)
- **Deps:** T-03
- **Tx:** inside the command tx
- **Locking:** **this is the core lock-order change**
- **Idempotency:** unchanged (claim already done)
- **Tests:** T-37 lock-order probe; Case C regression
- **Migration:** none
- **Risk:** HIGH

---

**T-06 — Lock `reservations` row last (step ④)**
- **File:** port impl + `apps/api/src/modules/reservations/infrastructure/repositories/reservation.repository.ts:433,557,605,636`
- **State:** repository locks `reservations` **first**; engine never locks it (F-6)
- **Change:** acquire `reservations … FOR UPDATE` after balance locks and before writing lifecycle state; remove first-position locks from the converted paths
- **Why:** D-24 step 4; prevents stale-state commit
- **Spec ref:** §23, §23 row 1
- **Deps:** T-02, T-05
- **Tx:** same tx
- **Locking:** completes the required order
- **Idempotency:** none
- **Tests:** lock-order probe; concurrency matrix
- **Migration:** none
- **Risk:** HIGH

---

**T-07 — First-touch population assignment (D-21)**
- **File:** `availability-assertion.service.ts:450-470` (`setPopulationInTransaction`), `:199-249` (`assertInTransaction` guard at `:204-206`)
- **State:** never called in production; guard throws `RESERVATION_POPULATION_MISMATCH` for missing rows (F-10)
- **Change:** on first Availability-managed operation, create the row; `ASSERTION_MANAGED` for port calls, `LEGACY` otherwise; absent row ⇒ **assign**, do not throw; sticky; no reclassification; no backfill
- **Why:** §27 (D-21); without it every live Reservation fails
- **Spec ref:** §27
- **Deps:** T-01
- **Tx:** assignment inside the operation tx
- **Locking:** assignment occurs after step ②
- **Idempotency:** assignment is idempotent by uniqueness of the state row
- **Tests:** G3; missing-state fail-closed case
- **Migration:** table + CHECK exist; **no backfill**
- **Risk:** HIGH

---

**T-08 — Retire/bypass standalone `assert()`**
- **File:** `availability-assertion.service.ts:50-196`
- **State:** owns its own tx, no claim, no state, no `reservations` lock; **no production caller** (extract §4.1)
- **Change:** remove it, or gate it so no Reservation lifecycle path can reach it
- **Why:** §21 forbids an off-contract mutation path
- **Spec ref:** §21, §5 (extract §4.1)
- **Deps:** none
- **Tx:** n/a after removal
- **Locking:** removes an ordering bypass
- **Idempotency:** currently has its own scheme — replaced by the journal
- **Tests:** assert no caller remains
- **Migration:** none
- **Risk:** LOW

---

**T-09 — `normalizeStatus` accepts `IN_HOUSE`**
- **File:** `packages/shared/src/reservation-state-machine.ts:59-65`
- **State:** `RESERVATION_STATUSES` exports `IN_HOUSE`; `normalizeStatus` only recognises `IN-HOUSE` → throws (F/X-21)
- **Change:** map both spellings to `CHECKED_IN`
- **Why:** §10.9 rule 1; §34.5.2 "must be closed"
- **Spec ref:** §10.9 rule 1, §34.5.2
- **Deps:** none
- **Tx:** n/a
- **Locking:** n/a
- **Idempotency:** n/a
- **Tests:** status mapping unit tests for all 9 canonical + alias forms
- **Migration:** none (no new status; **MUST NOT** add a reference row)
- **Risk:** LOW

---

### Wave R3 — Read path

---

**T-10 — Partition consumption source by population (C-07)**
- **File:** `apps/api/src/modules/availability/infrastructure/adapters/reservation-consumption.adapter.ts:12-36`, `availability-source.adapter.ts:53`
- **State:** counts **all** Reservations regardless of population; no population awareness (X-13/F-12)
- **Change:** count only Reservations whose population is `LEGACY` (or unassigned); ASSERTION_MANAGED Reservations are represented solely by assertion balances
- **Why:** §27 "must not be represented as consumed by both sources"; §32.25
- **Spec ref:** §27, §32.25
- **Deps:** T-01, T-07
- **Tx:** read path, no lock
- **Locking:** none (plain read)
- **Idempotency:** n/a
- **Tests:** G5 dual-count test; snapshot reconciliation
- **Migration:** none
- **Risk:** MEDIUM — ordering-sensitive with T-12

---

**T-11 — Canonical classification + quantity (C-08)**
- **File:** `reservation-consumption.adapter.ts:5,14-30`
- **State:** raw persisted strings; `:23` raises unresolved whenever any candidate exists; `quantity: null` (X-14/F-11)
- **Change:** classify on canonical form per §10.9 (`RESERVED→CONFIRMED`, `NO-SHOW→NO_SHOW`, `IN-HOUSE/IN_HOUSE→CHECKED_IN`); `quantity = candidates.length` (D-0); `CHECKED_OUT`/`CANCELLED`/`WAITLIST`/no-availability `PENDING` non-consuming; genuinely unclassifiable ⇒ UNRESOLVED (fail-closed preserved)
- **Why:** D-0 + D-1; extract §4.5 says this is what unblocks Phase 3 from Phase 1 output
- **Spec ref:** §10.9, §7, extract §4.5/§7
- **Deps:** T-09 (alias mapping)
- **Tx:** read path
- **Locking:** none
- **Idempotency:** n/a
- **Tests:** snapshot resolves on live-shaped data; unclassifiable still UNRESOLVED
- **Migration:** none; **MUST NOT** invent statuses
- **Risk:** MEDIUM

---

### Wave R5 — Lifecycle integration

> Each task below shares the same contract unless stated: **one transaction (T-02) · journal claim first (T-03) · state lock (T-05) · balance locks (existing) · `reservations` lock last (T-06) · port call inside the tx · `IdempotentCommand` (T-21) · no schema change.**

---

**T-12 — Create**
- **File:** `reservations/application/commands/create-reservation/create-reservation.handler.ts:21`, `infrastructure/repositories/reservation.repository.ts:409-516`
- **State:** no inventory consumption at all (X-1)
- **Change:** assert the full required date set before the consuming state commits; hold ⇒ assert hold set; no-availability PENDING ⇒ assert nothing; set quantity `1` implicitly via the port
- **Why:** §10.1, §11 r199/200, §32.8
- **Deps:** R2, R3, R4
- **Tx:** one tx covering reservation insert + assertion
- **Locking:** full D-24 order
- **Idempotency:** HTTP key → journal claim
- **Tests:** F-01, F-02, F-03
- **Migration:** none; **MUST NOT** add a quantity field
- **Risk:** HIGH

---

**T-13 — Modify dates**
- **File:** `create`-adjacent: `update-reservation/update-reservation.handler.ts:17`, `update-reservation.dto.ts:4-17`, `reservation.repository.ts:518-601`
- **State:** DTO snake_case vs repo camelCase ⇒ dates never persist; CRS runs in a second tx (X-2, X-3)
- **Change:** align the payload contract; single tx; `replaceInTransaction` for the date delta; retain prior commitment until replacement succeeds
- **Why:** §11, §12.2, §21 Replacement
- **Deps:** T-02, T-04
- **Tx:** one tx (replaces the two-tx split)
- **Locking:** full order
- **Idempotency:** command key
- **Tests:** F-04
- **Migration:** none
- **Risk:** HIGH

---

**T-14 — Room-type change**
- **File:** `change-room-type/change-room-type.handler.ts:23,30`
- **State:** silent no-op; no tx (X-2)
- **Change:** persist the type change and run `replaceInTransaction` across the old/new room-type balance key set
- **Why:** §10.8, §11 r218, §19
- **Deps:** T-13 (payload contract)
- **Tx:** one tx
- **Locking:** cross-room-type sorted balance locks (existing `:350-354`)
- **Idempotency:** command key
- **Tests:** F-05, F-16
- **Migration:** none
- **Risk:** HIGH

---

**T-15 — Cancel**
- **File:** `cancel-reservation/cancel-reservation.handler.ts:38`, `reservation.repository.ts:605-634`
- **State:** `crs.releaseInventory` opens a **separate** tx inside the repo tx (X-4); idempotent early-return is unreachable because transition validation precedes it
- **Change:** replace legacy release with `releaseInTransaction` on the **exact** assertion inside the same tx; validate after the idempotency check so replay returns the recorded result
- **Why:** §10.2, §16, §21
- **Deps:** T-02…T-06
- **Tx:** release + `CANCELLED` commit together or neither
- **Locking:** full order
- **Idempotency:** reorder so replay works
- **Tests:** F-06; concurrency cancel+modify (Case C)
- **Migration:** none
- **Risk:** HIGH

---

**T-16 — No-show**
- **File:** `process-no-show/process-no-show.handler.ts:23`, `reservation.repository.ts:636-655`
- **State:** releases through the legacy path in a separate tx (X-4)
- **Change:** release **remaining unelapsed nights only** via `releaseInTransaction`; never the elapsed arrival night; single tx with the `NO-SHOW` status write
- **Why:** §10.6, §17, §12.2 (D-3 — the legacy full-range release is a defect)
- **Deps:** T-02…T-06
- **Tx:** one tx
- **Locking:** full order
- **Idempotency:** durable Night Audit identity (T-31) + journal
- **Tests:** F-07; replay must not double-release
- **Migration:** none
- **Risk:** HIGH

---

**T-17 — Early departure**
- **File:** `front-office/application/commands/check-out/check-out.handler.ts:36,207-255`, `front-office/dto.ts:76`
- **State:** rewrites `departure_date`, releases nothing, billing work outside tx (X-7)
- **Change:** when `earlyDeparture` is set, `replaceInTransaction` removing `[E, scheduled departure)` atomically with the shortened dates; move the mutation under the single tx; leave billing/folio outside (separate domain)
- **Why:** §10.4, §13, §32.27 (D-3-adjacent)
- **Deps:** T-02…T-06
- **Tx:** dates + replacement in one tx
- **Locking:** full order
- **Idempotency:** command key; completed early departure is idempotent
- **Tests:** F-08
- **Migration:** none
- **Risk:** HIGH

---

**T-18 — Normal checkout (verify-only)**
- **File:** same handler, `earlyDeparture=false`
- **State:** no release (correct)
- **Change:** **none** — assert that no Availability call is issued
- **Why:** §10.3, §32.26 — a redundant release would violate the spec
- **Deps:** none
- **Tx:** unchanged
- **Locking:** none required
- **Idempotency:** unchanged
- **Tests:** F-09 (negative assertion)
- **Migration:** none
- **Risk:** LOW — **EXISTING AND COMPLIANT**

---

**T-19 — Extend / overstay**
- **File:** `reservations/…/extend-stay/extend-stay.handler.ts:23,35`; `front-office/…/extend-stay.handler.ts:33,58`
- **State:** Reservations side is a silent no-op; FO side is `isAvailable` check-only, never asserts; **two colliding command classes** (X-2, X-8, X-16)
- **Change:** assert added nights `[old departure, requested departure)` **before** committing extended dates; failure ⇒ original dates and assertion unchanged; single owner (see T-33)
- **Why:** §10.5, §15, §32.28/29 (D-6 — commit-first-and-swallow is a defect)
- **Deps:** T-02…T-06, T-33
- **Tx:** one tx; assert-then-commit ordering inside it
- **Locking:** full order
- **Idempotency:** request identity reused on retry
- **Tests:** F-10, F-11
- **Migration:** none
- **Risk:** HIGH

---

**T-20 — Reinstate**
- **File:** `reinstate-reservation/reinstate-reservation.handler.ts:23,27,31`
- **State:** silent no-op; gate tests `'NO_SHOW'` against stored `'NO-SHOW'` ⇒ no-shows never eligible; duplicate FO command (X-2, X-18, X-16)
- **Change:** fix the gate to canonical form (§10.9); fresh evaluation; **new assertion identity** before restoring the consuming state; failure leaves source state unchanged; single owner (T-33)
- **Why:** §10.7, §18, §11 r219/220
- **Deps:** T-02…T-06, T-33
- **Tx:** assertion + state restoration in one tx
- **Locking:** full order
- **Idempotency:** command key; released assertion never reused
- **Tests:** F-12, F-13
- **Migration:** none
- **Risk:** HIGH

---

**T-21 — Waitlist join / promote**
- **File:** `join-waitlist/join-waitlist.handler.ts:23`, `promote-from-waitlist/promote-from-waitlist.handler.ts:23`, `reservation.repository.ts:834-891`
- **State:** join flips state without releasing the hold; promote re-confirms without asserting; transition validated outside the tx (X-9)
- **Change:** join ⇒ release the hold assertion when leaving the consuming context; promote ⇒ assert **before** the consuming state commits; validate inside the tx
- **Why:** §10.1, §11 r205/r212, §32.9
- **Deps:** T-02…T-06
- **Tx:** one tx per item
- **Locking:** full order
- **Idempotency:** command key
- **Tests:** F-19
- **Migration:** none
- **Risk:** MEDIUM

---

**T-22 — Batch status update**
- **File:** `batch-update-status/batch-update-status.handler.ts:17`, `reservation.repository.ts:699-725`
- **State:** `findMany` + `updateMany` with **no transaction** and no inventory release; check-then-act TOCTOU (X-5)
- **Change:** per-Reservation interactive transaction; each item applies the §11 transition with its Availability effect; explicit per-item results; no cross-item atomicity, no item half-mutated
- **Why:** §32.23, §20, §21, D-9 (explicitly names `batchUpdateStatus`)
- **Deps:** T-02…T-06, T-15, T-16
- **Tx:** one per item
- **Locking:** full order per item
- **Idempotency:** per-item key derived from batch id + reservation id
- **Tests:** F-18
- **Migration:** none
- **Risk:** HIGH

---

**T-23 — Auto-cancel sweep**
- **File:** `execute-auto-cancel-sweep/execute-auto-cancel-sweep.handler.ts:35,98,153,167-183`
- **State:** raw SQL, no tx, bypasses `repo.cancel`, no release, no status guard, re-run re-emits events, casts through `(this.repo as any).prisma` (X-6)
- **Change:** route each selected item through the cancellation contract (exact-assertion release + `CANCELLED`) in one transaction; add a status guard so a re-run is a no-op; keep selection policy **outside** the domain contract (spec §11 does not define it)
- **Why:** §11 auto-cancel row, §10.2, D-9 (explicitly names the sweep)
- **Deps:** T-02…T-06, T-31 (durable scheduled identity)
- **Tx:** one per item
- **Locking:** full order per item
- **Idempotency:** `hotelId:businessDate:jobType:reservationId`
- **Tests:** F-17
- **Migration:** none
- **Risk:** HIGH

---

**T-24 — Active Reservation delete prohibition**
- **File:** `delete-reservation/delete-reservation.handler.ts:19`, `reservation.repository.ts:660-697`
- **State:** deletes an active Reservation with a legacy release in the same tx; six child deletes swallow failures with `.catch(()=>{})` (X-19)
- **Change:** refuse hard delete for a non-terminal Reservation; direct the caller to cancellation; remove failure-swallowing so partial child deletion cannot report success
- **Why:** §10.2 (Decision 12)
- **Deps:** T-15 (so cancellation is available as the alternative)
- **Tx:** unchanged for permitted deletes
- **Locking:** unchanged
- **Idempotency:** command key
- **Tests:** F-20
- **Migration:** none — **MUST NOT** rely on `ON DELETE RESTRICT` as the control (spec §10.2: it is only a partial guard)
- **Risk:** MEDIUM

---

**T-25 — Room-type upgrade**
- **File:** `front-office/application/commands/upgrade-room/upgrade-room.handler.ts`, `front-office/domain/services/upgrade.service.ts`
- **State:** check-only (`inventoryDomain.isAvailable`), no atomic replacement
- **Change:** `replaceInTransaction` across old/new room-type balances; old commitment retained if replacement fails; same-type physical move ⇒ **no Availability operation**
- **Why:** §10.8, §11 r218, §19
- **Deps:** T-02…T-06
- **Tx:** one tx
- **Locking:** cross-room-type sorted balance locks
- **Idempotency:** command key
- **Tests:** F-15, F-16
- **Migration:** none
- **Risk:** MEDIUM

---

**T-26 — Mass update execute**
- **File:** `mass-update-execute/mass-update-execute.handler.ts:23,50,90-94,191-199`
- **State:** per-item loop, raw `inventory` SELECT, no reserve/release, no tx, silent 1000-row truncation
- **Change:** per-item transaction with the Availability effect of the resulting status/date/type change; explicit partial results; surface truncation rather than dropping rows silently
- **Why:** §12.2, §32.23, §21
- **Deps:** T-02…T-06, T-13, T-15
- **Tx:** one per item
- **Locking:** full order per item
- **Idempotency:** per-item key
- **Tests:** F-21
- **Migration:** none
- **Risk:** MEDIUM

---

### Wave R6 — Idempotency

---

**T-27 — `IdempotencyPipe` mark-after-execute**
- **File:** `common/cqrs/pipes.ts:67-86`
- **State:** marks before execution with a 24 h TTL and no failure release ⇒ a failed command is permanently deduped as 409
- **Change:** mark after execution; record success/failure; replay returns the recorded outcome; conflict on hash mismatch (extract §5.3 invariants)
- **Why:** §22, D-15
- **Deps:** T-04
- **Tx:** n/a (Redis-side)
- **Locking:** n/a
- **Idempotency:** **core change**
- **Tests:** idempotency matrix
- **Migration:** none
- **Risk:** MEDIUM

---

**T-28 — Register HTTP `IdempotencyInterceptor`**
- **File:** `core/interceptors/idempotency.interceptor.ts:13,20`, `app.module.ts:109-110`
- **State:** never registered; header name mismatch; no verb skip-list
- **Change:** register globally for mutating verbs; accept `Idempotency-Key` **and** `x-idempotency-key`; skip safe verbs; `idempotency_keys` table already exists
- **Why:** §22 HTTP row (D-15)
- **Deps:** R1 (table exists)
- **Tx:** request-scoped
- **Locking:** n/a
- **Idempotency:** **core change**
- **Tests:** replay, conflict, passthrough-without-key
- **Migration:** table exists — no authoring
- **Risk:** MEDIUM (global interceptor; must not break GETs)

---

**T-29 — Frontend key generation and reuse**
- **File:** `apps/web/lib/api/client.ts:74-88,94-159`, `apps/web/services/api.ts:29-35`, `apps/web/components/Providers.tsx:21-23`
- **State:** no idempotency reference anywhere in `apps/web`; mutations retry once without a key (F-5)
- **Change:** generate a key per logical mutation, attach `Idempotency-Key`, **reuse the same key across retries**; stable across React Query retries
- **Why:** §22; D-15 consequence — a server-only implementation cannot dedup a retry that sends no key
- **Deps:** T-28
- **Tx:** client-side
- **Locking:** n/a
- **Idempotency:** **core change**
- **Tests:** frontend: retry reuses key; different mutations get different keys
- **Migration:** none
- **Risk:** MEDIUM — **frontend files affected**

---

**T-30 — Scheduled / system operation identity**
- **File:** `common/cqrs/pipes.ts:56-60` (`AuthorizationPipe`), `apps/api/src/core/context/request-context.ts:10-18`, `execute-auto-cancel-sweep.handler.ts:37-38`
- **State:** a service principal cannot be constructed: `AuthorizationPipe` rejects commands without `userId` and `requireHotelId()` needs the AsyncLocalStorage store; `?? 'SYSTEM'` is unreachable (F-15)
- **Change:** a first-class service principal that satisfies the authorization pipe and supplies hotel/business-date context; deterministic per-item operation key (§22 scheduled row)
- **Why:** §22 scheduled + Night Audit rows; without it no scheduled Availability mutation can ever run
- **Deps:** T-04
- **Tx:** unchanged
- **Locking:** n/a
- **Idempotency:** durable identity creation
- **Tests:** duplicate scheduled operation executes once
- **Migration:** none
- **Risk:** MEDIUM — must not weaken the authorization model; scope strictly to service identity

---

**T-31 — Adopt `IdempotentCommand`**
- **File:** create, update, change-room-type, cancel, process-no-show, extend-stay, reinstate-reservation, batch-update-status, execute-auto-cancel-sweep, delete, join-waitlist, promote-from-waitlist, upgrade-room, check-out, mass-update-execute command classes
- **State:** zero implementors (F-3); `IdempotencyPipe` inert
- **Change:** `extends IdempotentCommand` + `readonly idempotencyKey` on each inventory-affecting command
- **Why:** §22 command-level idempotency (D-15)
- **Deps:** T-27
- **Tx:** n/a
- **Locking:** n/a
- **Idempotency:** **core change**
- **Tests:** duplicate command dispatch
- **Migration:** none
- **Risk:** LOW per command; MEDIUM in aggregate

---

### Wave R7 — Command ownership

---

**T-32 — Delete Front Office duplicate commands (D-23)**
- **File:** `front-office/application/commands/extend-stay/extend-stay.command.ts`, `…/extend-stay.handler.ts`, `front-office/application/commands/reinstate/reinstate.command.ts`, `…/reinstate.handler.ts`, `front-office.module.ts:89,93,154,158,210,214`
- **State:** both names registered twice; FO wins last-wins; Reservations routes dispatch an incompatible payload (F-13/X-16)
- **Change:** delete the FO classes, providers and registrations; keep Reservations as sole owner; FO service maps its DTO onto the Reservations payload
- **Why:** D-23 (supersedes D-18)
- **Deps:** T-19 and T-20 must be correct **first**
- **Tx:** unchanged
- **Locking:** unchanged
- **Idempotency:** unchanged
- **Tests:** invert `command-bus-collision.spec.ts` to assert a single owner and no collision; both FO routes succeed
- **Migration:** none
- **Risk:** MEDIUM — sequencing critical (R7 gate)

---

**T-33 — Permission decorators on FO routes**
- **File:** `front-office/front-office.controller.ts:93,103`
- **State:** no `@Permission` decorator, unlike `reservations.controller.ts:171,178`
- **Change:** add `RESERVATION_REINSTATE` / `RESERVATION_EXTEND`
- **Why:** ownership parity once FO becomes an initiator only
- **Deps:** T-32
- **Tx:** n/a · **Locking:** n/a · **Idempotency:** n/a
- **Tests:** unauthorized call rejected
- **Migration:** none · **Risk:** LOW

---

**T-34 — Registry-wide duplicate commandName audit**
- **File:** `common/cqrs/command-bus.ts:20-27`; all `*.module.ts` registrations
- **State:** last-wins silently overwrites; only two collisions are known
- **Change:** detect and report every duplicated `commandName` at startup; fail fast or warn explicitly
- **Why:** D-23's mechanism is general
- **Deps:** T-32
- **Tx:** n/a · **Locking:** n/a · **Idempotency:** n/a
- **Tests:** synthetic duplicate registration is reported
- **Migration:** none · **Risk:** LOW

---

### Wave R8 — Verification

---

**T-35 — Case C regression test (mandatory)**
- **File:** `availability/infrastructure/__tests__/availability-assertion-concurrency.postgres.spec.ts:243-303`
- **State:** currently documents the **unsafe** behaviour
- **Change:** invert to assert the locked behaviour
- **Assertion:** **concurrent Cancel + Modify must not allow `RELEASED` to be overwritten while the Reservation still holds inventory**
- **Spec ref:** §23 row 1, §8, D-24
- **Deps:** T-05, T-06, T-15, T-13
- **Tx/lock/idempotency:** n/a (test)
- **Migration:** none · **Risk:** LOW

---

**T-36 — Lock-order probe test**
- **File:** new spec alongside the concurrency suite
- **Change:** assert the observed acquisition order is claim → state → balances → reservations
- **Spec ref:** §23 (D-24) · **Deps:** T-05, T-06 · **Risk:** LOW

---

**T-37 — Full matrix execution**
- **File:** §16 suites
- **Change:** implement/run functional, idempotency, concurrency, isolation and fail-closed suites
- **Spec ref:** whole specification · **Deps:** all · **Risk:** LOW

---

## 22. Traceability Matrix

`Locked clause → task(s) → test(s) → gate`

| Spec clause | Decision | Task(s) | Test(s) | Gate |
|---|---|---|---|---|
| §7 quantity = 1 | D-0 | T-11 (`quantity = candidates.length`), T-12 | F-01 | G9 |
| §9 exact assertion linkage | row 8 | T-05, T-06 | F-06, F-12 | G4 |
| §10.1 consuming contexts | row 1 | T-12, T-21 | F-01/F-02/F-19 | G4 |
| §10.2 cancellation + delete ban | row 12 | T-15, T-24 | F-06, F-20 | G4 |
| §10.3 checkout boundary | row 4 | T-18 (verify) | F-09 | G9 |
| §10.4 early departure | row 3 | T-17 | F-08 | G4 |
| §10.5 overstay | row 5 | T-19 | F-10, F-11 | G4 |
| §10.6 no-show | row 2, D-3 | T-16 | F-07 | G4 |
| §10.7 reinstatement | row 6 | T-20 | F-12, F-13 | G4 |
| §10.8 upgrade/transfer | row 13 | T-25 | F-15, F-16 | G4 |
| §10.9 status classification | D-1 | T-09, T-11 | status unit tests | G9 |
| §11 transition matrix | — | T-12…T-26 | F-01…F-21 | G4 |
| §12.2 date sets | — | T-13, T-16, T-17, T-19 | F-04, F-07, F-08, F-10 | G4 |
| §20 multi-room / batch | row 11 | T-22, T-26 | F-18, F-21 | G4 |
| §21 atomicity + synchronicity | D-9 | T-02, T-12…T-26 | F-01…F-21 | G4 |
| §21 unknown outcome | — | T-03, T-27 | idempotency matrix | G6 |
| §22 idempotency | D-15 | T-04, T-27, T-28, T-29, T-30, T-31 | idempotency matrix | G6 |
| §23 lock order | D-24 | T-03, T-05, T-06, T-36 | lock-order probe, Case C | G2 |
| §24 failure / fail-closed | — | T-19, T-20 | fail-closed suite | G8 |
| §26 tenancy | — | (existing guards) | isolation suite | G8 |
| §27 population | D-21 | T-07, T-10 | G3/G5 tests | G3, G5 |
| §28 command ownership | D-23 | T-32, T-33, T-34 | collision spec inverted | G7 |
| §32 invariants 1-30 | — | all | full matrix | G8, G9 |
| §34.5.2 `IN_HOUSE` | — | T-09 | status unit tests | G9 |
| Extract §4.5 / §7 | D-0, D-1 | T-11 | snapshot resolves | G5 |

---

## Self-Review (Stage D)

| Metric | Value |
|---|---|
| Total implementation tasks | **37** (T-01 … T-37) |
| Production files affected (API) | **~34** — `pipes.ts`, `cqrs.module.ts`, `command-bus.ts`, `reservation-operation-journal.ts`, `availability-assertion.service.ts`, `reservation-consumption.adapter.ts`, `availability-source.adapter.ts`, `reservation.repository.ts` (+ interface), `reservation-state-machine.ts`, `idempotency.interceptor.ts`, `app.module.ts`, `request-context.ts`, `reservations.module.ts`, `front-office.module.ts`, `front-office.controller.ts`, and the create/update/change-room-type/cancel/process-no-show/extend-stay/reinstate-reservation/batch-update-status/execute-auto-cancel-sweep/delete-reservation/join-waitlist/promote-from-waitlist/mass-update-execute commands + handlers, `check-out.handler.ts`, `upgrade-room.handler.ts`/`upgrade.service.ts`, `front-office` extend/reinstate services, `reservation-consumption.adapter.ts` |
| Production files affected (web) | **3** — `lib/api/client.ts`, `services/api.ts`, `components/Providers.tsx` |
| Test files affected | **~14** — inverting `availability-assertion-concurrency.postgres.spec.ts`, `command-bus-collision.spec.ts`; new lock-order, population, dual-count, Case C regression, idempotency and isolation suites; updates to adapter, journal, foundation and handler specs |
| Schema / migration files affected | **0 authored.** 1 migration **deployment** (18 pending) + 4 drift reconciliations; `schema.prisma` untouched |
| Frontend files affected | 3 (above) |
| Deferred items | 12 (§20) |
| Unresolved dependencies | 3: (a) drift reconciliation must precede migration apply; (b) T-19/T-20 must precede T-32 or FO routes strand; (c) scheduled identity (T-30) must precede auto-cancel/no-show scheduling |
| Conflicts recorded | **22** (X-1…X-22), all in §4.4; none silently adapted |
| Contradictions with the locked specification | **None found** |

**Classification audit:** of the 22 change-inventory entries, 20 are MUST IMPLEMENT, 0 MUST NOT CHANGE entries are scheduled for modification (quantity, status taxonomy, balance rules, linkage, snapshot arithmetic are all untouched), 2 are DEFERRED, and 6 components are explicitly marked EXISTING AND COMPLIANT (check-in, normal checkout, balance sort order, port binding, transaction manager, tenancy/fail-closed behaviour) so they are **not** converted into tasks.

---

**Stage D output complete. The Implementation Plan is READY FOR STAGE D READINESS REVIEW.**

No task has been implemented. Implementation begins only after a separate Readiness Review approval.

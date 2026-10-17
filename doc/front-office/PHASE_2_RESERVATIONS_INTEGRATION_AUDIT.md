# XYLO Front Office — Phase 2: Reservations ↔ Front Office Integration Audit

- **Date:** 2026-10-17
- **Phase:** Phase 2 — Reservations Integration & Linkage Audit (**AUDIT ONLY**)
- **Status:** see `docs/front-office/PHASE_2_BASELINE_AND_EXIT_GATE.md` for the exit declaration
- **Related deliverables:**
  - `docs/front-office/PHASE_2_INTEGRATION_GAP_REGISTER.md`
  - `docs/front-office/PHASE_2_INTEGRATION_MAP.md`
  - `docs/front-office/PHASE_2_BASELINE_AND_EXIT_GATE.md`
- **Phase 1 inputs consumed (not re-audited):**
  - `docs/front-office/PHASE_1_CURRENT_STATE_AUDIT.md`
  - `docs/front-office/PHASE_1_GAP_REGISTER.md`
  - `docs/front-office/PHASE_1_BASELINE_AND_EXIT_GATE.md`
- **Authoritative Reservations artifacts consumed (not re-audited):**
  - `docs/audit/Reservations/04_BACKEND_ARCHITECTURE.md`, `05_FRONTEND_ARCHITECTURE.md`, `07_DECISIONS.md` (ADR-072/073/074), `10_CODEBASE_MAP.md`, `12_API_CONTRACT.md`
  - `docs/enterprise/reservations-roadmap-v2.md`, `docs/RESERVATIONS_FORENSIC_AUDIT.md`

This document records only what is observable in the working tree at the time of audit. It changes no application code, tests, schema, migration, DB object, configuration, dependency, or lockfile.

---

## 1. Executive Summary

The Reservations → Front Office hand-off and the Front Office → Reservations write-back are **structurally sound in shape but inconsistent in identifier and contract detail**. The dominant architecture is correct and, in the one place it is asserted by test, verified:

- Front Office initiates the **owner's** commands for delegated operations (`ExtendStayCommand`, `ReinstateReservationCommand`), and room-assignment writes converge through the single-writer `IReservationRepository` (`command-ownership.spec.ts` — PASS).
- Check-in room assignment and the check-out status transition both route through Reservations-owned writers (`applyCheckIn` via the check-in flow aggregate, `applyCheckOut`).

The defects are integration-seam defects, not architecture defects:

1. **Identifier drift at the check-in guest panel** — the same dialog uses the reservation UUID for the check-in flow but the human confirmation number for the parties lookup, so the "Accompanying Guests & Shares" section can never resolve (`FO-P2-001`).
2. **Mark No-Show from the Front Office is non-functional** — it posts an undeclared field to the generic reservations update, which the global `forbidNonWhitelisted` validation pipe rejects, and it never calls the dedicated no-show command (`FO-P2-002`).
3. **Check-out has no idempotency protection over financial side effects** — refunds, penalties, minibar, payment and voucher redemption run *before* the status-transition guard and outside the transaction, with no HTTP idempotency key sent by the checkout modal (`FO-P2-003`).
4. **The common `@Permissions` guard cannot authorize non-admins** because `request.user.permissions` is never hydrated — 9 Front Office routes are effectively admin-only (`FO-P2-005`).
5. **Two independent folio families with colliding keys** are merged only client-side at checkout; the Front Office check-out writes only the legacy family (`FO-P2-006`).
6. **The cross-reservation overlap safeguard in room assignment is unreachable** because its input is never hydrated (`FO-P2-007`).

**Finding totals (Phase 2): P0 = 0, P1 = 0, P2 = 6, P3 = 4, Info = 3 (13 findings).** See `PHASE_2_INTEGRATION_GAP_REGISTER.md`.

---

## 2. Scope, Method, and Evidence Discipline

**In scope:** the contract between `apps/api/src/modules/reservations/**` and `apps/api/src/modules/front-office/**`, and the corresponding `apps/web/features/reservations/**` ↔ `apps/web/features/front-office/**` hand-off, across: navigation/entry points; API contract alignment; domain operations; lifecycle/state transitions; persistence ownership; availability; room assignment; billing/folio dependencies; permissions/property scope; cross-module error/state sync.

**Out of scope (already audited elsewhere, reused not repeated):** the internal completeness of the Reservations rebuild (Phase 0–5), and the Front Office current-state baseline (Phase 1).

**Evidence classes used per finding:** `direct` (source read in this phase), `test-derived` (an executed focused test), `documentation-derived`, `circumstantial`. **Confidence:** high / medium / low.

**Method:** static source tracing of full call chains (frontend action → URL/query → API client → controller → facade → command/query bus → handler → repository/persistence), plus focused single-worker test runs (see §11). No runtime API calls were made. No DB was read or written.

### 2.1 Call-chain shape (confirmed)

```
web reservations action
  → buildCheckInHref(reservationId)                  [UUID]
  → /front-office?tab=arrivals&dialog=check-in&reservation=<UUID>
  → useFrontOfficeUrlSync() → setDialogReservationId(<UUID>)
  → OperationDialogHost → <CheckInDialog reservationId=<UUID>>
  → use-check-in-workspace hooks → check-in.api.ts
  → POST/GET /api/v1/check-in/*  (@Controller('check-in'))
  → CommandBus/QueryBus → CheckIn* handlers
  → CheckInWorkflowService.loadReservation → prisma.reservations WHERE id=<UUID>
```

The Front Office HTTP controller class `FrontOfficeController` (`front-office.controller.ts:26-33`) does **not** depend on `FrontOfficeService` directly; its first dependency is the facade `CrsFrontDeskIntegrationService` (`crs-front-desk-integration.service.ts:14-20`), which delegates to `FrontOfficeService`. This extra facade layer adds a CRS restriction check on `changeRateCode` only (`crs-front-desk-integration.service.ts:97-115`) and is otherwise a pass-through.

---

## 3. Area A — Entry Points & Navigation

### 3.1 Reservations → Front Office check-in hand-off (confirmed end-to-end)

- `buildCheckInHref(reservationId)` builds `/front-office?tab=arrivals&dialog=check-in&reservation=<reservationId>` (`apps/web/features/reservations/model/reservation-actions.ts:124-131`).
- The action is offered only for `confirmed`/`guaranteed` statuses and only on the arrival day (`reservation-actions.ts:235-236`, `:145-148`).
- Call sites pass the reservation **UUID**: `ReservationDetailWorkspace.tsx:99-100` (`router.push(buildCheckInHref(r.id))`), and `ReservationsHomeView.tsx:308-310` (`const resId = r.id`). `Reservation.id` is mapped from the API row's `id` (`apps/web/features/reservations/api/reservation.api.ts:68-71`), which is the `reservations.id` UUID (created as `crypto.randomUUID()` at `apps/api/src/modules/reservations/infrastructure/repositories/reservation.repository.ts:486`).
- The FO URL-sync hook accepts `check-in` as a dialog param and reads `reservation` into `dialogReservationId` (`apps/web/features/front-office/hooks/use-front-office-url-sync.ts:10-24`; `store` setter used by `ArrivalPanel.tsx:25`).
- `OperationDialogHost` renders `<CheckInDialog ... reservationId={dialogReservationId} />` for `activeDialog === 'check-in'` (`OperationDialogHost.tsx:57`).

**Result:** the navigation hand-off contract is honored end-to-end for the check-in dialog. The only FO E2E test asserting it (`e2e/kebab-actions.spec.ts:233`) checks navigation/routing only; it was not executed here (requires live servers).

### 3.2 Other Front Office entry points (confirmed)

- Rack/room-grid and arrivals read models return reservation **UUIDs**: `GetRoomGridQueryHandler` sets `reservationId: active?.id` (`front-office.query.handlers.ts:240-242`) and `GetArrivalsQueryHandler` returns `id: r.id` from the `reservations` row (`front-office.query.handlers.ts:370-394`). `ArrivalPanel.tsx:94` calls `markNoShow.mutate(a.id)`; `QuickActions`/`ArrivalPanel` resolve a room by `r.reservationId === reservationId`.
- The URL `reservation=` deep-link only drives the 7 dialog kinds whose components accept a `reservationId` prop (`OperationDialogHost.tsx:57-63,67`). `reverse-check-in`, `reinstate` and `redeem-voucher` are rendered with only `room={room}` (`:64-66`) and derive the reservation from the selected room's `reservationId` (`ReverseCheckInDialog.tsx:364,491`; `ReinstateDialog.tsx:365,383`), ignoring `dialogReservationId`. This is an observation, not a defect (`FO-P2-011`).

---

## 4. Area B — API Contract Alignment

### 4.1 Check-in staged flow (confirmed)

`check-in.controller.ts` (11 routes under `/check-in`) drives the staged flow; all stage handlers load the reservation by UUID (`check-in/application/services/check-in-workflow.service.ts:39-46`: `prisma.reservations.findFirst({ where: { id: reservationId, hotel_id } })`). The `CheckInDialog` uses `rid` — the URL-provided `reservationId` — for every flow stage (`CheckInDialog.tsx:205,252,266,285,298,312`). Confirmed consistent.

### 4.2 Guest / shares panel — **identifier mismatch (confirmed defect, `FO-P2-001`)**

The same `CheckInDialog` passes the **confirmation number** (not the UUID) to the guest list:

- `CheckInDialog.tsx:556` → `<CheckInGuestList reservationId={reservation.confirmationNo || ''} />`
- `CheckInGuestList.tsx:73` → `useReservationParties(reservationId)`
- `use-reservation-queries.ts:217-223` → `GET /reservations/${reservationId}/parties`
- `reservations.controller.ts:281-286` → `reservationsService.getParties(id)`
- `reservation.repository.ts:1458-1463` → `SELECT * FROM reservation_guests WHERE reservation_id = $1 AND hotel_id = $2`
- `reservation_guests.reservation_id` is written as the reservation **UUID** (`reservation.repository.ts:1468` `data.reservationId`).

`reservation.confirmationNo` in the eligibility response is `reservations.confirmation_no` — a separate, nullable column (`check-in-workflow.service.ts:122`: `confirmationNo: row.confirmation_no ?? null`) — not the UUID. Unless `confirmation_no` happens to equal the UUID (it does not by construction; the reservation is created with `id = crypto.randomUUID()` and `confirmation_no` is a distinct optional field — `create-reservation.dto.ts:13`), the parties query returns `[]`, and `CheckInGuestList` renders `null` (it returns nothing when there is no primary/companion/share — `CheckInGuestList.tsx:75-90` region). **Net effect: the "Accompanying Guests & Shares" section is silently empty.** Severity **P2**, confirmed defect, high confidence (direct source; runtime not executed).

### 4.3 Mark No-Show — **non-whitelisted field (confirmed defect, `FO-P2-002`)**

- FE: `ArrivalPanel.tsx:94` → `useMarkNoShowMutation` (`use-front-office-mutations.ts:136-143`) → `frontOfficeApi.markNoShow` → `PUT /reservations/:id` with body `{ reservation_status: 'NO-SHOW' }` (`front-office.api.ts:208-210`).
- BE: `reservations.controller.ts:100-106` `@Put(':id')` binds `UpdateReservationDto`; `update-reservation.dto.ts:3-18` declares **no** `reservation_status`/`status` property.
- The global pipe is `whitelist: true, forbidNonWhitelisted: true` (`main.ts:28-40`).

Consequently the request is rejected at validation (HTTP 400) for a permitted caller (and 403 for an unpermitted one). It also **never reaches** the dedicated `POST /reservations/:id/no-show` → `ProcessNoShowCommand` (`reservations.controller.ts:123-126`; `reservations.service.ts:110-116`). Severity **P2**, confirmed defect, high confidence (config/DTO derived; runtime not executed).

### 4.4 Upgrade — **contract mismatch (confirmed defect, `FO-P2-004`)**

- FE posts `{ newRoomType, toRoomNo?, reason? }` to `/front-office/:id/upgrade` (`front-office.api.ts:135-137`; `UpgradeRequest` includes `toRoomNo?` — `mutation.types.ts:45-48`; schema `upgradeSchema.toRoomNo` — `room-ops.schemas.ts:9-14`).
- BE DTO carries `newRoomType`, `toRoomNo?`, `reason?` (`dto.ts:98-102`), and the controller spreads the body (`front-office.controller.ts:118-122`).
- `FrontOfficeService.upgradeRoom(data)` reads `data.newRoomId` — which the DTO never supplies — and passes it as `UpgradeRoomCommand(..., newRoomId=undefined, ...)` (`front-office.service.ts:112-113`).

Therefore the room-pinning branch is never taken over HTTP; the operation behaves as the (documented) **category-based** upgrade, changing room type only (`utils/upgrade.helpers.ts:7-12`). The latent bug is that the command's `newRoomId` path would use a room **number** where a room **UUID** is expected (`upgrade-room.handler.ts:119-132` region). Severity **P3** (works as documented today; would become **P2** if room-pinning is intended), confirmed defect, high confidence.

### 4.5 Clients and DTO validation

- The FO API client posts bodies verbatim (`apps/web/services/api.ts:109-115`); it does **not** attach an idempotency header unless a caller explicitly mints one (§5.2).
- Phase 1 already recorded that several FO mutating routes use `@Body() any`/inline bodies (`FO-P1-005`); this phase did not re-audit that, but notes it compounds the `FO-P2-002` class of contract drift.

---

## 5. Area C — Lifecycle & State Transitions

### 5.1 Shared vocabulary and guard (confirmed)

- Canonical statuses and allowed transitions: `packages/shared/src/reservation-state-machine.ts:18-42`. Notably `CHECKED_OUT → ['CHECKED_IN']` only — so `CHECKED_OUT → CHECKED_OUT` is **illegal** (`:39`).
- The single enforcement point is `ReservationStatusService.assertTransition` (`reservations/domain/services/reservation-status.service.ts:30-36`), used by the check-out handler at `check-out.handler.ts:235`.

### 5.2 Check-out ordering vs. the transition guard (confirmed defect, `FO-P2-003`)

In `check-out.handler.ts`, all financial side effects run **before** the status-transition assert (`:235`) and **outside** the status-transition transaction:

| Step | Line | Runs before assert (235)? | Outside the tx? |
|---|---|---|---|
| Early-departure refund `postCredit` | `:117` | yes | yes |
| Early-departure penalty `postCharge` | `:144` | yes | yes |
| Minibar `postCharge` | `:166` | yes | yes |
| Payment `postPayment` | `:189` | yes | yes |
| Voucher credit + redemption | `:196-226` | yes | yes |
| Status transition + `applyCheckOut` | `:232-235, 388` | — | inside tx |

The client attaches **no** idempotency key: `CheckoutFlowModal.tsx:130` calls `api.post('/front-office/check-out', body)` directly, and `api.ts` only forwards an `Idempotency-Key` when a caller minted one (`api.ts:22-51,109-115`). The global `IdempotencyInterceptor` is a **no-op without that header** (`core/interceptors/idempotency.interceptor.ts:21-24`). A duplicate submit/retry therefore re-posts the refund/penalty/minibar/payment/voucher **before** the (illegal) repeat transition is rejected at `:235`. Severity **P2**, confirmed defect, high confidence (direct source; runtime not executed).

### 5.3 Delegated lifecycle operations (confirmed good)

- Extend: FO `extendReservation` computes the absolute new departure and dispatches the **Reservations** `ExtendStayCommand` (`front-office.service.ts:97-110`), then post-commits a change-log row.
- Reinstate: FO `reinstateReservation` dispatches the Reservations `ReinstateReservationCommand`, then assigns the room through the repository writer `reservationRepository.assignRoomNumber(...)` (`front-office.service.ts:118-136`).
- The ownership contract is asserted by `apps/api/src/modules/front-office/__tests__/command-ownership.spec.ts` — **PASS (8 tests)** in this phase (§11). It asserts the extend payload mapping, hotel-id-from-context, change-log ordering, and that reinstate does **not** call `prisma.reservations.update` (converges on `assignRoomNumber`).

### 5.4 Reservations-side status; No-show / Cancel ownership

Reservations exposes owner routes: `PUT/PATCH :id` (update), `POST :id/cancel`, `POST :id/no-show`, `POST :id/confirm`, `POST :id/waitlist/promote` (`reservations.controller.ts:100-199`). Front Office delegates extend/reinstate to these owners but (a) overrides no-show with the broken generic update (`FO-P2-002`), and (b) has no cancel route (cancel is a Reservations-module surface). `FO-P2-010` records that reinstate is reachable via **two** routes (`POST /reservations/:id/reinstate` and `POST /front-office/reinstate`) that converge on the same owner command.

---

## 6. Area D — Persistence Ownership / Single Writer

### 6.1 Confirmed ownership split

Front Office handlers delegate reservation lifecycle/room writes to Reservations-owned single-writer methods and do not call `prisma.reservations.update` for those concerns (`command-ownership.spec.ts:157-158,193`). Front Office does own room-state and history/audit writes directly (`check-out.handler.ts:405-407` insert into `reservation_status_history`; `front-office.service.ts:142-165` `reservation_changes`/`audit_trail`).

### 6.2 Deviations / duplicate-writer smells

- **No-show** bypasses the owner command and targets a generic update with an undeclared field (`FO-P2-002`) — a duplicate/incorrect write path even though it currently fails validation.
- **Reinstate dual route** (`FO-P2-010`) is a second entry point, but it converges on the same owner command (verified good), so it is an observation not a defect.

---

## 7. Area E — Availability / Inventory

### 7.1 Check-in consumes no availability inventory (confirmed, by design)

`check-in-availability.postgres.spec.ts:41-49` documents the F-14 invariant: the staged check-in commit path must **never** reach into Availability (no second assertion, no journal, no balance movement; the `current_assertion_id` link is unchanged). `AssignRoomCommandHandler` injects no availability port (`assign-room.handler.ts` constructor region; identifiers `DistributedLock`, `TransactionManager`, `EventBus` only).

### 7.2 Overlap safeguard is unreachable (confirmed defect/risk, `FO-P2-007`)

`RoomAssignableSpecification` declares an input `assignedToOtherReservation?: boolean` to flag a room already linked to another overlapping in-house/arriving reservation (`check-in/domain/specifications/room-assignable.specification.ts:16-17`). `AssignRoomCommandHandler` calls `roomAssignableSpec.evaluate({ room, bookedRoomType, upgradeApproval })` — **without** `assignedToOtherReservation` — both before and inside the lock (`assign-room.handler.ts:84-88` and `:99-103`). The `ROOM_ALREADY_ASSIGNED` (cross-reservation overlap) branch is therefore never reachable via the assign-room path. Double-assignment relies instead on the per-room distributed lock (`assign-room.handler.ts:95`) and the room's own status transition (`:110-112`). Severity **P2**, potential risk, high confidence (direct source; the guarding Postgres test is skipped without `AVAILABILITY_TEST_DATABASE_URL` — see §11).

### 7.3 Extend re-validates availability (confirmed)

FO extend delegates to `ExtendStayCommand`, which performs per-night availability checking server-side (per the facade comment `crs-front-desk-integration.service.ts:82` and the Reservations availability wiring documented in `docs/availability/phase-5/*`). Not re-audited here.

---

## 8. Area F — Billing / Folio / Check-out

### 8.1 Two folio families with colliding keys (confirmed risk, `FO-P2-006`)

| Family | Tables | Key shape | Written by | Read by |
|---|---|---|---|---|
| Legacy header | `folio_header`, `folio_charges`, `folio_payments`, `folio_tax_details` | `folio_header.folio_id` = `Int @id @default(autoincrement())` (`packages/db/schema.prisma:4354-4355`); `folio_payments.folio_id` = `String` (`:4390-4393`) | `BillingService.postCharge/postCredit/postPayment/closeFolio` | `GET /billing/folio/:reservationId` → `billingStore.loadFolio` (`apps/web/store/billingStore.ts:81-90`) |
| Posting ledger | `folios`, `folio_postings`, `folio_tax_details` | raw SQL; **no Prisma `folios` model** (`schema.prisma:4414-4425` comment: "no Prisma `folios` model exists") | Cashiering / group-allotment / check-out master-folio auto-post | `useReservationFolios` (cashiering) |

Within the legacy family the keying is itself inconsistent: `getFolio` reads `folio_charges` by the **integer** header id (`billing.service.ts:53-59`) but `postPayment` writes `folio_payments` with `folio_id = data.reservationId` (the **UUID** — `billing.service.ts:151-156`) and reads payments back by UUID (`:66-67`). The current-checkout UI merges the two families **client-side** (`CheckoutFlowModal.tsx:17,324`). Front Office check-out (`check-out.handler.ts`) writes only the legacy family via `BillingService`. Severity **P2**, potential risk, high confidence for the split; the client-side reconciliation logic was not runtime-verified.

### 8.2 Check-out does not gate on full settlement (confirmed defect, `FO-P2-008`)

The checkout button is disabled only when no payment method is chosen (`CheckoutFlowModal.tsx:1152`); the backend posts `paymentAmount` as provided (`check-out.handler.ts:187-193`), then closes the guest folio (`:527`), swallowing any `closeFolio` error (`:528-530`). There is no post-settlement assertion that the folio balance is zero before/after the `CHECKED_OUT` write. A residual balance can remain on a checked-out folio while the UI copy says "Check Out & Settle". Severity **P3**, confirmed defect, high confidence (direct source; this is also a policy question — see the register's dependencies).

### 8.3 Check-in payment/compliance (not exhaustively traced)

The staged check-in verifies payment/compliance through its own handlers (`check-in/application/commands/verify-payment.*`). Folio linkage for check-in postings was not traced to a single authoritative writer in this phase; marked partially verified.

---

## 9. Area G — Permissions & Property Scope

### 9.1 Guard registration (confirmed)

`APP_GUARD`s: `JwtAuthGuard`, `PropertyScopeGuard`, `MultiTenantGuard` (`app.module.ts:107-109`); `RolesGuard`, common `PermissionGuard`, `PropertyAccessGuard`, `DepartmentAccessGuard`, `WarehouseAccessGuard` (`common/authorization/authorization.module.ts:18-22`). `APP_INTERCEPTOR`s include `IdempotencyInterceptor` (`app.module.ts:110-112`).

### 9.2 Common `@Permissions` cannot authorize non-admins (confirmed defect, `FO-P2-005`)

- The common guard reads `request.user?.permissions ?? []` and otherwise allows only `SUPER_ADMIN`/`ADMIN` (`common/authorization/permission.guard.ts:14-24`).
- `request.user` is produced by `JwtStrategy.validate` → `AuthService.validateUser`, whose `select` **omits** `permissions` and returns no `permissions` field (`core/auth/jwt.strategy.ts:28-30`; `core/auth/auth.service.ts:129-140`). No code assigns `request.user.permissions` (grep found no such assignment in `apps/api/src`).

Therefore the 9 FO routes carrying the common `@Permissions('front_office:*')` decorator (`front-office.controller.ts:42,49,59,80,319,327,337,354,363`) are effectively usable only by `ADMIN`/`SUPER_ADMIN`. This complements Phase 1's `FO-P1-001` (40 undecorated routes): routes decorated with the **common** `@Permissions` are locked to admins; routes decorated with the **platform** `@Permission` are enforced via the DB resolver and work for non-admins (§9.3). Severity **P2**, confirmed defect, high confidence (direct source; runtime not executed).

### 9.3 Platform `@Permission` routes work and enforce property scope (confirmed + test-derived)

The platform `PermissionGuard` resolves permissions via `PermissionResolverService.hasAllPermissions(userId, perms, propertyId)` and refuses a `REFUSED` scope (`platform/permission/permission.guard.ts:40-54`). The FO `@Permission` routes are reinstate, transfer, extend, upgrade, change-rate (`front-office.controller.ts:101,107,113,119,125`), covered by `fo-route-permissions.spec.ts` — **PASS (17 tests)** in this phase (§11), including a property-scope-`REFUSED` rejection case.

### 9.4 Frontend/back-end permission vocabulary mismatch (confirmed defect, `FO-P2-009`)

The FE FO permission map binds every FO capability to **legacy** codes (`Permissions.WRITE_RESERVATIONS = 'write:reservations'`, `READ_RESERVATIONS`, `WRITE_BILLING`, `READ_BILLING`, `READ_RATES`) (`apps/web/features/front-office/permissions/front-office.permissions.ts:3-23`; values from `apps/web/lib/permissions/usePermissions.ts:38-48`), while the back end uses `front_office:check_in:read/write`, `front_office:room_moves:read`, and `reservations:reservation:*`. The frontend gating therefore never matches back-end codes; the two systems are governed by different vocabularies. Severity **P3**, confirmed defect, high confidence (direct source). Cross-ref `FO-P1-001`.

### 9.5 Property scope on Front Office controllers (carry-forward)

Phase 1 `FO-P1-002` recorded class-level `@PropertyScope(false)` on both FO controllers (`front-office.controller.ts:24`; `check-in/api/check-in.controller.ts:50`), which makes `PropertyScopeGuard`/`PropertyAccessGuard`/`MultiTenantGuard` short-circuit. This phase re-confirmed the decorator and that the platform `@Permission` routes are the only FO routes where scope is freshly resolved (`platform/permission/permission.guard.ts:40-44`). Not re-classified.

---

## 10. Area H — Cross-Module Error / State Sync

- **Realtime invalidation** exists via a bridge that maps domain events to query-key invalidations (e.g. `front-office-realtime.bridge.ts:32` listens to `front-office.upgrade-applied` → room-grid/dashboard/room-detail). The full event→invalidation matrix was **not** exhaustively enumerated in this phase — marked partially verified.
- **Error surfacing** differs by path: Reservations actions raise typed `ApiError` with status (used for the 403-specific toast at `ReservationsHomeView.tsx:311-315`), while several FO check-out side effects swallow their own errors (`check-out.handler.ts:140,156,181,190-191,227-228,528-530`), so a partial failure can be invisible to the operator.
- **No-show** and **check-out** silent failures are the two most operationally significant cross-module error-sync issues found (`FO-P2-002`, `FO-P2-003`, `FO-P2-008`).

---

## 11. Verified Controls (positive results)

These were checked and found correct; recorded so subsequent phases do not "fix" working integration:

1. Check-in hand-off navigation contract (`tab=arrivals&dialog=check-in&reservation=<UUID>`) is honored end-to-end (§3.1).
2. Delegated extend/reinstate route to Reservations owner commands; reinstate room write converges on `IReservationRepository.assignRoomNumber`, and `prisma.reservations.update` is not used for it — asserted by `command-ownership.spec.ts` (PASS).
3. FO `@Permission` reservation routes enforce the correct reservation permission and property scope — asserted by `fo-route-permissions.spec.ts` (PASS).
4. The reservation status state machine centrally rejects illegal transitions (`CHECKED_OUT → CHECKED_OUT` illegal), so a duplicate check-out **status** write cannot occur (`reservation-state-machine.ts:32-42` + `check-out.handler.ts:235`). The gap (`FO-P2-003`) is the un-guarded *pre-transition side effects*, not the status write.
5. Check-in consumes no availability inventory by design (F-14) — invariant documented in `check-in-availability.postgres.spec.ts:41-49`.

---

## 12. Carry-Forward From Phase 1 (cross-referenced, not duplicated)

| Phase 1 ID | Phase 2 relationship |
|---|---|
| `FO-P1-001` (undecorated FO routes) | Complemented by `FO-P2-005` (decorated-with-common-`@Permissions` routes are admin-only) and `FO-P2-009` (FE/BE vocabulary mismatch). |
| `FO-P1-002` (`@PropertyScope(false)`) | Re-confirmed; scope still resolved only on platform `@Permission` routes (§9.5). |
| `FO-P1-004` (walk-in creates CONFIRMED only) | Integration implication recorded as `FO-P2-012` (walk-in never enters the staged check-in/room-assignment path). |
| `FO-P1-006` (`reg-card`/`checkout-history` kinds unrendered) | Re-confirmed at `OperationDialogHost.tsx:57-67`; recorded as `FO-P2-013`. |
| `FO-P1-005` (`@Body() any`) | Compounds `FO-P2-002`; not re-audited. |

---

## 13. Limitations

1. **No runtime/API execution.** All guard-enforcement, validation (HTTP 400) and idempotency conclusions are derived from source/config, not from live requests.
2. **Backend FO **full** suite still not runnable.** `npx jest "modules/front-office" --runInBand` (default ts-jest, no `isolatedModules`) still OOM-crashed on the ~4 GB host. Focused runs using the in-repo sanctioned `apps/api/.jest.lowmem.json` succeed (see the baseline doc). The default-config full-suite baseline remains **NOT VERIFIED**.
3. **Postgres-gated availability spec skipped** without `AVAILABILITY_TEST_DATABASE_URL` (`describePostgres`).
4. **Frontend FO suite** was passed in Phase 1 and not re-run here (memory budget); the Phase 2-specific frontend test re-run was the single pure-logic `reservation-actions.test.ts`.
5. **Realtime event→invalidation matrix** not exhaustively enumerated.
6. `apps/mobile` was not scanned (carried from Phase 1).
7. Findings describe the **working tree** (`HEAD = ed278bc`, heavily modified), not a clean commit.

---

## 14. Findings Index

See `docs/front-office/PHASE_2_INTEGRATION_GAP_REGISTER.md` for the 13 findings (`FO-P2-001` … `FO-P2-013`) with evidence, severity, classification, dependencies and acceptance criteria. Flow diagrams are in `docs/front-office/PHASE_2_INTEGRATION_MAP.md`. Commands, results and the 10-criterion exit gate are in `docs/front-office/PHASE_2_BASELINE_AND_EXIT_GATE.md`.

---

*End of Phase 2 integration audit.*

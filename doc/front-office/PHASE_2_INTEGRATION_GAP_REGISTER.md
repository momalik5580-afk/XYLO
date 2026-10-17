# XYLO Front Office — Phase 2 Integration Gap Register

- **Date:** 2026-10-17
- **Phase:** Phase 2 — Reservations ↔ Front Office Integration Audit (audit-only)
- **Companion documents:** `docs/front-office/PHASE_2_RESERVATIONS_INTEGRATION_AUDIT.md`, `docs/front-office/PHASE_2_INTEGRATION_MAP.md`, `docs/front-office/PHASE_2_BASELINE_AND_EXIT_GATE.md`
- **ID prefix:** `FO-P2-###`
- **Classification legend:** Confirmed defect · Potential risk · Observation
- **Severity legend:** P0 Critical · P1 High · P2 Medium · P3 Low · Info
- **Evidence legend:** direct · test-derived · documentation-derived · circumstantial
- **Confidence:** high / medium / low
- **Follow-up phase** is a recommendation only; no phase after Phase 2 is started here.

Findings are de-duplicated against the Phase 1 register (`FO-P1-*`); where a Phase 2 finding extends a Phase 1 finding, the cross-reference is in the body and in §Summary.

---

## FO-P2-001 — Check-in guest/shares panel requires a reservation UUID but is passed the confirmation number

- **Category:** INTEGRATION / FE→BE contract / data resolution
- **Type:** Integration defect (identifier misuse)
- **Description:** `CheckInDialog` uses the reservation UUID for every staged check-in call but passes `reservation.confirmationNo` to the accompanying-guests engine, whose backend query joins `reservation_guests.reservation_id` (a UUID).
- **Evidence (direct):**
  - `apps/web/features/front-office/components/dialogs/CheckInDialog.tsx:556` — `<CheckInGuestList reservationId={reservation.confirmationNo || ''} />`
  - `apps/web/features/front-office/components/check-in/CheckInGuestList.tsx:73` — `useReservationParties(reservationId)`
  - `apps/web/features/reservations/hooks/use-reservation-queries.ts:217-223` — `GET /reservations/${reservationId}/parties`
  - `apps/api/src/modules/reservations/api/controllers/reservations.controller.ts:281-286` — `getParties(@Param('id') id)` → `reservationsService.getParties(id)`
  - `apps/api/src/modules/reservations/infrastructure/repositories/reservation.repository.ts:1458-1463` — `SELECT * FROM reservation_guests WHERE reservation_id = $1`
  - `reservation.repository.ts:1468` — parties are written with `reservation_id = data.reservationId` (UUID)
  - `apps/api/src/modules/front-office/check-in/application/services/check-in-workflow.service.ts:122` — `confirmationNo: row.confirmation_no ?? null` (a distinct nullable column)
  - `apps/api/src/modules/reservations/infrastructure/repositories/reservation.repository.ts:486` — reservation `id` is `crypto.randomUUID()`
- **Contrast (direct):** the flow calls in the same dialog correctly use the UUID: `CheckInDialog.tsx:205,252,266,285,298,312` use `rid` (the URL-provided `reservationId`).
- **Impact:** the "Accompanying Guests & Shares" section of check-in silently renders nothing for every reservation whose `confirmation_no` differs from the UUID (the normal case); front-desk staff cannot see/verify accompanying guests or shares during check-in. Silent — no error is surfaced.
- **Severity / confidence:** **P2** / high.
- **Classification:** Confirmed defect.
- **Recommended follow-up phase:** Phase 9 (frontend reconstruction) with a backend contract note; verify with a component/route test.
- **Dependencies:** decision on whether `/reservations/:id/parties` should accept a confirmation number as an alternate key; otherwise pass `rid` (UUID).
- **Acceptance criteria:** the guest list resolves for a normal reservation; a test asserts the request uses the reservation UUID (or the endpoint accepts both keys).

---

## FO-P2-002 — Front Office "Mark No-Show" is rejected by validation and bypasses the no-show owner command

- **Category:** INTEGRATION / API contract / lifecycle
- **Type:** Integration defect (broken call + wrong owner)
- **Description:** FO no-show posts `{ reservation_status: 'NO-SHOW' }` to the generic `PUT /reservations/:id`, whose DTO does not declare that field; the global pipe rejects unknown fields. The dedicated `POST /reservations/:id/no-show` command is never used.
- **Evidence (direct):**
  - `apps/web/features/front-office/components/dashboard/ArrivalPanel.tsx:94` — `markNoShow.mutate(a.id)`
  - `apps/web/features/front-office/hooks/use-front-office-mutations.ts:136-143` → `front-office.api.ts:208-210` — `api.put('/reservations/' + reservationId, { reservation_status: 'NO-SHOW' })`
  - `apps/api/src/modules/reservations/api/controllers/reservations.controller.ts:100-106` — `@Put(':id')` binds `UpdateReservationDto`
  - `apps/api/src/modules/reservations/api/dto/update-reservation.dto.ts:3-18` — no `reservation_status`/`status` property
  - `apps/api/src/main.ts:28-40` — `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true })`
  - Owner route exists but is unused: `reservations.controller.ts:123-126` → `reservations.service.ts:110-116` (`ProcessNoShowCommand`)
- **Impact:** marking a no-show from the arrivals panel fails (HTTP 400 for a permitted caller; 403 for an unpermitted caller) and never triggers the no-show business logic (status change, availability release, history). Operators may believe the action succeeded if the UI does not surface the error.
- **Severity / confidence:** **P2** / high (direct + config-derived; runtime not executed).
- **Classification:** Confirmed defect.
- **Recommended follow-up phase:** Phase 9 (frontend wiring) — point `markNoShow` at `POST /reservations/:id/no-show`.
- **Dependencies:** none; the owner route already exists.
- **Acceptance criteria:** FO no-show calls the owner command and persists `NO-SHOW`/`NO_SHOW` canonically; a test asserts the endpoint used.

---

## FO-P2-003 — Check-out performs financial side effects before the transition guard with no idempotency protection

- **Category:** INTEGRATION / billing / idempotency
- **Type:** Integration defect (ordering + missing idempotency)
- **Description:** refund, penalty, minibar, payment and voucher redemption run before the illegal-transition guard and outside the status-transition transaction; the client sends no `Idempotency-Key`, and the interceptor no-ops without one.
- **Evidence (direct):**
  - `apps/api/src/modules/front-office/application/commands/check-out/check-out.handler.ts:117` (refund), `:144` (penalty), `:166` (minibar), `:189` (payment), `:196-226` (voucher credit + redemption) — all before `:235` `statusService.assertTransition(...)`
  - `check-out.handler.ts:388` — `reservationRepository.applyCheckOut(...)` inside the transaction (after `:235`)
  - `apps/web/components/CheckoutFlowModal.tsx:130` — `api.post('/front-office/check-out', body)` (no idempotency key)
  - `apps/web/services/api.ts:22-51,109-115` — the header is only sent when a caller mints one
  - `apps/api/src/core/interceptors/idempotency.interceptor.ts:21-24` — `if (!idempotencyKey) return next.handle();`
  - `packages/shared/src/reservation-state-machine.ts:39` — `CHECKED_OUT → ['CHECKED_IN']` only (second check-out is illegal)
- **Impact:** a duplicate/retried check-out re-posts refunds/penalties/minibar/payments/voucher redemptions before failing on the status transition — potential financial duplication. The voucher path also decrements the voucher balance and writes a redemption row before the guard.
- **Severity / confidence:** **P2** / high (direct; runtime not executed).
- **Classification:** Confirmed defect.
- **Recommended follow-up phase:** Phase 12 (production readiness / financial integrity) with a frontend idempotency-key mint in Phase 9.
- **Dependencies:** decision on whether the HTTP idempotency key or an in-handler idempotency guard is the canonical mechanism (the master-folio auto-post already has an internal idempotency token — `check-out.handler.ts:288-290`).
- **Acceptance criteria:** a repeated check-out request is a no-op (no additional postings); a test asserts single-application semantics.

---

## FO-P2-004 — Upgrade room-selection contract mismatch (`toRoomNo` vs `newRoomId`)

- **Category:** INTEGRATION / API contract
- **Type:** Integration defect (dead field / latent identifier bug)
- **Description:** the DTO and frontend expose `toRoomNo`, but the service reads `newRoomId`, which is never sent, so the room-pinning branch is unreachable; the underlying command would use a room number where a room UUID is expected.
- **Evidence (direct):**
  - `apps/web/features/front-office/api/front-office.api.ts:135-137` — posts `{ ...body }` (incl. `toRoomNo`) to `/front-office/:id/upgrade`
  - `apps/web/features/front-office/mutations/mutation.types.ts:45-48` and `validation/room-ops.schemas.ts:9-14` — `toRoomNo?`
  - `apps/api/src/modules/front-office/dto.ts:98-102` — `UpgradeRoomDto { newRoomType, toRoomNo?, reason? }`
  - `apps/api/src/modules/front-office/front-office.controller.ts:118-122` — spreads body into `service.upgradeRoom`
  - `apps/api/src/modules/front-office/front-office.service.ts:112-113` — reads `data.newRoomId` (undefined)
  - `apps/api/src/modules/front-office/application/commands/upgrade-room/upgrade-room.handler.ts:119-132` region — would use `command.newRoomId` against `rooms.id`
  - `apps/web/features/front-office/utils/upgrade.helpers.ts:7-12` — documents the operation as **category-based**
- **Impact:** no functional break today (category-based upgrade is the documented behavior), but the contract is inconsistent and the `newRoomId` branch is a latent room-number-as-UUID bug.
- **Severity / confidence:** **P3** / high (would escalate to P2 if room-pinning is intended). Direct.
- **Classification:** Confirmed defect.
- **Recommended follow-up phase:** Phase 9 (align the frontend/back-end contract; remove or wire `toRoomNo`).
- **Dependencies:** product decision — is upgrade category-only or room-specific?
- **Acceptance criteria:** either `toRoomNo` is removed from the DTO/UI, or it is mapped to a validated room UUID end-to-end.

---

## FO-P2-005 — Common `@Permissions` guard cannot authorize non-admins (FO routes effectively admin-only)

- **Category:** INTEGRATION / authorization
- **Type:** Integration defect (permission source never populated)
- **Description:** the common permission guard authorizes only `SUPER_ADMIN`/`ADMIN` in practice, because `request.user.permissions` is never hydrated; the 9 FO routes using the common `@Permissions` decorator are therefore admin-only.
- **Evidence (direct):**
  - `apps/api/src/common/authorization/permission.guard.ts:14-24` — reads `request.user?.permissions ?? []`; only `SUPER_ADMIN`/`ADMIN` pass otherwise
  - `apps/api/src/core/auth/jwt.strategy.ts:28-30` — returns `authService.validateUser(payload.sub)`
  - `apps/api/src/core/auth/auth.service.ts:129-140` — `select` omits `permissions`; no `permissions` field returned
  - grep over `apps/api/src` — no assignment to `request.user.permissions`
  - `apps/api/src/common/authorization/authorization.module.ts:18-22` — the common `PermissionGuard` is a registered `APP_GUARD`
  - FO routes using the common decorator: `apps/api/src/modules/front-office/front-office.controller.ts:42,49,59,80,319,327,337,354,363` (`front_office:check_in:read/write`, `front_office:room_moves:read`)
- **Contrast (direct + test-derived):** routes using the **platform** `@Permission` decorator enforce via `PermissionResolverService` and property scope and work for non-admins (`platform/permission/permission.guard.ts:40-54`; `fo-route-permissions.spec.ts` — PASS, §Baseline).
- **Impact:** front-desk roles without `ADMIN` cannot use walk-in defaults, discrepancies, resolve-discrepancy, room detail, the room-moves dashboard, or the mobile back-to-back route — while the frontend (which gates on different codes, `FO-P2-009`) may still show them. Broken authorization UX + admin-only backend on those routes.
- **Severity / confidence:** **P2** / high (direct; runtime not executed).
- **Classification:** Confirmed defect.
- **Recommended follow-up phase:** Phase 12 (property-scope/RBAC regression suite) with a fast fix; or a dedicated hardening task.
- **Dependencies:** choose the canonical FO permission source (populate `request.user.permissions` from grants, or migrate these routes to the platform `@Permission` + resolver).
- **Acceptance criteria:** a non-admin with the granted `front_office:*` permission can invoke the route; a test covers it. Cross-ref `FO-P1-001`.

---

## FO-P2-006 — Two folio families with colliding keys are merged only client-side at checkout

- **Category:** INTEGRATION / billing / data ownership
- **Type:** Potential risk (dual source of truth)
- **Description:** the legacy `folio_header` family (Int header key; `folio_payments` keyed by reservation UUID) and the new `folios`/`folio_postings` ledger (no Prisma model) are written by different code and only reconciled in the checkout UI.
- **Evidence (direct):**
  - `packages/db/schema.prisma:4354-4355` — `folio_header.folio_id Int @id @default(autoincrement())`
  - `packages/db/schema.prisma:4390-4393` — `folio_payments.folio_id String`
  - `packages/db/schema.prisma:4414-4425` — `folio_postings` exists; comment states "no Prisma `folios` model exists"
  - `apps/api/src/modules/billing/billing.service.ts:53-69` — charges read by Int header id; payments read by reservation UUID
  - `apps/api/src/modules/billing/billing.service.ts:151-156` — `postPayment` writes `folio_payments.folio_id = data.reservationId` (UUID)
  - `apps/api/src/modules/front-office/application/commands/check-out/check-out.handler.ts:527` — `billing.closeFolio(...)` (legacy family)
  - `apps/web/components/CheckoutFlowModal.tsx:17,324` — merges legacy `billingStore` and cashiering `useReservationFolios` client-side
- **Impact:** no single server-authoritative balance; risk of divergent balances between the checkout UI, the legacy folio, and the posting ledger; a future consumer that reads only one family sees a partial picture.
- **Severity / confidence:** **P2** / high for the split; medium for the reconciliation behavior (not runtime-verified).
- **Classification:** Potential risk.
- **Recommended follow-up phase:** Phase 6/7 (inventory/profile/billing integration) or a dedicated cashiering/billing convergence task.
- **Dependencies:** billing/cashiering convergence decision; folio-key canonicalization.
- **Acceptance criteria:** one authoritative folio source per reservation exposed by a single API, with the other family derived or retired.

---

## FO-P2-007 — Cross-reservation overlap safeguard in room assignment is unreachable

- **Category:** INTEGRATION / availability / room assignment
- **Type:** Potential risk (dead safety input)
- **Description:** `RoomAssignableSpecification` supports an `assignedToOtherReservation` flag, but the assign-room handler never hydrates it, so the overlap branch never fires.
- **Evidence (direct):**
  - `apps/api/src/modules/front-office/check-in/domain/specifications/room-assignable.specification.ts:16-17` — `assignedToOtherReservation?: boolean`
  - `apps/api/src/modules/front-office/check-in/application/commands/assign-room.handler.ts:84-88` and `:99-103` — `evaluate({ room, bookedRoomType, upgradeApproval })` only
  - `assign-room.handler.ts:95,110-112` — safety relies on the per-room lock and the room status transition
  - `apps/api/src/modules/front-office/check-in/__tests__/check-in-availability.postgres.spec.ts:41-49` — check-in intentionally consumes no availability (F-14)
- **Impact:** the only cross-reservation double-assignment guard in the assignment spec is inert; protection depends solely on the room lock and room status. If a room were force-marked assignable while another reservation holds it, the spec would not object.
- **Severity / confidence:** **P2** / high (direct; the Postgres safety test is skipped without env).
- **Classification:** Potential risk.
- **Recommended follow-up phase:** Phase 6 (inventory integration) — hydrate overlap from the assignments/holds model.
- **Dependencies:** the assignments/overlap source of truth (reservation assignment + holds).
- **Acceptance criteria:** assign-room rejects a room overlapping another active reservation, with a test.

---

## FO-P2-008 — Check-out does not gate on full settlement

- **Category:** INTEGRATION / billing / UX
- **Type:** Confirmed defect / policy gap
- **Description:** check-out is enabled with only a payment method selected; the backend posts the given amount and closes the folio without asserting a zero balance, swallowing close errors.
- **Evidence (direct):**
  - `apps/web/components/CheckoutFlowModal.tsx:1152` — `disabled={processing || (!isGroupReservation && !paymentMethod)}`
  - `apps/api/src/modules/front-office/application/commands/check-out/check-out.handler.ts:187-193` — posts `paymentAmount` as provided
  - `check-out.handler.ts:527-530` — `closeFolio` then swallow error
- **Impact:** a residual balance can remain on a `CHECKED_OUT` folio while the UI says "Check Out & Settle"; there is no server-side settlement assertion. (May be intentional if AR/city-ledger transfer exists elsewhere — not found in the FO path.)
- **Severity / confidence:** **P3** / high (direct; policy-dependent).
- **Classification:** Confirmed defect.
- **Recommended follow-up phase:** Phase 12 (production readiness) after confirming the AR policy.
- **Dependencies:** city-ledger/AR policy decision.
- **Acceptance criteria:** either check-out blocks on a non-zero balance, or transfers the residual to AR explicitly and reports it; a test covers the chosen behavior.

---

## FO-P2-009 — Frontend/back-end permission vocabulary mismatch

- **Category:** INTEGRATION / authorization / consistency
- **Type:** Confirmed defect
- **Description:** the Front Office UI gates on legacy codes (`write:reservations`, `read:billing`, …) while the API uses `reservations:reservation:*` and `front_office:*`.
- **Evidence (direct):**
  - `apps/web/features/front-office/permissions/front-office.permissions.ts:3-23`
  - `apps/web/lib/permissions/usePermissions.ts:38-48` — `WRITE_RESERVATIONS = 'write:reservations'`, etc.
  - Back-end codes: `front-office.controller.ts:42,319` (`front_office:check_in:read`, `front_office:room_moves:read`); `RESERVATION_PERMISSIONS.*` on `:101,107,113,119,125`
  - `apps/web/lib/permissions/usePermissions.ts:61-68` — the reservation-engine codes already exist in the same object
- **Impact:** UI visibility and back-end enforcement are governed by different vocabularies, so the UI can show an action the API denies (or vice versa). Consolidation is required before relying on either.
- **Severity / confidence:** **P3** / high (direct).
- **Classification:** Confirmed defect.
- **Recommended follow-up phase:** Phase 9 (frontend) + Phase 12 regression. Cross-ref `FO-P1-001`, `FO-P2-005`.
- **Dependencies:** the canonical permission catalogue.
- **Acceptance criteria:** FO UI and API reference the same permission codes; a test enumerates them.

---

## FO-P2-010 — Reinstate is reachable via two routes (converging)

- **Category:** INTEGRATION / ownership
- **Type:** Observation
- **Description:** the reservations pipeline can reinstate with or without a room, choosing between `POST /reservations/:id/reinstate` and `POST /front-office/reinstate`.
- **Evidence (direct):**
  - `apps/web/features/reservations/api/reservation.api.ts:367-376` — if no room, `POST /reservations/:id/reinstate`; else `POST /front-office/reinstate`
  - `apps/api/src/modules/reservations/api/controllers/reservations.controller.ts:195-199` — reservations reinstall route
  - `apps/api/src/modules/front-office/front-office.controller.ts:100-104` → `front-office.service.ts:118-136` — both converge on `ReinstateReservationCommand`, FO additionally calls `assignRoomNumber`
- **Impact:** none functionally (both converge on the owner command); recorded to prevent a future "duplicate writer" misread.
- **Severity / confidence:** **Info** / high.
- **Classification:** Observation.
- **Recommended follow-up phase:** n/a (informational).

---

## FO-P2-011 — Deep-link `reservation=` is ignored by three dialog kinds

- **Category:** INTEGRATION / navigation
- **Type:** Observation
- **Description:** `reverse-check-in`, `reinstate` and `redeem-voucher` derive the reservation from the selected room, not from the URL `reservation` param.
- **Evidence (direct):**
  - `apps/web/features/front-office/components/dashboard/OperationDialogHost.tsx:64-66` — rendered with `room={room}` only
  - `ReverseCheckInDialog.tsx:364,491` and `ReinstateDialog.tsx:365,383` — reservation derived from `room?.reservationId` / snapshot
- **Impact:** a deep link to these dialogs without a selected room lands in a "pick a room" phase; not a data defect.
- **Severity / confidence:** **Info** / high.
- **Classification:** Observation.
- **Recommended follow-up phase:** Phase 9 (optional UX consistency).

---

## FO-P2-012 — Walk-in creates a reservation but never enters the staged check-in / room-assignment path

- **Category:** INTEGRATION / lifecycle
- **Type:** Observation (integration implication of `FO-P1-004`)
- **Description:** FO `walkIn` creates a `CONFIRMED` reservation (`WalkInCommand`) and stops; it does not enter the staged check-in flow or assign a room, so the guest is not in-house.
- **Evidence (direct):**
  - `apps/api/src/modules/front-office/front-office.service.ts:138-140` — `WalkInCommand`
  - Backend create path: `application/commands/sub-resources/sub-resources.handlers.ts` (walk-in handler) → `ReservationsService.create` with status `CONFIRMED` (Phase 1 `FO-P1-004` evidence)
  - Check-in stages are only reachable via `/check-in/*` (`check-in.controller.ts`)
- **Impact:** a walk-in guest is created but not occupied; the rack shows them as not-in-house until a separate check-in. Extends `FO-P1-004`.
- **Severity / confidence:** **Info** / high (direct; partial, cross-referenced to Phase 1).
- **Classification:** Observation.
- **Recommended follow-up phase:** Phase 9 (define/implement walk-in → check-in).

---

## FO-P2-013 — `reg-card` and `checkout-history` dialog kinds remain unrendered

- **Category:** INTEGRATION / dead wiring
- **Type:** Confirmed defect (carry-forward `FO-P1-006`)
- **Description:** re-confirmed that `OperationDialogHost` maps permissions for `reg-card`/`checkout-history` but has no render branch for either.
- **Evidence (direct):**
  - `apps/web/features/front-office/components/dashboard/OperationDialogHost.tsx:35-36` (permission map) vs `:57-67` (render branches; neither appears)
- **Impact:** opening those dialog kinds renders nothing (`Suspense` fallback `null`).
- **Severity / confidence:** **P3** / high (direct).
- **Classification:** Confirmed defect.
- **Recommended follow-up phase:** Phase 9 (wire or remove). Cross-ref `FO-P1-006`.

---

## Summary

**Totals: 13 findings — P0 = 0, P1 = 0, P2 = 6, P3 = 4, Info = 3.**

| Severity | Count | IDs |
|---|---|---|
| P0 | 0 | — |
| P1 | 0 | — |
| P2 | 6 | 001, 002, 003, 005, 006, 007 |
| P3 | 4 | 004, 008, 009, 013 |
| Info | 3 | 010, 011, 012 |

| Classification | IDs |
|---|---|
| Confirmed defect | 001, 002, 003, 004, 005, 008, 009, 013 |
| Potential risk | 006, 007 |
| Observation | 010, 011, 012 |

**Cross-references to Phase 1:** `FO-P2-005` + `FO-P2-009` ↔ `FO-P1-001`; `FO-P2-012` ↔ `FO-P1-004`; `FO-P2-013` ↔ `FO-P1-006`; `FO-P1-002` re-confirmed (not duplicated).

---

*End of Phase 2 integration gap register.*

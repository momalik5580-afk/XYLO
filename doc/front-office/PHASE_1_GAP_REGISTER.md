# XYLO Front Office — Phase 1 Gap Register

- **Date:** 2026-10-16
- **Companion documents:** `docs/front-office/PHASE_1_CURRENT_STATE_AUDIT.md`, `docs/front-office/PHASE_1_BASELINE_AND_EXIT_GATE.md`
- **Classification legend:** Confirmed defect · Partial implementation · Potential risk · Improvement opportunity · Unverified
- **Severity legend:** P0 Critical · P1 High · P2 Medium · P3 Low · Info
- **Evidence legend:** direct · test-derived · documentation-derived · circumstantial
- **Confidence:** high / medium / low
- **Follow-up phase** is a recommendation only; no phase after Phase 1 is started here.

Findings are de-duplicated across the audit. Backend/security findings have `SEC`/`BE` categories; frontend findings `FE`; UX `UX`; testing `QA`; docs `DOC`; data `DATA`.

---

## FO-P1-001 — FO controller endpoints lack authorization decorators

- **Category:** SEC (authorization)
- **Description:** 40 of 54 routes on `FrontOfficeController` declare no `@Permissions`/`@Permission` decorator. Covered routes are only the 4 `walk-in/defaults`/`discrepancies`/`rooms/:roomNumber`/room-moves/mobile routes and the 5 reservation `@Permission` mutations. Undecorated routes include state-changing operations.
- **Evidence (direct):** grep over `apps/api/src/modules/front-office/front-office.controller.ts` returns 14 decorator usages for 54 routes (decorator lines 42, 49, 59, 80, 101, 107, 113, 119, 125, 319, 327, 337, 354, 363). Undecorated mutating routes include: `POST walk-in` (:85), `POST check-out` (:90), `POST reverse-checkin` (:95), `POST traces` (:162), `POST traces/:id/complete` (:167), `POST messages` (:178), `PUT messages/:id/read` (:183), `POST wake-up-calls` (:194), `POST wake-up-calls/:id/complete` (:199), `POST service-requests` (:210), `PUT service-requests/:id` (:215), `POST room-queue` (:226), `POST room-queue/:id/ready` (:231), `DELETE room-queue/:id` (:236), `POST day-use` (:247), `POST day-use/:id/close` (:252), `POST vouchers` (:263), `POST vouchers/:id/redeem` (:268), `POST vouchers/apply` (:273), `PATCH vouchers/:id/link` (:278), `POST telephone-messages` (:289), `PUT telephone-messages/:id/read` (:294), `POST reg-card/upload` (:300).
- **Enforcement context (direct, derived):** the common guard `apps/api/src/common/authorization/permission.guard.ts` checks `request.user.permissions` only when the `@Permissions` metadata is present; `RolesGuard` similarly no-ops without `@Roles`. Global `JwtAuthGuard` still requires authentication (`apps/api/src/app.module.ts:107`). Therefore the gap is **missing RBAC on authenticated users**, not unauthenticated access.
- **Reproduction / verification:** `rg "@Permissions\(|@Permission\(" apps/api/src/modules/front-office/front-office.controller.ts`; compare against the route list. (Not executed at runtime.)
- **Impact:** any authenticated user (regardless of role) can perform high-impact FO mutations: create/check-out reservations, issue/redeem vouchers, post day-use, upload reg-cards, and manipulate guest-service records. Breaks the project's stated RBAC principle (`docs/enterprise/phase3-reservations-front-desk-spec.md:20`).
- **Severity / confidence:** **P1** / high (code-direct; runtime not executed).
- **Classification:** Confirmed defect.
- **Recommended follow-up phase:** Phase 2 (integration) / dedicated hardening task before operational validation.
- **Dependencies:** permission constants for FO (e.g. `front_office:*`); agreement on the FO permission map.
- **Acceptance criteria:** every mutating route carries an explicit permission decorator; a route-permission test enumerates all routes; 403 returned for users lacking the permission.

---

## FO-P1-002 — Class-level `@PropertyScope(false)` disables property-scope enforcement

- **Category:** SEC / multi-tenancy
- **Description:** both FO controllers declare `@PropertyScope(false)` at class level, so `PropertyScopeGuard`, `PropertyAccessGuard`, `DepartmentAccessGuard`, `WarehouseAccessGuard`, and `MultiTenantGuard` short-circuit to allow. Property-scope "refused" evaluations are not rejected on these routes.
- **Evidence (direct):** `front-office.controller.ts:24`; `check-in/api/check-in.controller.ts:50` (grep confirms no method-level overrides). Guard short-circuits: `core/guards/property-scope.guard.ts:21-25`; `common/authorization/resource-access.guard.ts:21-25`; `core/guards/multi-tenant.guard.ts:17-21`. The platform permission guard still resolves scope for the 5 `@Permission` routes (`platform/permission/permission.guard.ts:40-44`).
- **Mitigation observed (direct):** ALS context is still populated by `TenantInterceptor` (`core/interceptors/tenant.interceptor.ts`), and `property-scope.resolver.ts:49-55` falls back to the principal's own identity, so handlers generally run scoped to the caller's own property via `requireHotelId()`. Client-supplied `x-property-id` does not redirect the tenant.
- **Documentation-derived:** the same pattern is flagged in `docs/RESERVATIONS_FORENSIC_AUDIT.md` (front-office endpoints "unscoped"; all FO handlers said to violate the `findFirst` constraint).
- **Impact:** weaker multi-tenant isolation guarantees than the rest of the platform; a compromised/odd scope evaluation is not refused on FO routes; inconsistent with the platform's `APP_GUARD` property-scope model.
- **Severity / confidence:** **P2** / high (code-direct). Downgraded from P1 because JWT auth still applies and ALS still resolves the caller's own property.
- **Classification:** Confirmed defect (pre-existing design).
- **Recommended follow-up phase:** Phase 2 (confirm intent with owner) then hardening task near Phase 12 property-scope regression suite.
- **Dependencies:** property-scope resolver behavior; multi-property authorization model.
- **Acceptance criteria:** either documented as an intentional exception with rationale, or property scoping enforced/refused consistently with other modules.

---

## FO-P1-003 — Person-Discrepancy reconcile actions are client-only no-ops

- **Category:** FE / functional
- **Description:** `PersonDiscrepancyTile` derives discrepancies from `useRoomGrid` and "resolves" them only in local React state; no network mutation is issued. The backend already exposes `GET /front-office/discrepancies` and `POST /front-office/discrepancies/:id/resolve`, and a `ResolveDiscrepancyCommand` handler exists — but the frontend does not call them.
- **Evidence (direct):** `apps/web/features/front-office/components/dashboard/PersonDiscrepancyTile.tsx:11` (local `resolvedIds` set), `:24-27` (`handleResolve` toast only), `:29-33` (`handleReconcileAll` toast only). Backend: `front-office.controller.ts:48-72`; `application/commands/resolve-discrepancy.handler.ts`; `infrastructure/repositories/discrepancy.repository.ts`.
- **Reproduction:** open a property with a FO/HK mismatch, click "Reconcile All"/"Verify Occupied" — toast shows success, but `refetch()`/reload restores the discrepancy.
- **Impact:** operators believe discrepancies are reconciled when nothing is persisted; audit trail absent; failed silent workflow.
- **Severity / confidence:** **P2** / high.
- **Classification:** Confirmed defect (partial implementation).
- **Recommended follow-up phase:** Phase 2 (wire FE to existing endpoints) — note: `GET /front-office/discrepancies` returns the repository's own discrepancy records, which may differ from the tile's client-derived SKIP/SLEEP heuristic; reconcile the two models.
- **Dependencies:** FO discrepancy repository + HK person-count commands (`housekeeping/application/commands/*`).
- **Acceptance criteria:** resolve actions call the backend, persist, and reflect server truth after refetch; a test covers the mutation.

---

## FO-P1-004 — Walk-in does not check in the guest; success toast is inaccurate

- **Category:** FE+BE / functional
- **Description:** `WalkInDialog` submits only `useWalkInMutation` and then shows `"Walk-in created and checked in"`. The backend `WalkInCommandHandler` creates a reservation with `reservation_status: 'CONFIRMED'` and issues **no** check-in. The CRS facade comment describing optional auto check-in is not implemented.
- **Evidence (direct):** `WalkInDialog.tsx:60-67`; `application/commands/sub-resources/sub-resources.handlers.ts:471-489` (creates via `ReservationsService.create`, status `CONFIRMED`, no `check-in` call). Rebuild doc anticipated walk-in → check-in linkage (`docs/enterprise/front-office-frontend-rebuild-architecture-2026-08-02.md:180-191`).
- **Reproduction:** create a walk-in; observe the reservation is `CONFIRMED`, not `CHECKED_IN`; no room is occupied.
- **Impact:** walk-in guests are not actually checked in; misleading feedback; requires a second manual check-in step the UI implies already happened.
- **Severity / confidence:** **P2** / high.
- **Classification:** Confirmed defect (partial implementation + incorrect feedback).
- **Recommended follow-up phase:** Phase 2 (define intended walk-in→check-in flow; wire `POST /check-in/*` or correct the message).
- **Dependencies:** check-in staged workflow; idempotency.
- **Acceptance criteria:** documented behavior (create-only vs create+check-in); toast matches reality; test covers the flow.

---

## FO-P1-005 — Mutating routes bypass DTO validation (`@Body() any` / inline bodies)

- **Category:** BE / data-integrity
- **Description:** several mutating routes accept untyped `any` or inline object-literal bodies, so the global `ValidationPipe` (`main.ts:28-40`) has no metatype to validate.
- **Evidence (direct):** `front-office.controller.ts:86` (`walkIn`, `any`), `:264` (`addVoucher`, `any`), `:269` (`redeemVoucher`, `any`); inline object bodies at `:62` (`resolveDiscrepancy`), `:102` (`reinstateReservation`), `:274` (`applyVoucher`), `:279` (`linkVoucher`), `:301` (`uploadRegCard`); `check-in.controller.ts:100` (`checkInGuest`, `{roomKeyCount?:number}`). Flows onward as `any` (`front-office.service.ts:138`; `sub-resources.handlers.ts`).
- **Impact:** missing `whitelist`/`forbidNonWhitelisted` protection; malformed input reaches handlers; harder to reason about contracts; risk of mass-assignment style issues.
- **Severity / confidence:** **P2** / high (code-direct). Runtime rejection behavior **not executed**.
- **Classification:** Confirmed defect (validation gap).
- **Recommended follow-up phase:** Phase 2/hardening (introduce DTOs for these routes).
- **Dependencies:** `dto.ts` exists already (`CheckOutDto`, etc.).
- **Acceptance criteria:** all mutating routes use class DTOs with decorators; tests assert 400 on invalid payloads.

---

## FO-P1-006 — `reg-card` and `checkout-history` dialogs registered but not rendered

- **Category:** FE / dead wiring
- **Description:** `OperationDialogHost` grants permissions for `'reg-card'` and `'checkout-history'` and accepts them as dialog kinds, but the render block has no branch for either; opening them renders nothing (Suspense fallback `null`). No caller opens them, and the `reg-card` read hooks/API are unused.
- **Evidence (direct):** `components/dashboard/OperationDialogHost.tsx:35-36` (permission map) vs `:57-67` (render branches; no `reg-card`/`checkout-history`). Dialog kinds declared in `store/front-office-ui-store.ts:24-25`; URL params accepted in `hooks/use-front-office-url-sync.ts:21-22`. Unused read paths: `hooks/use-front-office-queries.ts:156` (`useRegCardUrl`), `api/front-office.api.ts:110` (`getRegCardUrl`). Feature-local `components/guest-services/CheckoutHistoryModal.tsx` is exported (`guest-services/index.ts:10`) but not rendered.
- **Impact:** registration-card and dialog-based checkout-history flows are unreachable; dead code paths; potential for a future "silent nothing happened" bug.
- **Severity / confidence:** **P2** / high.
- **Classification:** Partial implementation.
- **Recommended follow-up phase:** Phase 2 (wire or remove).
- **Dependencies:** `RegCardService` + `GetRegCardUrlQuery` (backend ready).
- **Acceptance criteria:** either a rendered, tested dialog for each kind, or the kinds are removed from the store/permission maps.

---

## FO-P1-007 — `dayuse` / `history` guest-service panels have no host branch

- **Category:** FE / dead wiring
- **Description:** `GuestServicesPanel` includes `'dayuse'` and `'history'`, but `GuestServicesHost` renders only messages/traces/wakeup/service/queue/telephone/early-checkout. `openServicePanel('dayuse'|'history')` sets state with nothing rendered. No caller opens them.
- **Evidence (direct):** `store/front-office-ui-store.ts:29-39`; `components/guest-services/GuestServicesHost.tsx:19-25` (no branch).
- **Impact:** latent silent no-op states.
- **Severity / confidence:** **P3** / high.
- **Classification:** Confirmed defect (dead wiring).
- **Follow-up phase:** Phase 2.
- **Acceptance criteria:** remove the unused panel kinds or render them.

---

## FO-P1-008 — "Transportation" quick-action permanently disabled

- **Category:** UX / improvement
- **Description:** a static, permanently `aria-disabled` tile with tooltip "Transportation is not available in this system yet".
- **Evidence (direct):** `components/guest-services/QuickActionGrid.tsx:88-100`.
- **Impact:** minor visual noise; honest but incomplete surface.
- **Severity / confidence:** **P3** / high.
- **Classification:** Improvement opportunity.
- **Follow-up phase:** Phase 2/9 (UX).

---

## FO-P1-009 — In-house Call / Email actions are malformed

- **Category:** UX / functional
- **Description:** Call builds `tel:<guestName>` (not a phone number); Email builds `mailto:?subject=…` with no recipient.
- **Evidence (direct):** `components/guest-services/InHouseGuestsPanel.tsx:115` (Call), `:118` (Email).
- **Impact:** actions cannot work as labelled.
- **Severity / confidence:** **P3** / high.
- **Classification:** Confirmed defect.
- **Follow-up phase:** Phase 2/9.
- **Acceptance criteria:** use a real phone/email from guest data, or disable when absent.

---

## FO-P1-010 — Day-use drawer "Closed at" hard-coded

- **Category:** UX / functional
- **Description:** the drawer always renders `—` for "Closed at", never deriving the close timestamp.
- **Evidence (direct):** `components/day-use/DayUseDrawer.tsx:160`.
- **Impact:** closed day-use accounts show no close time.
- **Severity / confidence:** **P3** / high.
- **Classification:** Confirmed defect.
- **Follow-up phase:** Phase 2/9.

---

## FO-P1-011 — Legacy orphan FO components and dead `frontOfficeStore.ts`

- **Category:** FE / cleanup / risk
- **Description:** multiple pre-rebuild FO components remain but are unreferenced; `store/frontOfficeStore.ts` is imported only by two of those orphans, making it effectively dead. Duplicate/competing implementations of the same concepts remain.
- **Evidence (direct):** grep over `apps/**/*.{ts,tsx}` — `VouchersModal`, `TelephoneOperatorPanel`, `CheckoutHistoryModal` (top-level), `TransferModal`, `FolioPanel`, `RecordPaymentModal`, `KeyPacketLabelsModal`, `StayTypeSettingsModal`, `CreateActivityModal` appear only in their own definitions (the only external-looking matches are an unrelated `showRecordPaymentModal` state variable in `features/group-allotment/views/GroupBookingDetailView.tsx`). `store/frontOfficeStore.ts` importers: `components/VouchersModal.tsx:6`, `components/TelephoneOperatorPanel.tsx:6` (both orphans).
- **Impact:** dead code and duplicated endpoint logic; risk of future edits targeting the wrong component; larger maintenance surface.
- **Severity / confidence:** **P3** / high.
- **Classification:** Potential risk.
- **Follow-up phase:** Phase 2/9 cleanup.
- **Acceptance criteria:** orphan components/store removed or explicitly re-adopted; no duplicate FO endpoint definitions.

---

## FO-P1-012 — Thin frontend FO test coverage; backend capability gaps

- **Category:** QA
- **Description:** frontend FO coverage is limited to 3 files; many backend capabilities have no tests; E2E is a single navigation test.
- **Evidence (direct):** frontend tests: `apps/web/features/front-office/utils/__tests__/tax-computation.test.ts`, `apps/web/features/front-office/components/dialogs/__tests__/fo-change-rate-failure.test.tsx`, `apps/web/components/__tests__/module-switcher.test.tsx`. No test files under `apps/web/app/(dashboard)/front-office/**`. Backend uncovered: walk-in, reverse check-in, discrepancies, sub-resource commands, registration card, dashboard/arrivals/due-outs/overstayed/checkout-history handlers (`apps/api/src/modules/front-office/**`). E2E: `e2e/kebab-actions.spec.ts` (one FO handoff test).
- **Impact:** regressions in core FO UI/backend paths may go undetected; coverage confidence low.
- **Severity / confidence:** **P2** / high.
- **Classification:** Potential risk (coverage gap).
- **Follow-up phase:** Phase 2 and ongoing gates.
- **Acceptance criteria:** added tests for walk-in/reverse/discrepancy handlers and key FO components; at least one FO workflow E2E.

---

## FO-P1-013 — Referenced FO source-of-truth audits are missing

- **Category:** DOC
- **Description:** `docs/enterprise/front-office-frontend-rebuild-architecture-2026-08-02.md:6-7` cites `docs/audit/front-office-frontend-discovery-audit-2026-08-02.md` and `docs/audit/front-office-architecture-audit-2026-08-02.md` as locked source-of-truth. Neither exists.
- **Evidence (direct):** `glob docs/audit/front-office*` → none; `docs/audit/` contains only `Reservations/`.
- **Impact:** the baseline cannot be reconciled against the referenced "locked" discovery audit; provenance gap.
- **Severity / confidence:** **Info** / high.
- **Classification:** Unverified/dependency.
- **Follow-up phase:** Phase 2 (locate, restore, or re-derive).

---

## FO-P1-014 — Duplicate ERP module entries map to `/front-office`

- **Category:** UX / navigation
- **Description:** the ERP registry lists `front-office` (status `ready`) and `front-desk` (status `integration-pending`) both at `path: '/front-office'`.
- **Evidence (direct):** `apps/web/lib/os/erpModules.ts:56` and `:58`.
- **Impact:** duplicate launcher destination; "integration-pending" status on a route that is actually implemented is misleading.
- **Severity / confidence:** **P3** / high.
- **Classification:** Confirmed defect.
- **Follow-up phase:** Phase 2/9.

---

## FO-P1-015 — Rack HK-status filter compares the wrong field

- **Category:** FE / functional
- **Description:** in `RoomRack.matchesFilters`, the housekeeping-status filter compares `filters.hkStatus` against `room.status` instead of `room.housekeepingStatus`, so the HK filter can never match correctly.
- **Evidence (direct):** `apps/web/features/front-office/components/rack/RoomRack.tsx:18` (`if (filters.hkStatus !== 'all' && room.status !== filters.hkStatus) return false;`). The companion status filter at `:17` also uses `room.status`.
- **Impact:** housekeeping-status filtering in the rack is wrong; operators cannot isolate dirty/clean rooms reliably.
- **Severity / confidence:** **P3** / high.
- **Classification:** Confirmed defect.
- **Follow-up phase:** Phase 2/9.

---

## FO-P1-016 — Duplicate checkout-history implementations

- **Category:** FE / consistency
- **Description:** three checkout-history surfaces exist: legacy `apps/web/components/CheckoutHistoryModal.tsx` (orphan), feature-local `features/front-office/components/guest-services/CheckoutHistoryModal.tsx` (exported, unrendered), and `features/front-office/components/dashboard/CheckoutHistoryPanel.tsx` (rendered in the departures tab).
- **Evidence (direct):** grep results for `CheckoutHistoryModal` (two files + index export); `dashboard/CheckoutHistoryPanel.tsx` rendered in `Dashboard.tsx:145`.
- **Impact:** conceptual duplication; risk of divergence.
- **Severity / confidence:** **Info** / high.
- **Classification:** Potential risk.
- **Follow-up phase:** Phase 2/9 cleanup.

---

## FO-P1-017 — Pervasive raw SQL (`$queryRawUnsafe` / `$executeRawUnsafe`)

- **Category:** DATA / maintainability
- **Description:** the FO module (and its check-in submodule) uses raw SQL extensively for JSONB/new tables and change logs. Most calls are parameterized, but `$queryRawUnsafe`/`$executeRawUnsafe` is used broadly.
- **Evidence (documentation-derived from code):** `front-office.controller.ts:372`; `front-office.service.ts:153,160`; `application/services/traces-inventory.service.ts:71`; `application/services/room-moves-dashboard.service.ts:46,85,113`; `check-out.handler.ts:294,302,314,328,404,410,457,465,678`; `reverse-check-in.handler.ts:79,90`; `transfer-room.handler.ts:73,82,112`; `upgrade-room.handler.ts:135,277`; `change-rate-code.handler.ts:157`; `sub-resources.handlers.ts:393,405,409,433,434`; `front-office.query.handlers.ts:127,398,447,494,805`; `discrepancy.repository.ts:41,65,85`; `check-in-commit.service.ts:94`; `property-config.adapter.ts:47`; `check-in-flow.repository.ts:21,58,99`; `check-in-guest.handler.ts:49,73,84,100`; `release-room-assignment.handler.ts:45-48,56`; `get-check-in-flow.handler.ts:53,59`; `crs-front-desk-integration.service.ts:133`.
- **Impact:** SQL strings are not covered by Prisma's type-safety; maintenance/refactor risk; injection risk **only if** any interpolation exists (not confirmed here — flagged as a review item, not a defect).
- **Severity / confidence:** **Info** / medium.
- **Classification:** Potential risk.
- **Follow-up phase:** Phase 2 review (handled elsewhere as DB hardening).

---

## Summary

**Totals: 17 findings — P1 = 1, P2 = 6, P3 = 7, Info = 3.**

| Severity | Count | IDs |
|---|---|---|
| P0 | 0 | — |
| P1 | 1 | 001 |
| P2 | 6 | 002, 003, 004, 005, 006, 012 |
| P3 | 7 | 007, 008, 009, 010, 011, 014, 015 |
| Info | 3 | 013, 016, 017 |

| Classification | IDs |
|---|---|
| Confirmed defect | 001, 002, 003, 004, 005, 007, 009, 010, 014, 015 |
| Partial implementation | 006 |
| Potential risk | 011, 012, 016, 017 |
| Improvement opportunity | 008 |
| Unverified | 013 |

---

*End of Phase 1 gap register.*

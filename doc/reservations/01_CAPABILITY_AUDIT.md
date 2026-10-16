# Reservations — Capability & Feature-Gap Audit

**Date:** 2026-10-16
**Scope:** Reservations capability scope (11 items) across backend, frontend, persistence, integration, and tests.
**Type:** Audit only. No application code, schema, migration, deployment, or data changes were made.
**Method:** Static inspection (file reads, targeted greps) plus reuse of existing remediation evidence. No broad Jest/build jobs were run and no database was written to.

**Related prior authority (reused, not re-audited):**
- `docs/enterprise/reservations-roadmap-v2.md`
- `docs/RESERVATIONS_FORENSIC_AUDIT.md`
- `docs/reservations/phase-1/01_PHASE1_ASSESSMENT_REPORT.md`, `02_PHASE1_CLOSURE_AND_ROADMAP.md`
- `docs/reservations/phase-3/01_R8_CLASSIFICATION_LEDGER.md`
- `docs/audit/Reservations/*`

**Evidence tags:** `[V]` verified by direct file read/grep this session; `[D]` taken from an authoritative existing document; `[H]` hypothesis / requires runtime confirmation.

---

## 0. Executive Summary

The Reservations module has a **production-grade core** and a **weak revenue-intake and reporting perimeter**.

- The **domain and write path are strong**: a real `Reservation` aggregate, an enforced status state machine, CQRS commands/queries, and a rigorous availability-assertion engine with row-level locking, idempotent operation journaling, and fail-closed behavior.
- The **inventory-integrity engine is the standout**: `AvailabilityAssertionService` locks balance rows in canonical ascending date order (`FOR UPDATE`), rechecks idempotency inside the transaction, and rejects on unresolved capacity. [V]
- **Channel/OTA intake is not operationally functional.** The inbound webhook route is behind global `JwtAuthGuard` + `PropertyScopeGuard` + `MultiTenantGuard` with no `@Public()`/`@PropertyScope(false)` exemption, so unauthenticated OTA callbacks are rejected before the handler. Outbound channel "push" writes an audit-log row only. [V]
- **Website booking intake does not exist.** No public booking route group or endpoint. [V]
- **Search/saved-views/history are partially broken**: several declared search filters are silently dropped; the display-sets endpoint queries a table that does not exist in the schema; the "History" tab fabricates a timeline in the browser instead of reading the audit tables the backend writes. [V]
- **Deposit/guarantee ownership is unresolved**: deposits are written to `folio_payments` by the Reservations repository, read from `reservation_deposits` by the billing adapter, and modeled separately in `folios`/`folio_postings` by the Billing module. `DepositPolicyService` is unused outside tests. [V]

**Overall readiness: PARTIAL — not ready for an OTA-dependent revenue model.** Core booking integrity is trustworthy; the intake and reporting surfaces are not.

---

## 1. Capability Findings

### Capability 1 — Website, OTA, and channel booking intake
**Status: FAIL (operationally).** `[V]`

**Evidence**
- Channel CRUD + config: `apps/api/src/modules/channels/channels.controller.ts` (`@Controller('channels')`) and `channels.service.ts`. Real DB CRUD against `channels`. [V]
- Inbound webhooks: `apps/api/src/modules/channels/webhook-ingress/webhook.controller.ts` (`@Controller('webhooks')`, `POST :channel`) → `webhook.service.ts` (`WebhookService.process`). Normalizers exist for `booking-com` and `expedia` only; HMAC-SHA256 signature verification via `verifySignature`; hotel/room/rate resolution via `channel_*_mapping`; creation delegates to `ReservationsService.create(...)`. [V]
- **Blocking defect:** `WebhookController` (and `ChannelsController`) carry no `@Public()` / `@PropertyScope(false)`. The global guards registered in `apps/api/src/app.module.ts:107-109` (`JwtAuthGuard`, `PropertyScopeGuard`, `MultiTenantGuard`) therefore apply. `PropertyScopeGuard` resolves to `NONE` for a request with no principal and no `x-property-id` header and returns `false` → 403. `MultiTenantGuard` likewise returns `false` without a tenant header. `JwtAuthGuard.PUBLIC_PATHS` (`jwt-auth.guard.ts:8-15`) lists `/^\/webhooks\/.+/`, but the app calls `app.setGlobalPrefix('api/v1')` (`main.ts:21`), so `request.path` is `/api/v1/webhooks/...` and the allowlist regex does not match `[H]`. Either layer blocks the callback. [V]
- Outbound push: `channels.service.ts` `pushRatesToChannels` / `pushAvailabilityToChannels` only `INSERT` into `channel_availability_log` and return `{ status: 'queued' }`; no channel adapter transmits anything. [V]
- `getChannels()` fabricates metrics: `revenue: bookings * 350`, `lastSyncStatus: 'success'` hardcoded, `syncFrequency: 15`. [V]
- `pullReservationsFromChannels()` reads `channel_availability` rows, not reservations — the name does not match the behavior. [V]
- No website/direct booking surface: no `apps/web/app/(public)` group, no `book` route, no booking-engine controller. [V]

**Actual behavior:** A well-scaffolded webhook normalizer that real OTAs cannot reach; channel management that stores configuration and logs intent but performs no distribution.

**Missing / failure modes:** OTA callbacks 401/403; no website booking; no real two-way channel sync; fabricated operational metrics can mislead the channel dashboard.

**Dependencies:** Global guard policy; channel adapter implementation; OTA credentials.

**Next action:** Make the webhook surface explicitly public (guard-level exemption) and prove it with a route-level test; then implement (or clearly mark as non-functional) outbound distribution.

---

### Capability 2 — Reservation lifecycle: create, modify, confirm, cancel, reinstate
**Status: PASS.** `[V]`

**Evidence**
- Controller routes: `reservations.controller.ts` — `POST /reservations` (create), `PUT/PATCH /:id` (modify/amend), `POST :id/cancel`, `POST :id/no-show`, `POST :id/confirm`, `POST :id/guarantee`, `POST :id/reinstate`, `POST :id/extend`, `PUT :id/room-type`, `PUT :id/rate`, `POST batch/status`, `DELETE :id`. [V]
- Commands registered in `reservations.module.ts:250-311` (Create, Update, Cancel, ProcessNoShow, Delete, BatchUpdateStatus, Confirm, Guarantee, Reinstate, ExtendStay, ChangeRoomType, ChangeRate, …). [V]
- State machine single enforcement point: `domain/services/reservation-status.service.ts` wraps `assertTransition/normalizeStatus` from `@xylo/shared`; historical storage form handles `NO-SHOW` vs `NO_SHOW`. [V]
- Cancel path is substantive: `cancel-reservation.handler.ts` applies `CancellationPolicyService` + `CancellationSettlementService`, emits `CancellationPenaltyDue`/`Settled` events. [V]
- Repository lifecycle methods transact through `joinTransaction(this.prisma, ...)` and write status transitions. [V]
- Aggregate: `domain/entities/reservation.ts` owns `cancel/confirm/setGuarantee/reinstate/noShow/extendStay/changeRoomType/changeRate`. [V]

**Limitations:** Modify/amend is route- and guard-limited (`ProtectedAmendFieldsGuard`); cross-property `TransferReservation` remains deferred per ADR-072 (documented owner constraint, not a defect).

**Tests (existence):** `reservations/__tests__/reservation-status.spec.ts`, `reservation-aggregate.spec.ts`, `reservation-integration.spec.ts`, `cancellation-penalty.spec.ts`, `scheduled-room-moves.spec.ts`, `t552-write-identity.spec.ts`; Postgres specs `reservation-cancel.postgres.spec.ts`, `reservation-delete.postgres.spec.ts`. These were not executed in this audit.

---

### Capability 3 — Guest details, stay dates, occupancy, rates, and room types
**Status: PASS (with minor notes).** `[V]`

**Evidence**
- Value objects: `domain/value-objects/stay.value-object.ts` (arrival/departure/adults/children, LOS, occupancy), `rate-plan.value-object.ts`, `guarantee.value-object.ts`, `holds.value-object.ts`. [V]
- Aggregate primitives cover stay, ratePlan, roomType, roomNumber, guests, source/market, company/agent, group/block/allotment, confirmation, payment. [V]
- Guest linkage: repository `create` extracts profile fields (`firstName`/`lastName`/`email`/`phone`/…) and links `guest_id`; `findAll` resolves guest names from `guests`. [V]
- Rates read from `rate_details` in `findAll` enrichment. [V]

**Notes:** Prior audit finding **F12** (frontend collects dead create-form fields) is carried forward as an observation, not re-audited.

---

### Capability 4 — Availability calculation and booking-overlap protection
**Status: PASS.** `[V]`

**Evidence**
- `AvailabilityAssertionService` (`apps/api/src/modules/availability/application/services/availability-assertion.service.ts`) is bound as `RESERVATION_AVAILABILITY_PORT` by `availability.module.ts:49` and exported (`:60`). [V]
- Assertions lock `availability_assertion_balances` rows (`FOR UPDATE`) in canonical ascending date order, insert-on-conflict first, recheck the idempotency key inside the transaction, and fail closed on `UNRESOLVED_CAPACITY` / `CAPACITY_BLOCKED` / `INSUFFICIENT_CAPACITY`. [V]
- Create/update/cancel are wired through `application/services/reservation-availability-wiring.ts` (`assertReservationCreate`, `createConsumesAvailability` fail-closed for unknown statuses) and the repository. [V]
- Read models exist: `GET /reservations/view/arrivals|departures|in-house`. [V]

**Limitations:** Capacity truth depends on the Phase-1 snapshot (`AvailabilitySnapshotService.getSnapshot`); when unresolved, booking fails closed by design. [V]

---

### Capability 5 — Groups, blocks, allotments, and rooming lists
**Status: PARTIAL.** `[V]`

**Evidence (real)**
- Full bounded module `apps/api/src/modules/group-allotment/**`: aggregates `GroupBooking`, `GroupBlock`, `Allotment`; entities for pickups, daily allocations, vouchers, stop-sales; domain services (lifecycle, attrition, shoulder); ports/adapters (pickup-availability, reservation association, folio, billing instruction); pickup-availability assertions. [V]
- Routes: `group-booking.controller.ts` (`@Controller('group-bookings')` — blocks, grid, allocations, pickups, release, shoulder, categories, master folio) and `allotment.controller.ts` (`@Controller('allotments')` — quotas, vouchers, stop-sales, pickups, negotiated rates). [V]
- Persistence is real: `group_bookings`, `group_blocks`, `group_block_daily_allocations`, `group_pickups`, `allotment_contracts`, `allotment_daily_quotas`, `allotment_vouchers` in `packages/db/schema.prisma`. [V]
- Frontend routes `apps/web/app/(dashboard)/reservations/group-blocks/**` and `.../allotments/**` render real feature views. [V]
- Rooming list: `GroupBookingDetailView.tsx` `RoomingList` uses `useCreateGroupPickup` → `POST /group-bookings/:bookingId/blocks/:blockId/pickups` (persists). [V]

**Gaps**
- "Export CSV" and "Import Rooming List" buttons have no `onClick` handler; there is no bulk rooming-list import/export endpoint. [V]
- Rooming list is expressed as pickups, not as a first-class named manifest with room assignments. [V]

**Tests (existence):** `group-allotment/__tests__/*` incl. `hotel-isolation.spec.ts`, `t65-legacy-write-absence.spec.ts`, `t61-error-semantics.contract.spec.ts`; Postgres specs `infrastructure/__tests__/voucher-code-unique.postgres.spec.ts`, `t68-canonical-write-switch.postgres.spec.ts`. Not executed here.

---

### Capability 6 — Search, filters, saved views, and reservation history
**Status: PARTIAL** (search PARTIAL · saved views FAIL · history read FAIL). `[V]`

**Evidence**
- Search route `GET /reservations` → `SearchReservationsHandler` → `repo.findAll`. Working filters in `reservation.repository.ts:88-184`: `dateRange`, `status`, `dateFrom/dateTo`, `dateFilter`, `roomType`, `minAdults`, free-text `search` (guest name via `guests`, `confirmation_no`, `id`), and `flexFieldFilters`. [V]
- **Silently ignored filters:** `SearchReservationDto` (`api/dto/search-reservation.dto.ts:16-23`) accepts `confirmationNo`, `guestName`, `roomNumber`, `companyName`, `agentName`, `marketCode`, `sourceCode`, `reservationType`, and the frontend sets them (`features/reservations/api/reservation.api.ts:202-209`), but the repository where-clause never reads them. Results are not narrowed. [V]
- **Saved views / display sets:** `GET /reservations/display-sets` → `GetDisplaySetsHandler` runs `SELECT ... FROM reservation_display_sets`. That table does **not** exist anywhere in `packages/db` (only match is the handler itself). The query throws, the handler returns `Result.failure`, and the frontend `useDisplaySets` hook surfaces an error. No create/update persistence exists. [V]
- **Reservation history:** tables `reservation_status_history` and `reservation_changes` exist (`schema.prisma:9601`, `:9231`) and are written by the repository (status transitions, cancel, delete, batch, room moves). [V] However, there is **no read endpoint** for them, and the UI `AuditPanel` (`workspace/detail/panels/AuditPanel.tsx`) fetches `/reservations/:id/deposits` and then constructs a synthetic timeline from the in-memory reservation object (`const auditLog: AuditEntry[] = []`), not from the audit tables. [V]

**Actual behavior:** Users can search by text/date/status/room type; dedicated field filters are no-ops; the rate "display set" selector errors; the History tab shows a reconstructed, non-authoritative timeline.

**Tests (existence):** `features/reservations/api/__tests__/reservation-list-contract.test.ts` asserts list contract shape; it does not assert filter application. No test covers display-sets persistence or history reads.

---

### Capability 7 — Deposit / booking guarantee (ownership)
**Status: PARTIAL — duplicated ownership.** `[V]`

**Evidence**
- API surface owned by Reservations: `POST /:id/deposits`, `GET /:id/deposits`, `POST /:id/deposits/transfer`, `POST /:id/guarantee` (`reservations.controller.ts`), guarded by `reservations:deposit:post` / `reservations:reservation:view`. [V]
- `ReservationRepository.postDeposit` writes and reads **`folio_payments`** (creating a `folio_header` row idempotently). `reservations.service.ts getDeposits` also reads `folio_payments`. [V]
- `PrismaBillingAdapter` (`infrastructure/adapters/prisma-billing.adapter.ts`) reads **`reservation_deposits`** for deposit status/settlement and writes `folio_postings` under `folios`. Its own comment asserts `reservation_deposits` is the deposit store. [V]
- The Billing module models folios with `folio_header`/`folio_payments`; `reservation_payment` (legacy `resv_name_id`-keyed) is also read. [V]
- `DepositPolicyService` (the single rule for whether a deposit is required) is registered and exported but has **no production consumer** — only tests import it. [V]
- Guarantee value object exists (`domain/value-objects/guarantee.value-object.ts`) and `POST :id/guarantee` is wired. [V]

**Actual behavior / failure mode:** The deposit *writer* (`folio_payments`) and the deposit *status reader* in the billing adapter (`reservation_deposits`) disagree; settlement may not see deposits that were posted through the Reservations path, and vice versa. Policy is not enforced automatically.

**Verdict:** Ownership is genuinely split. Recommended target: **Billing owns the deposit ledger; Reservations owns deposit/guarantee policy + the HTTP surface, via one shared port.** As-is: **duplicated and inconsistent.**

---

### Capability 8 — Check-in / check-out execution
**Status: OUT OF SCOPE (Front Office).** Not assessed as a Reservations gap, per the approved scope.

**Integration contract observed:** `apps/api/src/modules/front-office/application/commands/check-out/check-out.handler.ts` and `upgrade-room/upgrade-room.handler.ts` consume `RESERVATION_AVAILABILITY_PORT`, so Front Office correctly asserts/releases inventory through the Reservations-owned availability contract. `[V]`

---

### Capability 9 — Room readiness and departure worklists
**Status: OUT OF SCOPE (Front Office).**

---

### Capability 10 — Authorization and tenant/property isolation
**Status: PARTIAL.** `[V]`

**Evidence (strong on the reservations module)**
- Global guards: `app.module.ts:107-109` registers `JwtAuthGuard`, `PropertyScopeGuard`, `MultiTenantGuard`; `PermissionGuard` is also registered as `APP_GUARD` (`platform/permission/permission.module.ts:26`, `common/authorization/authorization.module.ts:19`). [V]
- `ReservationsController` is `@PropertyScope(true)` and every route carries `@Permission(...)`. [V]
- `PropertyScopeResolver` rejects scope set by client input alone; accepts it only when it matches the principal identity or a cross-property grant (`SUPER_ADMIN`, `PLATFORM_ADMIN`, or an unexpired membership). [V]
- Repository reads/writes are hotel-scoped via `requireHotelId()`/`getHotelId()` ALS-backed context. [V]
- Availability module ships isolation regression specs (`t564-api-isolation`, `t565-cross-hotel-write`, `t566-property-scope-equivalence`, `t567-async-scoping`). [V, not executed]

**Gaps**
- `ChannelsController` and `WebhookController` declare **no `@Permission`** and no `@PropertyScope`. Because `PermissionGuard` fails open when no permission metadata is present (`permission.guard.ts:24`), any authenticated, property-scoped user can create/update/delete channels and trigger push/pull. [V]
- The webhook guard interaction (Capability 1) is also an isolation/authorization defect (public surface not exempted). [V]
- `findAll` still reads `const hotelId = getHotelId() || params.hotelId;` (`reservation.repository.ts:91`) — an internal scope fallback that bypasses ALS if a caller ever supplies `hotelId`. Over HTTP the DTO exposes `propertyId` (ignored), so this is defense-in-depth, not an open hole. [V]
- Two `PermissionGuard` `APP_GUARD` registrations create ambiguity about the single authorization authority. [V]

---

### Capability 11 — Inventory concurrency and booking integrity
**Status: PASS (with caveats).** `[V]`

**Evidence**
- Row-level locking with `FOR UPDATE` in canonical ascending date order; `ON CONFLICT DO NOTHING` first-touch; in-transaction idempotency recheck; `ReadCommitted` with 30s timeout. [V]
- Durable operation journal (`ReservationOperationJournal`) claims `ASSERT`/`RELEASE`/`REPLACE` before state reads (`D-24`), so concurrent cancel/modify on the same reservation serialize and replay deterministically. [V]
- Fail-closed semantics on unresolved/absent capacity (`unassertableReason`, `validateContext` rejecting `'default'`/empty hotel). [V]
- Create handler rejects a missing or `'default'` hotel id (`create-reservation.handler.ts:26-28`). [V]

**Caveats / dependencies**
- Concurrency proofs live in Postgres specs gated by `AVAILABILITY_TEST_DATABASE_URL` (e.g. `availability-assertion-concurrency.postgres.spec.ts`, `availability-lock-order.postgres.spec.ts`). Without that env var they `describe.skip` — so in a default run the strongest integrity claims are **not exercised**. [V]
- The inbound webhook path does not supply a channel-level idempotency key (it relies on `command.commandId`), so the only duplicate defense in that path is the pre-transaction `confirmation_no` lookup — a check-then-act race. [V]

---

## 2. Confirmed Strengths

1. Availability-assertion engine with deterministic lock ordering, idempotency journal, and fail-closed behavior. `[V]`
2. Enforced reservation status state machine as a single enforcement point. `[V]`
3. A real domain aggregate owning lifecycle/pricing/status behavior, decoupled from Prisma/Nest. `[V]`
4. Complete lifecycle command surface with CQRS registration and event emission. `[V]`
5. Group/allotment bounded context with real aggregates, routes, and persistence. `[V]`
6. Property-scope resolution that cannot be set by client input alone for authenticated principals. `[V]`
7. Front Office consumes the Reservations availability port rather than re-implementing inventory truth. `[V]`

## 3. Critical Gaps (summary; full register in `02_GAP_REGISTER.md`)

- **RG-01 (Critical):** OTA webhook ingress is unreachable behind global guards.
- **RG-02 (High):** Channel outbound distribution is a log write; channel metrics fabricated.
- **RG-03 (High):** Declared search filters silently ignored.
- **RG-04 (High):** Saved views / display sets query a non-existent table.
- **RG-06 (High):** Deposit/guarantee ledger ownership duplicated across three stores.
- **RG-05 (Medium):** Reservation history is written but never exposed; UI fabricates a timeline.
- **RG-07 (Medium):** Missing authorization on the channels surface.

## 4. Unresolved Questions / NOT VERIFIED

- **NOT VERIFIED (runtime):** Whether `request.path` in the deployed API includes the `/api/v1` prefix (affects `JwtAuthGuard.PUBLIC_PATHS`). Static evidence indicates it does; a route-level test would confirm. `[H]`
- **NOT VERIFIED (runtime):** Postgres concurrency/isolation specs were not executed (no local test DB; `AVAILABILITY_TEST_DATABASE_URL` unset). Claims rest on source inspection plus existing documents.
- **NOT VERIFIED:** Whether `reservation_display_sets` exists in any environment outside the repository schema (no migration found in-repo).
- **NOT VERIFIED:** Live OTA credentials/behavior; only the code path was inspected.
- Carried-forward, not re-audited: the known pre-existing red frontend test `features/reservations/__tests__/b31-fnd09.test.ts` (unrelated to these capabilities).

## 5. Limitations

- No database was read or written; no server was started; no broad test/build job was run.
- Findings are from static inspection and existing audit artifacts. Where a claim depends on runtime behavior it is tagged `[H]`/NOT VERIFIED.
- Front Office-owned capabilities (8, 9) were intentionally excluded from the gap inventory.

## 6. No-Change Confirmation

No application code, tests, schema, migrations, deployments, or data were modified. Only the three audit documents under `docs/reservations/feature-gap-audit/` were created.

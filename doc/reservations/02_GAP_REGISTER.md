# Reservations — Gap Register

**Date:** 2026-10-16
**Companion:** `01_CAPABILITY_AUDIT.md`
**Type:** Audit artifact. Deduplicated gaps only. No fix is authorized by this document.

Severity scale: **Critical** = blocks revenue or corrupts/duplicates money or inventory · **High** = capability advertised but materially broken · **Medium** = degrades operations/audit · **Low** = hygiene / defense-in-depth.
Confidence: `CONFIRMED` = code-inspected · `SUSPECTED` = reported, not fully verified · `RUNTIME` = needs execution to confirm.

| ID | Capability | Severity | Confidence | Summary |
|----|-----------|----------|-----------|---------|
| RG-01 | 1 | Critical | CONFIRMED (runtime caveat) | OTA webhook ingress blocked by global guards |
| RG-02 | 1 | High | CONFIRMED | Channel outbound push is a log write; metrics fabricated |
| RG-03 | 6 | High | CONFIRMED | Declared search filters silently ignored |
| RG-04 | 6 | High | CONFIRMED | Display sets / saved views query a non-existent table |
| RG-05 | 6 | Medium | CONFIRMED | Reservation history written but not exposed; UI fabricates timeline |
| RG-06 | 7 | High | CONFIRMED | Deposit/guarantee ledger ownership duplicated across three stores |
| RG-07 | 10 | Medium | CONFIRMED | Channels surface has no permission checks; PermissionGuard fails open |
| RG-08 | 1, 11 | Medium | CONFIRMED | Webhook intake has no channel-level idempotency key (check-then-act gap) |
| RG-09 | 1 | Medium | CONFIRMED | Website / direct booking intake absent |
| RG-10 | 5 | Medium | CONFIRMED | Rooming-list bulk import/export not implemented |
| RG-11 | 10, 11 | Low | CONFIRMED | `findAll` residual `params.hotelId` scope fallback |
| RG-12 | 10 | Low | CONFIRMED | Two `PermissionGuard` `APP_GUARD` registrations (authority ambiguity) |
| RG-13 | 11 | Medium | CONFIRMED | Strongest integrity specs skip without `AVAILABILITY_TEST_DATABASE_URL` |
| RG-14 | 7 | Medium | CONFIRMED | `DepositPolicyService` has no production consumer |

---

## RG-01 — OTA webhook ingress blocked by global guards
- **Capability:** 1 (Channel/OTA intake)
- **Severity:** Critical
- **Evidence:** `apps/api/src/modules/channels/webhook-ingress/webhook.controller.ts` (`@Controller('webhooks')`, no `@Public()`, no `@PropertyScope`); `apps/api/src/app.module.ts:107-109` (global `JwtAuthGuard`, `PropertyScopeGuard`, `MultiTenantGuard`); `core/guards/property-scope.guard.ts` (returns `false` when resolution ≠ RESOLVED); `core/guards/multi-tenant.guard.ts:38` (returns `false` without tenant); `core/guards/jwt-auth.guard.ts:8-15` + `main.ts:21` (allowlist regex vs `/api/v1` prefix).
- **Actual behavior:** An unauthenticated OTA callback with no `x-property-id`/`x-tenant-id` header is rejected (401 via JWT, or 403 via property/tenant guard) before `WebhookService.process` runs. Only unit tests instantiate the service directly; no HTTP-level test exists.
- **Impact:** Booking.com/Expedia reservations cannot be received. Silent revenue loss; the module appears present but is inert.
- **Dependencies:** Global guard policy; the intended public-surface contract (`PropertyScopeResolver` comment already anticipates webhook/widget requests).
- **Acceptance criteria:** `POST /api/v1/webhooks/:channel` is reachable unauthenticated and processes a validly-signed OTA payload end-to-end (creates a hotel-scoped reservation); an invalid signature is rejected 400; a route-level test exercises the real guard chain.

## RG-02 — Channel outbound distribution is a log write; metrics fabricated
- **Capability:** 1
- **Severity:** High
- **Evidence:** `channels.service.ts` `pushRatesToChannels`/`pushAvailabilityToChannels` insert into `channel_availability_log` and return `status: 'queued'`; `getChannels()` returns `revenue: bookings * 350`, `lastSyncStatus: 'success'`, `syncFrequency: 15`; `pullReservationsFromChannels` reads `channel_availability`.
- **Impact:** No rate/availability is actually distributed to OTAs; overbooking risk if operators rely on "queued"/"success"; misleading revenue figures on the channel screen.
- **Dependencies:** Channel adapter layer; OTA APIs.
- **Acceptance criteria:** Either a real adapter transmits and reports per-channel results, or the endpoints and UI clearly mark the feature as not-yet-implemented and stop emitting fabricated metrics.

## RG-03 — Declared search filters silently ignored
- **Capability:** 6 (Search/filters)
- **Severity:** High
- **Evidence:** `api/dto/search-reservation.dto.ts:16-23` declares `confirmationNo`, `guestName`, `roomNumber`, `companyName`, `agentName`, `marketCode`, `sourceCode`, `reservationType`; `features/reservations/api/reservation.api.ts:202-209` sends them; `reservation.repository.ts:88-184` applies only `dateRange/status/dateFrom/dateTo/dateFilter/roomType/minAdults/search/flexFieldFilters`.
- **Actual behavior:** Filters are accepted (no 400) but have no effect — users get broader results than requested with no error.
- **Impact:** Operational search false positives; front-desk staff cannot narrow by confirmation number / guest / company reliably.
- **Dependencies:** Repository query builder.
- **Acceptance criteria:** Each declared filter either narrows the result set (with a test proving it) or is removed from the DTO/UI.

## RG-04 — Display sets / saved views query a non-existent table
- **Capability:** 6 (Saved views)
- **Severity:** High
- **Evidence:** `application/queries/get-display-sets/get-display-sets.handler.ts:45` → `FROM reservation_display_sets`; no such model/table anywhere in `packages/db` (only match is this handler). Route `GET /reservations/display-sets`. Frontend `hooks/use-display-sets.ts` / `api/sales-screen.api.ts:29`.
- **Actual behavior:** The query throws; the handler returns `Result.failure`; the hook errors. The hardcoded default list is unreachable because the failure happens before the `length === 0` fallback.
- **Impact:** No saved views exist; the Sales Screen rate display-set selector is non-functional.
- **Dependencies:** A real persistence table/model if saved views are in scope.
- **Acceptance criteria:** Either a persisted `reservation_display_sets` store with CRUD, or the endpoint returns the documented defaults without touching a missing table; covered by a test.

## RG-05 — Reservation history written but not exposed; UI fabricates timeline
- **Capability:** 6 (History)
- **Severity:** Medium
- **Evidence:** `schema.prisma:9601` `reservation_status_history`, `:9231` `reservation_changes`; writes in `reservation.repository.ts:721,810,815,868,910,1056,1121,1126`. No read route in `reservations.controller.ts`; `workspace/detail/panels/AuditPanel.tsx` fetches `/reservations/:id/deposits` and builds `auditLog` client-side from the reservation object.
- **Impact:** No authoritative audit trail is available to users; the "History" tab can misrepresent what happened. Compliance/audit weakness.
- **Dependencies:** A history read model/query over the existing tables.
- **Acceptance criteria:** A `GET` returns status/field changes for a reservation, scoped by hotel, and the History tab renders it; test asserts real rows are surfaced.

## RG-06 — Deposit/guarantee ledger ownership duplicated across three stores
- **Capability:** 7 (Deposit/guarantee)
- **Severity:** High
- **Evidence:** `reservation.repository.ts:1267` `postDeposit` writes `folio_payments` (+ `folio_header`); `reservations.service.ts` `getDeposits` reads `folio_payments`; `prisma-billing.adapter.ts:90,169` reads `reservation_deposits` and writes/reads `folio_postings`/`folios`; schema has `folio_header`, `folio_payments`, `reservation_deposits`, `reservation_guarantee`, `reservation_payment`, `folios`, `folio_postings`.
- **Actual behavior:** Writer and reader of deposit state disagree across paths; settlement/front-desk handoff can miss deposits posted through the Reservations route (and vice versa).
- **Impact:** Financial inconsistency; refund/settlement errors; reconciliation pain.
- **Dependencies:** Billing module; a single shared deposit port.
- **Acceptance criteria:** One authoritative deposit ledger; Reservations reads/writes through one port; a test proves a deposit posted via Reservations is visible to the billing/settlement path.

## RG-07 — Channels surface has no permission checks; PermissionGuard fails open
- **Capability:** 10 (Authorization)
- **Severity:** Medium
- **Evidence:** `channels.controller.ts` / `webhook-ingress/webhook.controller.ts` have no `@Permission` / `@PropertyScope`; `platform/permission/permission.guard.ts:24` returns `true` when no permission metadata is present.
- **Impact:** Any authenticated, property-scoped user can create/modify/delete channel configuration and trigger push/pull.
- **Dependencies:** Permission model for channel administration.
- **Acceptance criteria:** Channel admin routes require an explicit permission code; a test asserts denial without it.

## RG-08 — Webhook intake has no channel-level idempotency key
- **Capability:** 1, 11
- **Severity:** Medium
- **Evidence:** `webhook.service.ts:59-68` dedups via a `SELECT` on `confirmation_no` before calling `ReservationsService.create`; `create-reservation.handler.ts:33` falls back to `command.commandId` for the assertion idempotency key.
- **Actual behavior:** Duplicate OTA deliveries are prevented only by a check-then-act read; concurrent duplicates can both pass the pre-check and book twice.
- **Dependencies:** A stable OTA idempotency key mapped into the availability assertion key.
- **Acceptance criteria:** Replaying the same `channelReservationId` never creates a second reservation, including concurrently; proven by a test.

## RG-09 — Website / direct booking intake absent
- **Capability:** 1
- **Severity:** Medium
- **Evidence:** No `(public)` route group, no `book` page under `apps/web/app/**`, no booking-engine controller in `apps/api`.
- **Impact:** Direct channel revenue cannot be captured in-product.
- **Acceptance criteria:** A documented, public, rate-limited booking intake (or an explicit decision that direct booking is out of scope).

## RG-10 — Rooming-list bulk import/export not implemented
- **Capability:** 5
- **Severity:** Medium
- **Evidence:** `features/group-allotment/views/GroupBookingDetailView.tsx:660-661` "Export CSV" / "Import Rooming List" buttons have no handler; no bulk endpoint in `group-booking.controller.ts`.
- **Impact:** Groups cannot be onboarded at scale; manual entry only.
- **Acceptance criteria:** Import/export endpoints with validation, or removal of the buttons with a documented decision.

## RG-11 — `findAll` residual `params.hotelId` scope fallback
- **Capability:** 10, 11
- **Severity:** Low
- **Evidence:** `reservation.repository.ts:91` `const hotelId = getHotelId() || params.hotelId;`.
- **Impact:** Defense-in-depth only (HTTP DTO exposes `propertyId`, ignored); an internal caller passing `hotelId` could bypass ALS scope.
- **Acceptance criteria:** Property scope derives solely from the resolved request context.

## RG-12 — Two `PermissionGuard` `APP_GUARD` registrations
- **Capability:** 10
- **Severity:** Low
- **Evidence:** `platform/permission/permission.module.ts:26` and `common/authorization/authorization.module.ts:19` both register a `PermissionGuard` as `APP_GUARD`.
- **Impact:** Ambiguous single authority; double evaluation; confusing failure modes.
- **Acceptance criteria:** Exactly one global permission guard.

## RG-13 — Strongest integrity specs skip without `AVAILABILITY_TEST_DATABASE_URL`
- **Capability:** 11
- **Severity:** Medium
- **Evidence:** `availability-assertion-concurrency.postgres.spec.ts:9-10`, `availability-lock-order.postgres.spec.ts` (via harness `describePostgres`) skip unless the env var is set.
- **Impact:** Concurrency/lock-order guarantees are unproven in default runs; regressions could ship silently.
- **Acceptance criteria:** Integrity specs run in an environment that provides the test DB (CI job or documented local requirement), with results recorded.

## RG-14 — `DepositPolicyService` has no production consumer
- **Capability:** 7
- **Severity:** Medium
- **Evidence:** `deposit-policy.service.ts` imported only by `reservations/__tests__/reservation-status.spec.ts` and `reservation-integration.spec.ts`; registered/exported in `reservations.module.ts` but not injected anywhere.
- **Impact:** "Single rule for deposit requirement" is not enforced; deposit decisions remain ad-hoc.
- **Acceptance criteria:** Create/guarantee paths consult `DepositPolicyService`, or the service is removed and the rule documented where it actually lives.

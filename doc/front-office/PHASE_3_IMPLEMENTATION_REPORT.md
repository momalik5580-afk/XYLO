# XYLO Front Office — Phase 3 Implementation Report

- **Date:** 2026-10-18
- **Work package:** Front Office **Phase 3** — implementation, integration, testing and closure of the operational workflows: walk-in (ADR-073), room moves ↔ check-in flow assignment sync (FO-P2-007 follow-through), no-show desk flow, and the in-scope recon fixes (arrivals KPI accuracy, wake-up calls property scoping). Ends with the Phase 3 exit-gate assessment below. **Phase 4 not started.**
- **Inputs:** `docs/front-office/PHASE_1_*` / `PHASE_2_*` (incl. `PHASE_2_REMEDIATION_REPORT.md` Parts I–II), `docs/audit/Reservations/07_DECISIONS.md` (ADR-073 walk-in authority), the Phase 3 directive, and the Phase 2 handoff (§17 of the Phase 2 report).
- **State:** working tree, `HEAD = ed278bc` (unchanged; no fetch/pull/restore/reset/checkout/stash/commit).
- **Status vocabulary (mandated):** `CLOSED` · `IMPLEMENTED — VERIFICATION PENDING` / `IMPLEMENTED — IMPLEMENTATION VERIFIED (unit); RUNTIME/DB EFFECT NOT VERIFIED` · `BLOCKED BY EXPLICIT DECISION` · `FAILED`. Runtime/DB checks are never marked passed unless actually executed; none were executed here (no live server/DB, no mutation).
- **Directory note:** the authoritative directory is `docs/front-office/` (as established in Phase 2; `doc/front-office/` does not exist). No Phase 3 plan document existed at entry; the requirement list below is derived from the Phase 3 directive and the Phase 2 handoff, and each item maps to implementation + evidence.

---

## 1. Scope and method

- Implementation-first closure: inspect → implement → test → fix → re-verify, on the operational workflow surfaces only:
  1. **Walk-in** — resolve the ADR-073 semantic gap end-to-end (backend command, canonical reservation create, staged check-in pipeline, honest response contract; frontend contract, idempotency, desk UX).
  2. **Room moves (transfer / upgrade)** — keep the authoritative check-in-flow assignment store in sync inside the same move transaction.
  3. **No-show** — desk-side confirmation and honest success/failure feedback on the canonical `POST /reservations/:id/no-show` route (route contract was fixed in Phase 2 / FO-P2-002; not re-implemented).
  4. **Recon fixes** (in-scope, cheap, correctness/security): dashboard arrivals KPI over-count; `GetWakeUpCalls` cross-tenant read.
- **Not done / not reopened, per the Phase 3 boundary:** Hotel ID / property-isolation model and Gates A–D were **not reopened** (re-run, not redesigned); no OTA/Channel-Manager work; no Phase 4 UI redesign; no schema change, no migration, no seeder execution, no DB mutation (target `xylo_cloud.public` only if ever authorized — nothing here needed it).
- All changes use the existing architecture: custom CQRS `CommandBus`, the existing staged check-in pipeline commands, `PrismaService` repositories, `IdempotentCommand` idempotency infra, React Query hooks. No duplicate models, no competing state machines.
- Tests ran **sequentially, single-worker** (`--runInBand`); API suites used the pre-existing read-only `apps/api/.jest.lowmem.json` (host memory constraint carried from Phase 2; the full API suite is not attemptable here).

---

## 2. Requirement-by-requirement disposition

### 2.1 Walk-in (ADR-073) — **IMPLEMENTED — IMPLEMENTATION VERIFIED (unit); RUNTIME/DB EFFECT NOT VERIFIED**

**Semantic resolution (the ADR-073 ambiguity):** walk-in is a Front Office entry point, not a bare reservation insert. The reservation is created through the canonical `ReservationsService.create` (→ `CreateReservationCommand`); the established staged check-in pipeline runs only when the request names a physical room.

**Backend implementation** (`sub-resources.handlers.ts` — `WalkInCommandHandler`, rewritten; `sub-resources.commands.ts:130` — now `IdempotentCommand`; `front-office.service.ts:138-140` — call-site):

- **Validation before any write:** `arrivalDate`, `departureDate`, `roomType` required → `400 VALIDATION_ERROR`, nothing created.
- **Canonical create with full mapping:** dates/occupancy, `rate_code` default `BAR`, `roomRate ?? rate` → `room_rate`, `deposit` → legacy folio payment (`CREDIT_CARD`) + folio header, property-configured walk-in source/market defaults (`WalkInDefaultsService`; explicit request values win), `guarantee_type` default `WALK`, `payment_status`/`payment_source` (default `FRONT_DESK`), `reservation_status: 'CONFIRMED'`, special requests → `reservation_special_requests`, profile fields incl. nationality → guest. A create failure fails the whole command (`WALK_IN_CREATE_FAILED` if no identifier).
- **Staged pipeline (room given):** property-scoped room lookup → `AssignRoomCommand` → `VerifyCheckInPaymentCommand` → `VerifyCheckInComplianceCommand` (accepts `documentType`/`documentNumber`, plus `idType`/`idNumber` aliases, `issuingCountry`) → `FinalizeCheckInCommand` (`initiatedChannel: 'WALK_IN'`). Each stage is dispatched through the `CommandBus` (bounded context intact), stops the chain on failure, and reports a pending step.
- **Honest response contract (what the phase adds):**
  ```
  { success: true, checkedIn, reservationId, roomNumber, checkedInAt,
    reservation, pendingSteps: [{ step, code, message }], walkInDefaults }
  ```
  `checkedIn: true` only after Finalize committed. `checkedIn: false` means **"reservation created, check-in incomplete"**; `pendingSteps` carries the exact blocker: `ROOM_NOT_SELECTED` (no room given), `ROOM_NOT_FOUND`, `ROOM_NOT_ASSIGNABLE`, `PAYMENT_NOT_VERIFIED`, `COMPLIANCE_NOT_VERIFIED`, `FINALIZE_FAILED`, or the surfaced business-rule code (see §3.1). Stage-specific codes are normalized: the `BUSINESS_RULE_` error-class prefix is stripped so the pending-step vocabulary is bare rule codes (e.g. `PAYMENT_DECLINED`).
- **Transaction semantics:** the reservation create runs inside the nested `CreateReservationCommand` dispatch's unit of work (the repository create joins that transaction); each pipeline stage command likewise owns its transaction and commits sequentially — a failure after creation leaves the reservation (CONFIRMED) and any prior completed stages committed, exactly the outcome the response reports. The desk finishes the checklist from the check-in workspace.
- **Idempotency:** `WalkInCommand extends IdempotentCommand`; an explicit key wins, otherwise the HTTP `Idempotency-Key` is inherited from the request context (global `IdempotencyPipe`) — one durable identity per logical walk-in, replay-safe. Pinned by the spec's identity section (inheritance + explicit-key short-circuit).

**Frontend implementation (web):**

| File | Change |
|---|---|
| `mutations/mutation.types.ts` | `WalkInRequest` + `idType`/`idNumber`/`issuingCountry`/`paymentMethod`; new `WalkInPendingStep`, `WalkInResult` (`checkedIn: boolean`, `checkedInAt`, `pendingSteps`). |
| `api/front-office.api.ts` | `walkIn(data, options?)` → `WalkInResult` (carries the idempotency header). |
| `hooks/use-front-office-mutations.ts` | `useWalkInMutation` mints one `Idempotency-Key` per logical walk-in (`idempotencyOnMutate` + `idempotencyRequest`, mirroring checkout); invalidates room-grid/arrivals/dashboard. |
| `validation/walk-in.schema.ts` | ID type / ID number / issuing country form fields. |
| `components/dialogs/WalkInDialog.tsx` | ID fields (Passport / National ID / Driver's License / Other — the vocabularies used by the reservation surface and accepted by manual verification); **honest toast branching**: success only when `checkedIn`, otherwise a `warning` listing the pending-step messages; room-number placeholder corrected (“Leave blank to save without check-in” — it previously claimed auto-assignment). |
| `components/dashboard/ArrivalPanel.tsx` | No-show now goes through a confirm dialog and reports success/failure via toast (previously a silent fire-and-forget mutation). |

**Evidence:** `walk-in.handler.spec.ts` — **13 tests** (canonical create + full stage order; alias acceptance; incomplete-check-in honesty on payment/compliance/assignment failures; no-room → `ROOM_NOT_SELECTED` with no stages; `ROOM_NOT_FOUND`; create-failure fails the command; validation-before-write; property-context rejection; identity inheritance). Frontend: web typecheck clean; no front-office lint findings; `no-show-canonical-route.test.ts` still green (canonical route untouched).

**Limits:** the pipeline SQL (`rooms.findFirst`, flow-store writes, folio payment on deposit, finalize) and the HTTP contract are unit-proven only — no live DB/HTTP (see §4).

---

### 2.2 Room moves — check-in flow assignment sync (FO-P2-007 follow-through) — **IMPLEMENTED — IMPLEMENTATION VERIFIED (unit); RUNTIME/DB EFFECT NOT VERIFIED**

**Problem:** transfer/upgrade updated `reservations.room_number` and room statuses, but NOT `reservation_check_in_flow.assigned_room_id/number` — the authoritative assignment store that FO-P2-007's overlap probe reads. After a move, the probe could still see the vacated room as claimed (and the destination as free).

**Implementation:**

- New `ICheckInFlowRepository.updateAssignment(reservationId, hotelId, roomId, roomNumber, tx?)` + implementation (`check-in-flow.repository.ts:114-135`): `INSERT … ON CONFLICT (reservation_id) DO UPDATE` touching **assignment columns only** (+ `updated_at`). An in-progress/finalized flow keeps every other column; a legacy (pre-pipeline) check-in gains a minimal claim row. `room_status_before_assignment` is deliberately never rewritten — the occupancy snapshot belongs to the original assignment.
- `transfer-room.handler.ts:71-77` and `upgrade-room.handler.ts:160-166` call it **inside the same `withTransaction`** as the `reservations.room_number` write and the room-status flips (upgrade: only on the physical-room path, when a target room was resolved).
- `check-in.module.ts:77` exports `CHECK_IN_FLOW_REPOSITORY` so the commands module can inject it.

**Deliberately NOT done (documented, §4.3):** extending the FO-P2-007 overlap probe with a legacy `reservations.room_number` arm. The flow store remains the single assignment truth; the legacy column is a migration-phase concern (the check-in room-options prefilter already protects the interactive path).

**Evidence:** `transfer-room.handler.spec.ts` + `upgrade-room.handler.spec.ts` — **2 suites / 18 tests**, including assertions that `updateAssignment` is invoked with the destination room inside the move transaction. Suites were updated for the new constructor dependency — a deliberate, minimal wiring change, not a weakened contract.

---

### 2.3 No-show — **IMPLEMENTED (UX hardening); canonical route carried from Phase 2 (`FIXED AND VERIFIED` — not re-implemented)**

- The canonical owner remains `POST /reservations/:id/no-show` (`ProcessNoShowCommand`); `frontOfficeApi.markNoShow` uses it (Phase 2 / FO-P2-002, pinned by `no-show-canonical-route.test.ts` — still green). No second state machine, no DTO weakening.
- Phase 3 adds the missing desk-side UX: confirm before the irreversible action, success/error toasts, per-panel pending state (`ArrivalPanel.tsx`).

---

### 2.4 Recon fix — dashboard arrivals KPI over-count — **IMPLEMENTED — IMPLEMENTATION VERIFIED (unit); DB EFFECT NOT VERIFIED**

- **Defect:** `GetDashboardQueryHandler` counted *all* reservations arriving today — including `CANCELLED` / `NO_SHOW` (and already checked-in/out) rows — while the Arrivals list shows only actionable arrivals. The KPI could exceed the list it links to.
- **Fix** (`front-office.query.handlers.ts`): the arrivals count now mirrors the Arrivals list semantics — `arrival_date` in today ∧ `reservation_status NOT IN ('CHECKED_IN','IN_HOUSE','CHECKED_OUT','CANCELLED','NO_SHOW','NO-SHOW')`.
- **Evidence:** `front-office-query-scoping.spec.ts` — asserts the exact status exclusion set on the arrivals count and the KPI plumbing.

---

### 2.5 Recon fix — `GetWakeUpCalls` cross-tenant read — **IMPLEMENTED — IMPLEMENTATION VERIFIED (unit); DB EFFECT NOT VERIFIED**

- **Defect:** the raw SQL listed `wake_up_calls` with no property filter — any caller received every tenant's wake-up calls.
- **Fix:** `WHERE hotel_id = $1` with the ambient property id; no property context → empty list, no query issued. (The INSERT path was already scoped.)
- **Evidence:** `front-office-query-scoping.spec.ts` — asserts the `hotel_id = $1` predicate + parameter, and the no-context short circuit (no query executed).

---

### 2.6 Arrivals / check-in, in-house rooms, departures / checkout — **carried; verified by re-run, not reopened**

Per the Phase 3 boundary, the Phase 2 four-gate results (Gates A–D: authorization, seeder mapping, checkout transaction/idempotency safety, single server-side checkout Folio contract) stand and were **not** re-opened or re-designed. This phase's integration changes touch them only additively:

- The walk-in entry point now converges on the canonical reservation create and the established check-in stage commands (2.1).
- Room moves (transfer/upgrade) keep `reservations.room_number` and the flow store consistent in one transaction (2.2).
- The full Front Office API module and the cross-module Phase 2 suites were re-run green on the final code (see §3), including `check-out.handler.spec.ts` (Gate C) and `checkout-folio-contract.spec.ts` (Gate D).
- The checkout read contract (`GET /front-office/checkout-folio/:reservationId` → `CheckoutFolioReadService`; legacy family wins, cashiering fallback, never summed, `balance: null` never fabricated `0`) is untouched by Phase 3.

---

## 3. Verification log (final code — exact runs, exact results)

All runs `--runInBand`, sequential, no other process running concurrently. API runs used `apps/api/.jest.lowmem.json`.

| # | Command (working dir) | Result |
|---|---|---|
| 1 | `npx jest --config .jest.lowmem.json --runInBand modules/front-office/application/commands/sub-resources/__tests__/walk-in.handler.spec.ts` (`apps/api`) | **PASS — 13/13** (after §3.1) |
| 2 | `npx jest --config .jest.lowmem.json --runInBand modules/front-office/application/commands/transfer-room modules/front-office/application/commands/upgrade-room` (`apps/api`) | **PASS — 2 suites / 18 tests** |
| 3 | `npx jest --config .jest.lowmem.json --runInBand modules/front-office` (`apps/api`, full module sweep on final code) | **PASS — 30 suites / 336 tests**, 1 suite + 2 tests skipped (the Postgres-gated `check-in-availability.postgres.spec.ts`; needs `AVAILABILITY_TEST_DATABASE_URL`), 0 failures |
| 4 | `npx jest --config .jest.lowmem.json --runInBand modules/front-office/application/queries/__tests__/front-office-query-scoping.spec.ts` (`apps/api`) | **PASS — 4/4** |
| 5 | `npx jest --config .jest.lowmem.json --runInBand --testPathPattern "idempotency-pipe\|idempotent-command-adoption\|permission-guard-hydration\|front-office-permission-seed\|permission-registry-transfer-seed\|checkout-folio-contract"` (`apps/api`, cross-module Phase 2 infrastructure on final code) | **PASS — 6 suites / 60 tests** |
| 6 | `pnpm typecheck` (`apps/api`) | **PASS (clean)** — after §3.2 |
| 7 | `pnpm typecheck` (`apps/web`) | **PASS (clean)** |
| 8 | `pnpm lint` (`apps/web`) | **0 errors / 43 pre-existing warnings**; none in any `front-office` file |
| 9 | `npx jest front-office --silent` (`apps/web`) | **PASS — 9 suites / 26 tests** |

### 3.1 Verification caught a real defect (fixed in the handler, not the test)

The first walk-in spec run was **10/13**: three failure-path tests showed the pending-step `code` leaking the error-class prefix (`BUSINESS_RULE_PAYMENT_DECLINED` etc.). `describeStageFailure` was corrected to strip the `BUSINESS_RULE_` prefix (bare rule code in the response vocabulary); final run **13/13**. The UI contract is the bare code.

### 3.2 Verification caught a typing defect (fixed in the handler)

`pnpm typecheck` (API) surfaced 6 × `TS2339` on the five awaited stage results: the check-in stage command classes extend bare `Command` (default generic `void`), so type inference cannot recover the payload type at the bus boundary. Fixed with explicit `any` annotations on the awaited results in the walk-in handler (consistent with the handler's dynamic-data pipeline and the `CommandHandler<…, any>` declarations; no shared file changed). Typecheck clean afterwards.

### 3.3 Test-proof scopes (what “green” does and does not mean)

- Green unit suites prove handler/service/command semantics against mocks/fakes; they do **not** prove Postgres, HTTP, Redis, or runtime behavior.
- The walk-in honest-response contract is proven by unit dispatch ordering and response-shape assertions, not by a live end-to-end run.
- `updateAssignment` is proven to be invoked with the right parameters inside the move transaction; the `INSERT … ON CONFLICT` SQL itself ran only against fakes/assertions.

---

## 4. NOT VERIFIED / deferred items (explicit)

### 4.1 Not verified in this environment

| Item | Status | Reason |
|---|---|---|
| Walk-in full pipeline against Postgres (room lookup, flow writes, deposit→folio payment, finalize; per-stage commit semantics) | `NOT VERIFIED` (runtime/DB) | no live server/DB; no DB mutation permitted |
| Transfer/upgrade `INSERT … ON CONFLICT` against real Postgres (assignment sync) | `NOT VERIFIED` (runtime/DB) | unit-proven only |
| HTTP contract of walk-in (status codes, Idempotency-Key replay/409) | `NOT VERIFIED` (runtime) | no live HTTP; pipe semantics pinned by existing idempotency specs (re-run green) |
| Frontend walk-in toast branching / no-show confirm at runtime | `NOT VERIFIED` (runtime) | no render test for `WalkInDialog`; web typecheck + existing suites green |
| Arrivals KPI / wake-up scoping SQL against real data | `NOT VERIFIED` (DB) | unit assertions only |
| Postgres-gated availability spec (`check-in-availability.postgres.spec.ts`) | skipped | needs `AVAILABILITY_TEST_DATABASE_URL` (carried) |
| Full API suite / full web suite / Playwright E2E | `NOT VERIFIED` | host memory limit (carried); no live servers |
| Seeder/boot effects (FRONT_DESK grants) | `NOT VERIFIED` | carried from Phase 2; procedure in Phase 2 §12.3 |
| `apps/mobile` FO usage | `UNKNOWN` | carried; not scanned |

### 4.2 Carried Phase 2 residuals (unchanged)

- **FO-P2-008** checkout settlement/AR policy — `BLOCKED BY EXPLICIT DECISION`.
- **FO-P2-004** physical-room upgrade policy (target must match category? OOO eligibility?) — `BLOCKED BY EXPLICIT DECISION`; Phase 3 does not touch the policy.
- **FO-P1-002** `@PropertyScope(false)` intent — `BLOCKED BY EXPLICIT DECISION`.
- reg-card dialog build (dependency for the `reg-card` operation kind); `PermissionCacheService` cache key; cashiering read-handler scoping; legacy `getFolio` error swallowing — all carried as documented in the Phase 2 report.

### 4.3 New residuals documented by Phase 3 recon (not fixed — out of scope, with rationale)

1. **Room grid does not consult the flow store.** `GetRoomGridQueryHandler` derives assignment from `reservations.room_number` (+ a room-type allocation fallback for legacy rows) and never reads `reservation_check_in_flow.assigned_room_id`. With 2.2 the two stores now move together, so the interactive path is consistent; the divergence window is legacy/migration-era rows → **Phase 11 (legacy migration)** target.
2. **Grid status mapping: legacy `IN_HOUSE` renders as `RESERVED`.** `front-office.query.handlers.ts` maps only `CHECKED_IN` → `OCCUPIED`, everything else active → `RESERVED`; `IN_HOUSE` (a valid in-house status elsewhere) therefore shows as a reservation on the rack. Cosmetic/read-model; deferred (touching the status vocabulary belongs with the migration sweep).
3. **Room-options prefilter uses the legacy column.** The check-in room-options probe reads `reservations.room_number`, while `AssignRoom` consults the flow store; the probe is only a UI prefilter and the authoritative overlap probe guards the assignment (FO-P2-007). Deliberately **not** extended with a legacy arm — documented as a **Phase 11 migration gap**.
4. **`rooms.front_office_status` is never written** by any FO flow, so any readiness gate based on it is effectively constant. Fixing it means defining the FO↔housekeeping status contract — deferred.
5. **Room-moves dashboard counters are structurally equal.** `room-moves-dashboard.service.ts:63-64` sets `moveIns` and `moveOuts` to the same `pendingMoves.length`; the per-room-type counts do differ. KPI-semantics cleanup; deferred.
6. **`CompleteWakeUpCallCommandHandler` raw SQL is not hotel-scoped.** The list read was fixed (§2.5); the complete-path `UPDATE … WHERE id = $1` / `SELECT … WHERE id = $1` remain unscoped (UUID-keyed entry point, same defect class). Deferred — needs a decision on not-found semantics; recorded here so it is not lost.

---

## 5. Files changed by this work package

**API** (all under `apps/api/src/modules/front-office/`)

| File | Purpose |
|---|---|
| `application/commands/sub-resources/sub-resources.handlers.ts` | Walk-in handler rewrite: canonical create, staged pipeline, honest `checkedIn`/`pendingSteps` contract, `BUSINESS_RULE_` code normalization. |
| `application/commands/sub-resources/sub-resources.commands.ts` | `WalkInCommand` → `IdempotentCommand` (request-context key inheritance). |
| `application/commands/sub-resources/__tests__/walk-in.handler.spec.ts` | **New** — 13 tests (pipeline order, honesty, failures, validation, identity). |
| `check-in/domain/repositories/check-in-flow.repository.interface.ts` | `updateAssignment` contract. |
| `check-in/infrastructure/check-in-flow.repository.ts` | `updateAssignment` implementation (`INSERT … ON CONFLICT`, assignment columns only). |
| `check-in/check-in.module.ts` | Export `CHECK_IN_FLOW_REPOSITORY`. |
| `application/commands/transfer-room/transfer-room.handler.ts` (+ its spec) | Flow-store assignment sync inside the move transaction. |
| `application/commands/upgrade-room/upgrade-room.handler.ts` (+ its spec) | Same, physical-room path only. |
| `application/queries/front-office.query.handlers.ts` | Arrivals KPI terminal-status exclusion; wake-up list `hotel_id = $1` scoping + no-context guard. |
| `application/queries/__tests__/front-office-query-scoping.spec.ts` | **New** — 4 regression tests for both fixes. |
| `front-office.service.ts` | Walk-in call-site (unchanged contract, threaded command). |

**Web** (all under `apps/web/features/front-office/`)

| File | Purpose |
|---|---|
| `mutations/mutation.types.ts` | `WalkInRequest` ID fields; `WalkInPendingStep`; `WalkInResult`. |
| `api/front-office.api.ts` | `walkIn(data, options?)` → `WalkInResult`. |
| `hooks/use-front-office-mutations.ts` | Walk-in idempotency minting; invalidation set. |
| `validation/walk-in.schema.ts` | ID fields. |
| `components/dialogs/WalkInDialog.tsx` | ID fields; honest toast branching; placeholder fix. |
| `components/dashboard/ArrivalPanel.tsx` | No-show confirm + toasts. |

**Docs**

- `docs/front-office/PHASE_3_IMPLEMENTATION_REPORT.md` — this report (new).

---

## 6. Phase 3 exit gate — 11 items

| # | Gate item | Status | Evidence |
|---|---|---|---|
| 1 | Walk-in resolves ADR-073: canonical create + staged pipeline + honest `checkedIn`/`pendingSteps` contract; failure paths never claim success | `IMPLEMENTED — IMPLEMENTATION VERIFIED (unit); RUNTIME/DB EFFECT NOT VERIFIED` | §2.1; 13/13 spec |
| 2 | Walk-in idempotent (one durable identity per logical attempt; context-key inheritance) | same | §2.1; identity tests + `idempotent-command-adoption` re-run |
| 3 | Room moves sync the authoritative assignment store inside the move transaction; no other column corrupted | same | §2.2; 18/18 + assertion updates |
| 4 | No-show desk flow uses the canonical route; confirm + honest feedback; no DTO/state-machine change | `IMPLEMENTED (source + route test); RUNTIME NOT VERIFIED` | §2.3; `no-show-canonical-route.test.ts` green |
| 5 | Arrivals KPI aligned with the Arrivals list (no CANCELLED/NO_SHOW inflation) | `IMPLEMENTED — IMPLEMENTATION VERIFIED (unit); DB EFFECT NOT VERIFIED` | §2.4; 4/4 spec |
| 6 | Wake-up calls property-scoped (cross-tenant read closed; no context → empty) | same | §2.5; 4/4 spec |
| 7 | Gates A–D not reopened; FO module suites re-run green on final code, incl. Gate C/D suites | `MET` | §3 rows 3/5 (30 suites/336 + 6 suites/60) |
| 8 | Frontend consumes the honest contract (no unconditional “checked in” claim); typecheck + lint clean | `MET (implementation)`; dialog runtime `NOT VERIFIED` | §2.1 web table; §3 rows 7/8/9 |
| 9 | No schema change, no migration, no DB mutation, no seeder execution, no git state change | `MET` | §1; `HEAD = ed278bc`; no `pnpm db:*` run |
| 10 | NOT-VERIFIED items disclosed; deferred residuals enumerated with owners/targets | `MET` | §4 (incl. 4.3 new residuals) |
| 11 | Report delivered at the authoritative path; **Phase 4 NOT started** | `MET` | This document; §7 |

**Exit-gate assessment: PASS at implementation level.** Every Phase 3 requirement maps to implementation + executed test evidence; the runtime/DB checks that could not be executed in this environment (no live server/DB; no mutation authorized) are explicitly `NOT VERIFIED` (§4.1), and the deferred recon residuals are enumerated (§4.3) rather than silently closed.

---

## 7. Handoff (before Phase 4)

1. **Runtime/DB verification of §4.1** on a host with Postgres (authorized DB only; `xylo_cloud.public`, never `xylo_test`): walk-in happy + failing-stage paths via HTTP with `Idempotency-Key`, transfer/upgrade assignment-sync read-back, KPI vs list equality, wake-up scoping probe.
2. **Convert the Postgres-gated availability spec** (`AVAILABILITY_TEST_DATABASE_URL`) from skipped to green.
3. **Decide the carried policy items** (§4.2: FO-P2-008, FO-P2-004, FO-P1-002) before touching those behaviors.
4. **Schedule the §4.3 recon residuals** — grid/flow-store convergence and the `IN_HOUSE` mapping for the Phase 11 migration; `front_office_status` contract; room-moves counters; wake-up complete-path scoping.
5. Uncommitted changes: review/commit the §5 file set together with the pre-existing dirty tree owned by other streams. No git state operation was performed.

*End of Phase 3 implementation report. Phase 3 closed at implementation level; Phase 4 not started.*

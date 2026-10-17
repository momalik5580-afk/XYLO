# XYLO Front Office — Phase 2 Baseline & Exit Gate

- **Date:** 2026-10-17
- **Phase:** Phase 2 — Reservations ↔ Front Office Integration Audit (audit-only)
- **Companion documents:** `docs/front-office/PHASE_2_RESERVATIONS_INTEGRATION_AUDIT.md`, `docs/front-office/PHASE_2_INTEGRATION_GAP_REGISTER.md`, `docs/front-office/PHASE_2_INTEGRATION_MAP.md`

---

## 1. Commands Executed and Actual Results

All commands were read-only or scoped test-only. No installs, migrations, schema changes, DB writes, or configuration changes were made. The host is memory-constrained (~4 GB); commands were run **sequentially**, single-worker (`--runInBand`), and scoped to individual test files.

| # | Command (working dir) | Purpose | Actual result |
|---|---|---|---|
| 1 | `npx jest "features/reservations/model/__tests__/reservation-actions.test.ts" --runInBand --silent` (`apps/web`) | Verify the Reservations action engine / check-in hand-off href contract | **PASS** — `1 suite passed`; `15 tests passed`; 9.919 s |
| 2 | `npx jest "modules/front-office/__tests__/command-ownership.spec.ts" --runInBand --silent` (`apps/api`, default ts-jest config) | Verify single-writer delegation for extend/reinstate | **FAILED TO RUN — heap OOM.** `FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory` after ~102 s; exit code 1. No assertion failure. |
| 3 | `npx jest --config .jest.lowmem.json --runInBand --testPathPattern "modules/front-office/__tests__/command-ownership.spec.ts" --silent` (`apps/api`) | Same, using the in-repo low-memory Jest config (`isolatedModules`) | **PASS** — `1 suite passed`; `8 tests passed`; 5.021 s |
| 4 | `npx jest --config .jest.lowmem.json --runInBand --testPathPattern "modules/front-office/__tests__/fo-route-permissions.spec.ts" --silent` (`apps/api`) | Verify FO `@Permission` enforcement + property scope | **PASS** — `1 suite passed`; `17 tests passed` |
| 5 | `npx jest --config .jest.lowmem.json --runInBand --testPathPattern "(party-commands\|check-out.handler)" --silent` (`apps/api`) | Verify parties query semantics and check-out handler unit behavior | **PASS** — `2 suites passed`; `52 tests passed` |
| 6 | Read-only source inspection (`read`/`grep`/`glob`) | Trace call chains, contracts, guards | Completed — evidence cited throughout |
| 7 | `git status --short`, `git rev-parse --short HEAD` | Prohibited-change check | See §5 |

### 1.1 Note on `apps/api/.jest.lowmem.json`

This file is **pre-existing, untracked project infrastructure** referenced by an earlier audit (`docs/reservations/ota-audit/06_IMPLEMENTATION_STATUS.md:25,31`) as the sanctioned way to run focused API specs on this host. It sets `isolatedModules: true` (skips ts-jest full type-checking, lowering memory). It was used **read-only**; it was **not** created, modified, or committed by this phase.

### 1.2 Commands deliberately NOT run

| Command | Reason |
|---|---|
| `npx jest "modules/front-office"` (full suite, default config) | Known heap OOM on this host (Phase 1 and this phase); would not complete. |
| `pnpm typecheck` / `pnpm build` / `turbo run …` | Whole-repo compile is memory-heavy; host constrained. |
| `npx playwright test` | Requires live app servers; audit-only, resource-constrained. |
| Any `pnpm db:*`, migration, or DB query | Prohibited by task rules (no DB mutations); no read-only DB target was required for the findings. |
| API `lint` | No-op by design (`apps/api/package.json` → `echo 'ok'`). |

---

## 2. Tests Passed / Failed / Skipped / Blocked

### 2.1 Passed (executed in Phase 2)

| Suite | Tests | Result |
|---|---|---|
| `apps/web/features/reservations/model/__tests__/reservation-actions.test.ts` | 15 | PASS |
| `apps/api/src/modules/front-office/__tests__/command-ownership.spec.ts` (lowmem config) | 8 | PASS |
| `apps/api/src/modules/front-office/__tests__/fo-route-permissions.spec.ts` (lowmem config) | 17 | PASS |
| `apps/api/src/modules/reservations/__tests__/party-commands.spec.ts` (lowmem config) | (part of 52) | PASS |
| `apps/api/src/modules/front-office/application/commands/check-out/__tests__/check-out.handler.spec.ts` (lowmem config) | (part of 52) | PASS |

### 2.2 Failed

- None. Command #2 did not fail on an assertion; it was killed by a V8 heap OOM before completing.

### 2.3 Blocked / not verified

| Item | Status | Reason |
|---|---|---|
| Full backend FO suite (default config) | **NOT VERIFIED** | Heap OOM on the ~4 GB host, even with `--runInBand` (§1 row 2). This confirms and extends the Phase 1 limitation. |
| `check-in-availability.postgres.spec.ts` (F-14 invariant) | **NOT VERIFIED (skipped)** | Gated by `describePostgres`; requires `AVAILABILITY_TEST_DATABASE_URL` (`docs/front-office/PHASE_1_BASELINE_AND_EXIT_GATE.md` §2.3). |
| Full-repo typecheck/build | **NOT VERIFIED** | Host memory constraint. |
| Playwright E2E (`e2e/kebab-actions.spec.ts`) | **NOT VERIFIED** | Requires live servers; not appropriate for audit-only run. |
| Runtime API behavior (HTTP 400/403 claims) | **NOT VERIFIED at runtime** | Conclusions are source/config-derived. |
| `apps/mobile` FO usage | **UNKNOWN** | Not scanned (carried from Phase 1). |

---

## 3. What the Passing Tests Do and Do Not Prove

- `command-ownership.spec.ts` proves the **delegation shape** (extend → `ExtendStayCommand`; reinstate → `ReinstateReservationCommand` + `reservationRepository.assignRoomNumber`; no `prisma.reservations.update`) and the change-log ordering. It does **not** exercise the real DB, HTTP, or the FO router.
- `fo-route-permissions.spec.ts` proves the **platform `@Permission`** routes enforce the correct reservation permission and property scope (including `REFUSED`). It does **not** cover the common `@Permissions('front_office:*')` routes (the subject of `FO-P2-005`).
- `party-commands.spec.ts` proves the parties handler/service shape; it uses an in-memory fake repo and cannot catch the frontend passing a confirmation number where the SQL query expects a UUID (`FO-P2-001`).
- `check-out.handler.spec.ts` proves the handler's unit behavior with mocks; it contains no settlement-balance gate assertion, so it does not contradict `FO-P2-008`.

**No test in this phase proves or disproves the OOM-blocked full suite.** Do not represent the backend FO baseline as “passing”.

---

## 4. Baseline Limitations

1. **Backend FO full suite remains NOT VERIFIED** on this host (heap OOM, default ts-jest config). Focused lowmem runs are green but are a subset.
2. **No runtime/API execution** — HTTP-status/clientside claims are source/config-derived.
3. **Postgres-gated availability test skipped** without `AVAILABILITY_TEST_DATABASE_URL`.
4. **Working tree is heavily modified** (large pre-existing uncommitted diff). The audited state is the **working tree**, `HEAD = ed278bc`.
5. **Realtime event→invalidation matrix** not exhaustively enumerated.
6. `apps/mobile` not scanned.
7. The Phase 2 frontend test baseline is limited to one pure-logic file; the broader Phase 1 web FO run (2 suites / 8 tests) is carried forward, not re-executed here.

---

## 5. Prohibited-Change Check

- **No application code, tests, schema, migration, DB object, configuration, dependency, or lockfile was modified** by this phase.
- The only files created are the four Phase 2 deliverables under `docs/front-office/`:
  - `PHASE_2_RESERVATIONS_INTEGRATION_AUDIT.md`
  - `PHASE_2_INTEGRATION_GAP_REGISTER.md`
  - `PHASE_2_INTEGRATION_MAP.md`
  - `PHASE_2_BASELINE_AND_EXIT_GATE.md`
- No commits, pushes, pulls, fetches, resets, restores, branch switches, or deploys.
- No DB connection, query, or mutation was performed.
- **Verification method:** `git status --short` before/after; `HEAD` unchanged at `ed278bc`. Pre-existing modifications/untracked files in `apps/**`, `packages/**`, `docs/**` are unrelated to this phase and were left untouched. `apps/api/.jest.lowmem.json` pre-existed and was used read-only.

---

## 6. Evidence Index (Phase 2)

| Claim area | Primary evidence |
|---|---|
| Hand-off href | `apps/web/features/reservations/model/reservation-actions.ts:124-131`; test PASS |
| URL sync | `apps/web/features/front-office/hooks/use-front-office-url-sync.ts:10-24` |
| Dialog host | `apps/web/features/front-office/components/dashboard/OperationDialogHost.tsx:57-67` |
| Guest/shares identifier | `CheckInDialog.tsx:556`; `CheckInGuestList.tsx:73`; `reservation.repository.ts:1458-1463,1468` |
| No-show | `front-office.api.ts:208-210`; `update-reservation.dto.ts:3-18`; `main.ts:28-40`; `reservations.controller.ts:123-126` |
| Check-out ordering | `check-out.handler.ts:117,144,166,189,196-226,235,388,527-530` |
| Idempotency | `CheckoutFlowModal.tsx:130`; `services/api.ts:22-51,109-115`; `idempotency.interceptor.ts:21-24` |
| Upgrade contract | `dto.ts:98-102`; `front-office.service.ts:112-113`; `front-office.api.ts:135-137` |
| Permissions | `common/authorization/permission.guard.ts:14-24`; `jwt.strategy.ts:28-30`; `auth.service.ts:129-140`; `platform/permission/permission.guard.ts:40-54`; `authorization.module.ts:18-22` |
| Availability/overlap | `room-assignable.specification.ts:16-17`; `assign-room.handler.ts:84-88,99-103`; `check-in-availability.postgres.spec.ts:41-49` |
| Folio families | `packages/db/schema.prisma:4354-4355,4390-4393,4414-4425`; `billing.service.ts:53-69,151-156`; `CheckoutFlowModal.tsx:17,324` |
| State machine | `packages/shared/src/reservation-state-machine.ts:32-42`; `reservation-status.service.ts:30-36` |

---

## 7. Phase 2 Exit Criteria (10) and Evidence

| # | Exit criterion | Status | Evidence |
|---|---|---|---|
| 1 | Reservations→FO entry points enumerated and the deep-link contract (`tab` / `dialog` / `reservation`) traced end-to-end | **MET** | Audit §3.1; `reservation-actions.ts:124-131`; `use-front-office-url-sync.ts:10-24`; `OperationDialogHost.tsx:57`; `reservation-actions.test.ts` PASS |
| 2 | Each FO lifecycle operation mapped to its authoritative owner command/handler and persistence writer; single-writer honored or deviation recorded | **MET** | Audit §5.3, §6; Map §3; `command-ownership.spec.ts` PASS; no-show deviation `FO-P2-002` |
| 3 | Identifier integrity verified across the hand-off (UUID vs confirmation number vs legacy composite); mismatches recorded | **MET** | Audit §3.1, §4.2; `FO-P2-001` |
| 4 | API contract alignment (route/method/DTO/permission) verified for every touched FO↔Reservations surface | **MET** | Audit §4; `FO-P2-002`, `FO-P2-004`, `FO-P2-009` |
| 5 | Lifecycle/state-transition consistency verified against the shared state machine; illegal/alternate writers recorded | **MET** | Audit §5; Map §7; `reservation-state-machine.ts:32-42`; `check-out.handler.ts:235` |
| 6 | Availability/inventory linkage verified or explicitly recorded as a by-design bypass with evidence | **MET** | Audit §7; `check-in-availability.postgres.spec.ts:41-49`; `FO-P2-007` |
| 7 | Billing/folio/checkout dependencies traced with authoritative-balance determination recorded | **MET** | Audit §8; Map §4; `FO-P2-003`, `FO-P2-006`, `FO-P2-008` |
| 8 | Permission/property-scope enforcement for integration routes verified and reconciled with the frontend vocabulary | **MET** | Audit §9; Map §5; `fo-route-permissions.spec.ts` PASS; `FO-P2-005`, `FO-P2-009` |
| 9 | Cross-module error/state-sync and realtime invalidation behavior traced or marked unverified | **MET (partial, explicitly marked)** | Audit §10; realtime matrix marked partially verified |
| 10 | Findings carry evidence class + severity; no unsupported claims; prohibited-change check passed; test baseline recorded or NOT VERIFIED | **MET** | Register (13 findings, each evidenced); §5; §1–§4 |

**Exit declaration:** Phase 2 is **COMPLETE** against the 10 criteria, with these recorded limitations: (1) the backend FO **full** suite remains **NOT VERIFIED** on the ~4 GB host (default-config heap OOM; focused low-memory runs passed), (2) the Postgres-gated availability spec is skipped without its env var, (3) no runtime/API calls were made, so HTTP-status/idempotency conclusions are source/config-derived, and (4) the realtime event→invalidation matrix is only partially verified. None of these limitations changes the code-direct findings; they do mean that the **backend full-suite** and **runtime** baselines must be re-established before operational validation.

---

## 8. Recommended Next Step and Finding Routing

**Do not start the next phase as part of this audit.** Recommended sequencing:

1. **Immediate hardening (pre-Phase 3):** fix `FO-P2-002` (no-show) and `FO-P2-001` (guest/shares identifier) — small, high-operational-impact, and directly traceable.
2. **Phase 9 — Frontend Reconstruction:** `FO-P2-001`, `FO-P2-002`, `FO-P2-004`, `FO-P2-009`, `FO-P2-011`, `FO-P2-012`, `FO-P2-013`.
3. **Phase 6 — Inventory/Profile/Billing Integration:** `FO-P2-006`, `FO-P2-007`.
4. **Phase 12 — Production Readiness:** `FO-P2-003` (financial idempotency), `FO-P2-005` (RBAC source), `FO-P2-008` (settlement policy), plus a re-run of the full backend FO suite on a larger host and the Postgres-gated availability test.
5. **Informational (no action required):** `FO-P2-010`.

| Finding | Severity | Recommended owner phase |
|---|---|---|
| `FO-P2-001` | P2 | Phase 9 + immediate fix |
| `FO-P2-002` | P2 | Phase 9 + immediate fix |
| `FO-P2-003` | P2 | Phase 12 |
| `FO-P2-004` | P3 | Phase 9 |
| `FO-P2-005` | P2 | Phase 12 (RBAC regression) |
| `FO-P2-006` | P2 (risk) | Phase 6 / billing convergence |
| `FO-P2-007` | P2 (risk) | Phase 6 |
| `FO-P2-008` | P3 | Phase 12 |
| `FO-P2-009` | P3 | Phase 9 / Phase 12 |
| `FO-P2-010` | Info | — |
| `FO-P2-011` | Info | Phase 9 (optional) |
| `FO-P2-012` | Info | Phase 9 (with `FO-P1-004`) |
| `FO-P2-013` | P3 | Phase 9 (with `FO-P1-006`) |

---

*End of Phase 2 baseline & exit gate.*

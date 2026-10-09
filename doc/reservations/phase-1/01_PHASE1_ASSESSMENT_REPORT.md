# Reservations Phase 1 — Assessment & Foundation Report

**Plan:** XYLO Reservations Improvement Plan (4 phases: 1 Assessment & Foundation · 2 Core Reservations Backend · 3 API & Frontend · 4 Verification & Legacy Cleanup)
**Phase:** 1 — Assessment only (read-only; no code, schema, or migration changes made)
**Date:** 2026-10-09
**Status:** COMPLETE — awaiting owner review of findings and minimum foundation scope before any implementation

**Evidence tags used throughout:** `[V]` = verified directly this session (file read, targeted grep, or executed command); `[R]` = reported by read-only code exploration, spot-check still pending; `[D]` = taken from an existing authoritative document, not re-derived.

---

## 0. Documentation path verification (deliverable precondition)

- The intended location `documents/Reservations/` **does not exist**: repository root has no `documents/` directory, and no file in the repo references a `documents/...` path `[V]`.
- The repository's actual convention for phase work is `docs/<domain>/phase-<n>/NN_NAME.md` (see `docs/availability/phase-4/…phase-6/`, `docs/architecture/01…06_*.md`) and locked Reservations specs live in `docs/audit/Reservations/01…12_*.md` `[V]`.
- **Decision:** this report is created at `docs/reservations/phase-1/01_PHASE1_ASSESSMENT_REPORT.md`, mirroring the availability phase convention. No existing document was moved, duplicated, or overwritten.
- **Non-redundancy check:** `docs/audit/Reservations/08_CURRENT_STATE.md` (locked baseline, 2026-08-26) remains the authoritative current-state reference `[D]`. This report does **not** restate it; it records only the delta observed against the code as of 2026-10-09, plus verified defects, test evidence, and the proposed minimum foundation.

---

## 1. Scope and evidence examined

### 1.1 Documents read (authoritative inputs)

| Document | Use in this report |
|---|---|
| `docs/architecture/03_ARCHITECTURE_FINDINGS_AND_VERDICT.md` (648 lines) | 53-finding register; ARCH-007/008 (tenancy), ARCH-012 (dual reservation/guest identity + **F-05 readiness closure 2026-10-09, GO WITH GUARDRAILS**, lines 206, 574), ARCH-013 (FO→reservations 35 edges), ARCH-016/019/024; preconditions P1–P11 `[V]` |
| `docs/audit/Reservations/07_DECISIONS.md` §ADR-076 (lines 1319–1349) | PATCH canonical / PUT retained / no lifecycle via amendment `[V]` |
| `docs/audit/Reservations/12_API_CONTRACT.md` §3 (line 67), §77–78 (lines 1079–1097) | business-intent verbs, PATCH semantics, PUT constraint `[V]` |
| `docs/audit/Reservations/08_CURRENT_STATE.md` (977 lines) | locked baseline; §43 testing, §55/56 preserve/do-not-preserve, §60 freeze `[V]` |
| `docs/audit/Reservations/04_BACKEND_ARCHITECTURE.md` (headings) | single authority, explicit ownership, explicit transactions, no dual writes `[V]` |
| `docs/enterprise/reservations-roadmap-v2.md` (175 lines) | phase table (2b/6/7 open; 9 partial), port definitions, exit gates `[V]` |
| `AGENTS.md` | quick commands, gotchas, rebuild status `[V]` |

### 1.2 Code examined (read-only)

- Backend: `apps/api/src/modules/reservations/` (api/ application/ domain/ infrastructure/ permissions/ + legacy `reservations.service.ts`, `reservations.module.ts`); `packages/shared/src/reservation-state-machine.ts`; `packages/db/schema.prisma` (model names via exploration); `apps/api/src/modules/front-office/` (check-in/check-out/transfer/upgrade writers).
- Frontend: `apps/web/app/(dashboard)/reservations/**`, `apps/web/features/reservations/**`, `apps/web/components/reservations/**`, `apps/web/store/reservationStore.ts`, `apps/web/features/reservations/api/*`.
- Integrations: availability assertion wiring (`reservation-availability-wiring.ts` + specs), front-office state writes, channels webhook ingress (doc-referenced).

### 1.3 Out of scope (explicit)

No runtime service was booted; no database was contacted; no package was installed; no code, test, schema, or migration was modified; no Phase 2/3/4 work was started; no legacy file was deleted.

---

## 2. Current-state findings (verified)

### 2.1 Reservation identity & guest identity

1. **Authoritative reservation identity is `reservations.id` (UUID); `reservation_name` (composite `resv_name_id + hotel_id`) is a write-through legacy projection and allocator — closed by F-05/ARCH-012 on 2026-10-09 "GO WITH GUARDRAILS"** `[D — 03_ARCHITECTURE_FINDINGS_AND_VERDICT.md:206,574]`.
   - Guardrails (must hold for every subsequent phase): new reservation writes only through `ReservationRepository` / `ReservationPersistenceService` / registered commands; do not rely on `reservation_guests.profileId` semantics, on `reservation_name.reservation_status` synchronization, or on the `guest_profiles` raw-SQL table; three deferred decisions (guest matching/merge, `profileId` meaning, projection status policy) remain gated behind P9 `[D]`.
2. **Seven front-office sites write `reservations` directly** (`check-in-commit.service.ts:80`, `check-out.handler.ts:235`, `transfer-room.handler.ts:49`, `upgrade-room.handler.ts:113`, `change-rate-code.handler.ts:71`, `reverse-check-in.handler.ts:70`, `front-office.service.ts:126`) `[V]`. The two sites inspected include `hotel_id` in the `where` clause `[V]`, so these are **ownership/boundary divergences (ARCH-012/ARCH-013), not verified tenant leaks**. Whether they satisfy the ARCH-012 "writes only through ReservationRepository" guardrail is an open item for Phase 2 review (FO owns check-in/out; Reservations owns lifecycle — both remain in force).

### 2.2 Lifecycle & state transitions

3. **Single enforcement point exists and is used:** `reservation-status.service.ts` (`assertTransition/canTransition/toStorage/normalize`) delegating to `packages/shared/src/reservation-state-machine.ts` (transitions: `CONFIRMED → GUARANTEED|CHECKED_IN|CANCELLED|NO_SHOW|CHECKED_OUT`, etc.) `[V]`.
4. **Storage-form drift is real:** `toStorage()` maps only `NO_SHOW → 'NO-SHOW'`; `STORAGE_STATUS.CHECKED_IN = 'CHECKED_IN'` `[V]`. Characterization test `reservation-integration.spec.ts:86` expects `'CHECKED-IN'` → **fails** `[V-executed]`. Additional drift (`NO_SHOW` vs `NO-SHOW` written by different paths) reported `[R]`.
5. **Status vocabulary split:** backend canonical = `SCREAMING_SNAKE` (`CHECKED_IN`); frontend types/mappings use kebab-case (`checked-in`) `[V]`, and shared `normalizeStatus('checked-in')` **throws** `Unknown reservation status` (`reservation-state-machine.ts:65`) `[V-executed]`. Live production exposure of that throw path is **not** verified.

### 2.3 Controllers / DTOs / handlers / routes

6. **64 routes on `@Controller('reservations')` with class-level `@PropertyScope(true)` and `@Permission(...)` on every route** `[V — decorator scan]`.
7. **`PUT /:id` exists (`reservations.controller.ts:84`); `@Patch` appears nowhere in the module** `[V]`. Under ADR-076 the PATCH route is *authorized* but **not yet implemented**; PUT remains temporarily for compatibility. Not a defect — a pending, pre-approved slice item.
8. **`UpdateReservationDto` whitelist (14 fields) contains no `reservation_status`** `[V]`, and `main.ts:29-31` sets `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true })` `[V]` → any lifecycle field sent to PUT/PATCH is a hard **400** (correct per ADR-076 §3/§4; the *frontend* violates it — see §2.5).
9. **Route-order hazard:** static `GET room-moves` (line 254) and `GET display-sets` (line 512) are declared **after** single-segment `GET :id` (line 69) `[V]`. Express matches in registration order, so these are expected to be shadowed; **runtime confirmation not performed** (no HTTP-level test exists — `supertest`/`INestApplication` count in module = 0 `[R]`).
10. **Runtime crash site:** `reservations.service.ts:588-589` — `const repo = (this as any).repo; return repo.findLinkedReservations(...)`; the class has no `repo` member `[V]` → `GET /reservations/:id/linked` (controller line 480) is expected to throw `TypeError`. Not executed.
11. **CQRS:** 44 commands + 18 queries registered in `reservations.module.ts` (manual `CommandBus.register`) `[R]`; 3 query handlers on disk never wired (`get-email-templates`, `get-package-exclusions`, `get-l2b-config`) `[R]`.

### 2.4 Authorization / property scoping / transactions

12. **Repository scoping is conditional, not fail-closed** `[V]`:
    - `reservation.repository.ts:93` — `const hotelId = getHotelId() || params.propertyId;` (client-supplied `SearchReservationDto.propertyId` can act as tenant scope)
    - `:130` — `if (hotelId) where.hotel_id = hotelId;` (absent ALS + absent propertyId ⇒ **unscoped read**)
    - `:359` — `findById(id, hotelId?)` optional hotel filter; `:876-878` delete path conditional
    This sits under global precondition **P1/ARCH-007** ("tenant scope from authenticated identity; no client-supplied header authoritative") `[D]`, whose verdict remains open at platform level. For Reservations, the repository-level conditional is the actionable, in-boundary part.
13. **Guard stack** (documented, not executed): platform `PermissionGuard` + common `PermissionGuard`/`RolesGuard`/`PropertyAccessGuard` + `PropertyScopeGuard` + `MultiTenantGuard`, ALS filled by `tenant.interceptor` `[R]`. Two permission vocabularies on every request = ARCH-016 (platform-wide, not Reservations-specific) `[D]`.
14. **Transaction mechanism** (ambient unit-of-work + CQRS transaction pipe) is judged SOUND at platform level (ARCH-003/Theme G exceptions are FO money paths) `[D]`.

### 2.5 Frontend (screens, API usage, state)

15. **Live routes:** `/reservations`, `/reservations/[id]`, `quick-book`, `waitlist`, `availability`, `allotments`, `group-blocks`, `profiles`, `client-relations` under `app/(dashboard)/reservations/` `[R]`.
16. **Verified contract breaks (backend will reject these requests):**
    - **No-show via PUT** — `features/reservations/api/reservation.api.ts:233` and `store/reservationStore.ts:385` send `PUT /reservations/:id { reservation_status: 'NO-SHOW' }` → DTO whitelist ⇒ **400** `[V]`. Canonical route `POST :id/no-show` exists (controller line 98) `[V]`.
    - **Share payee 404** — `features/reservations/api/shares-payee.api.ts:22,29` call `/reservations/:id/select-share-payee` and `/unselect-share-payee`; backend exposes `POST :id/shares/:gid/select-payee|unselect-payee` (lines 323, 335) `[V]`. **Reachable from the live detail workspace**: `SharesPayeePanel` ← `BillingPanel.tsx:8,124` ← `ReservationDetailWorkspace` `[V]` (subject to `shares.enabled` gating).
    - **Linked reservations 404** — `linked-reservations.api.ts:45` POSTs `/reservations/:id/link`; backend is `POST :id/linked` (line 487) `[V]`; its panel appears unmounted today (latent) `[R]`.
    - Reported, spot-check pending `[R]`: `ChangeRateDialog` sends `nightly_rate/total_rate` instead of `PUT :id/rate`; `MassUpdateDialog` status casing (`checked-in`) rejected by repository status validation; `ReservationsHomeView.handleAction` missing `noShow/reinstate/guarantee/modify` cases; detail workspace `noShow` opens the Cancellation dialog; `/reservations/new` and `/reservations/import` links 404.
17. **State ownership:** correct pattern = TanStack Query hooks in `features/reservations/hooks` (query keys `reservationKeys`) + UI-local `reservation-ui-store` `[R]`. Violations: `apps/web/store/reservationStore.ts` holds **server state in Zustand** (reservations/profiles/rateCodes/KPIs fetched via direct `api.*`) and is still mounted through `CreateProfileModal` and `CheckoutFlowModal` `[R]`; two duplicate query-key registries (`lib/query-keys.ts` unused) `[R]`; dual reinstate implementations `[R]`.
18. **Dead/legacy candidates (evidence: zero importers, no deletion performed):** `components/reservations/events/` (23 files), `activities/`, `modals/EditReservationModal|ReservationDetailModal`, `ReservationToolbar`, orphan config panels (9 endpoints that would 404), `ReservationWorkspaceNav` vs `ReservationSidebarNav` duplication `[R]`. ARCH-042 records 82 zero-importer files repo-wide including Reservations Phase-6 port adapters — **classify only, do not delete in Phase 1** `[D]`.

### 2.6 Integrations

19. **Availability:** reservation create/modify paths assert via `application/services/reservation-availability-wiring.ts`; guardrail cases **F-01/F-02/F-03** pinned in `reservation-create-availability.postgres.spec.ts` + `reservation-availability-wiring.spec.ts`, **F-04** in `reservation-modify-dates.postgres.spec.ts:60`, **F-05** in `reservation-room-type-change.postgres.spec.ts:69` `[V]`. These must be preserved untouched.
20. **Front Office:** delegates some operations to Reservations CQRS (`reinstate`, `extend`) but also mutates `reservations` directly (§2.1) `[R/V]`. Handoff contract = `FrontOfficeHandoffPort`, Phase 6 `[D]`.
21. **Channels:** webhook ingress creates through the reservations facade (ARCH-006, SOUND) `[D]`.

---

## 3. Critical risks and blockers (ranked)

| # | Rank | Risk | Evidence | Class |
|---|---|---|---|---|
| **R1** | **P0** | **Reservation reads are not fail-closed on tenant scope.** Conditional `if (hotelId)` + client `propertyId` fallback + optional `findById` hotel filter can yield cross-property reads if ALS is empty. | `reservation.repository.ts:93,130,359` `[V]` under P1/ARCH-007 `[D]` | Verified defect (scoped) |
| **R2** | **P0** | **Verification blindness: known-red tests + broken fixtures.** 11 web tests and 3 API tests fail today; one core fixture (`res()`) silently ignores its overrides, so the action-engine characterization suite is largely meaningless; a stale source-pin (`inventoryDomain`) fails against a cleaned repository. Full API suite baseline unknown. | executed this session §4 `[V]` | Verified |
| **R3** | **P0** | **Frontend breaks core lifecycle operations at the contract level:** no-show → 400 (live: table row actions, FO `markNoShow`); share-payee → 404 (live: detail workspace). Users cannot complete spec-mandated procedures. | §2.16 `[V]` | Verified defect |
| **R4** | **P1** | **Two dead/broken backend endpoints:** `GET :id/linked` crashes (`(this as any).repo`); `GET room-moves` and `GET display-sets` expected shadowed by `GET :id`. | `reservations.service.ts:588`; controller line order `[V]` (runtime effect unconfirmed) | Verified code / inferred runtime |
| **R5** | **P1** | **Status vocabulary + storage-form drift** (kebab vs SCREAMING_SNAKE; `CHECKED_IN` vs `CHECKED-IN`; `NO_SHOW` vs `NO-SHOW`) — red tests today, silent mapping risk across web/api/shared. | §2.4–2.5 `[V]` | Verified |
| **R6** | **P1** | **Bypass of the single-writer guardrail:** 7 FO direct `reservations.update/create` sites vs ARCH-012 "new reservation writes only through ReservationRepository / registered commands". | §2.1 `[V]` | Verified code; guardrail conflict open |
| **R7** | **P2** | **Zustand server-state store + duplicate reinstate/keys/endpoints** — violates "TanStack Query owns server state"; two divergent reinstate paths can disagree operationally. | §2.17 `[R]` | Reported |
| **R8** | **P2** | Dead components/panels/3 unwired handlers and9 config endpoints that would 404 — misleading wiring (ARCH-042). | §2.18 `[R]` | Reported; classify only |

**Not blockers (explicitly):** absence of PATCH (ADR-076 authorizes it in the first amendment slice `[D]`); dual reservation/guest models (F-05 closed with guardrails `[D]`); module-boundary depth (ARCH-013, global, P4) — constraint, not a Reservations Phase-1 fix.

---

## 4. Existing tests and safety coverage

### 4.1 Inventory

- **API:** 223 spec files total; **48 under `modules/reservations`** (~7.2k lines) `[V --listTests]`. Includes party commands, scheduled room moves, flex fields, mass update, availability guardrail pins (§2.19), write-identity (`t552`), delete-never-releases (`t560`).
- **Web:** 21 suites / 166 tests in `pnpm test` `[V-executed]`; reservation-relevant: `features/reservations/model/__tests__/reservation-actions.test.ts` (15 tests), quick-book mapping, availability display, action-engine tests.
- **HTTP/e2e:** none for reservations (no supertest/INestApplication) `[R]`.
- **Postgres-gated suites:** require `AVAILABILITY_TEST_DATABASE_URL` (`availability-postgres.harness.ts:8,116` throws without it) `[V]` — **none executed** (would need a database; prohibited this phase).

### 4.2 Actually executed this session (4 GB constraint: sequential, single worker)

| Command | Result |
|---|---|
| `cd apps/api && pnpm typecheck` | **PASS** (no errors) `[V]` |
| `cd apps/web && pnpm typecheck` | **PASS** (no errors) `[V]` |
| `cd apps/web && pnpm test` (full) | **FAIL** — 11 failed / 155 passed / 166 total, 2 failing suites `[V]` |
| `apps/web` `reservation-actions.test.ts` + `t542-dead-artifacts.test.ts`, `-w 1` | **FAIL** — 11 failed / 14 passed / 25 `[V]` |
| `apps/api` `…/application/services/__tests__/reservation-availability-wiring.spec.ts`, `-w 1` | **PASS** — 20/20 (26.5 s) `[V]` |
| `apps/api` `…/__tests__/reservation-integration.spec.ts`, `-w 1` | **FAIL** — 3 failed / 14 passed / 17 `[V]` |
| `cd apps/api && pnpm test` (full) | **NOT COMPLETED** — timed out at 900 s; **no pass/fail conclusion** `[V]` |

**Diagnosed failures (root cause verified):**
1. `reservation-actions.test.ts` — fixture defect: `res(over)` (line 16) **never spreads `over`** into the returned literal (lines 17–30), so every case runs against a fixed `confirmed / balance: 0` reservation → 10 failures (e.g. pending expects `confirm`, receives `guarantee`). The production model (`reservation-actions.ts:172-263`) reads correct `[V]`. **This is a broken safety net, not (yet) proof of a product defect.**
2. `t542-dead-artifacts.test.ts:112` — stale pin: expects `reservation.repository.ts` to contain `inventoryDomain`; it now contains **0** matches `[V]`.
3. `reservation-integration.spec.ts` — 3 failures: `rejects unknown statuses`, `normalizes lowercase statuses` (throws on `checked-in`), `maps to storage form` (`CHECKED_IN` vs expected `CHECKED-IN`) `[V-executed]` — confirms R5.

### 4.3 Missing safety coverage (gap list)

- HTTP-level tests for route ordering, `@Permission`, `@PropertyScope` (gap that lets R4 exist).
- Tenant-scoping tests: absent ALS ⇒ must fail closed; client `propertyId` must not act as scope (R1).
- Web↔API **contract tests** (request-shape assertions per mutation) — the class of R3 is currently undetectable.
- Lifecycle E2E (create→confirm→amend→cancel→reinstate→no-show) — Phase 4/roadmap exit gate `[D]`.
- Green baseline: all suites must be green or explicitly waived before Phase 2 behavior changes (R2).

### 4.4 Smallest safe test commands (recommended, low-memory)

```bash
cd apps/api
npx jest "modules/reservations/application/services/__tests__/reservation-availability-wiring.spec.ts" -w 1   # 20 tests, PASS
npx jest "modules/reservations/__tests__/reservation-integration.spec.ts" -w 1                                # 17 tests, 3 FAIL (baseline)
cd apps/web
npx jest features/reservations/model/__tests__/reservation-actions.test.ts -w 1                               # 15 tests, 10 FAIL (baseline)
pnpm typecheck   # apps/api and apps/web — PASS
```

Avoid until approved (RAM): full `pnpm test` in `apps/api` (223 files), `pnpm build`, coverage, parallel workers, postgres-gated specs.

---

## 5. Minimum required foundation changes (PROPOSED — not started; requires explicit approval)

Scope rule: smallest changes that remove P0 blockers and make verification trustworthy. No schema/migration change; no refactoring for style; no legacy deletion.

| ID | Change | Files (expected) | Why minimum | Risk control |
|---|---|---|---|---|
| **F1** | Fix `res()` fixture to spread overrides; re-run the 15 tests; fix only genuine product defects they then expose | `apps/web/features/reservations/model/__tests__/reservation-actions.test.ts` (+ model only if a real defect is proven) | Restores the action-engine safety net (R2) | Test-only first; any model change justified per failing case |
| **F2** | Resolve the 3 `reservation-integration.spec.ts` failures: owner decision on canonical storage/status vocabulary (align spec ↔ `reservation-status.service` ↔ shared machine) | spec + possibly `reservation-status.service.ts` / `reservation-state-machine.ts` | R5 must be decided before any lifecycle work | Decision recorded as a decision note; behavior change minimal and covered by the same spec |
| **F3** | Wire frontend no-show to `POST /reservations/:id/no-show`; fix share-payee paths to `:id/shares/:gid/select-payee\|unselect-payee`; fix `/link` → `:id/linked` | `reservation.api.ts`, `store/reservationStore.ts`, `shares-payee.api.ts`, `linked-reservations.api.ts` | Removes live 400/404 on spec-mandated procedures (R3) | Contract-only; no backend change; verify by typecheck + targeted tests |
| **F4** | Fix `getLinkedReservations` (dispatch a registered query/repository instead of `(this as any).repo`) | `reservations.service.ts:585-590` | Removes guaranteed runtime crash (R4) | Single-method change; add a unit test |
| **F5** | Move static GETs (`room-moves`, `display-sets`) above `GET :id`, and add the first HTTP-level route test | `reservations.controller.ts` + new supertest spec | R4 needs proof, not inference | Route test also locks guard/permission behavior |
| **F6** | Make repository tenant scope fail-closed: require `hotelId` (from ALS) for `findAll`/`findById`/delete; stop treating client `propertyId` as tenant scope | `reservation.repository.ts:93,130,359,876` | R1 — the in-boundary half of P1/ARCH-007 | Ship with new scoping tests (positive + cross-property negative); no schema change |

**Explicitly NOT in the minimum foundation:** PATCH route (belongs to the first amendment slice per ADR-076), FO direct-writer convergence (R6 — needs owner ruling against ARCH-012 guardrail), Zustand server-state migration (R7), dead-code removal (R8), status-enum refactors beyond F2, any schema/index/RLS work (roadmap Phase 2b).

---

## 6. Recommended implementation order and acceptance criteria

**Phase 1 exit gate (this report):** owner approves/rejects/amends F1–F6 and the ranking in §3.

**Phase 2 — Core Reservations Backend** (order):
1. **F2 → F1 → F4 → F5** (green baseline + crash/route fixes). *AC:* targeted suites green (§4.4 commands); route test proves `GET room-moves`/`display-sets` reachable; no red reservations suite in `-w 1` runs.
2. **F3 + F6 scoping** with characterization tests. *AC:* no-show works end-to-end (UI → `POST :id/no-show` → state transition asserted); cross-property read attempts rejected when ALS empty/foreign; `pnpm typecheck` green in api+web.
3. **ADR-076 PATCH slice** (same DTO, same validation, no lifecycle fields; PUT retained). *AC:* PATCH and PUT share one code path; a test proves `reservation_status` is rejected on both; contract text unchanged.
4. R5/R6 owner rulings recorded (storage form; FO-writer disposition) before any FO work. *AC:* decision note filed; no behavior change without it.
5. Exit gate per roadmap v2 for Phase 4: each command handler ships concurrency/idempotency/authorization tests `[D]`.

**Phase 3 — API & Frontend:** remaining reported contract breaks (rate dialog, mass-update casing, action-engine cases, missing routes), Zustand server-state migration to Query, single query-key registry, dead-panel classification (not deletion). *AC:* contract tests per mutation; no Zustand store fetching server state for reservations.

**Phase 4 — Verification & Legacy Cleanup:** HTTP/e2e lifecycle sweep (create→confirm→amend→cancel→reinstate→no-show), full suite green under documented commands, RETAIN/DELETE ledger for R8 items with authorization `[D — roadmap exit gates]`.

---

## 7. Explicitly deferred

- **Phase 2:** everything in §5 beyond approved minimum; PATCH slice; unwired handlers; status-storage convergence beyond F2.
- **Phase 3:** frontend contract repairs not listed in F3; state-management migration; route/breadcrumb fixes.
- **Phase 4:** legacy deletion (events/activities/modals/orphan panels, `ReservationPersistenceService`, duplicate nav), e2e + load (p95), full-suite green.
- **Roadmap-tracked, out of all four phases here:** Phase 2b DB integrity hardening, Phase 6 ports (incl. FrontOfficeHandoffPort), Phase 7 groups, Phase 8 channels, P1–P11 platform preconditions (ARCH-007/008/010/013 are platform decisions, not Reservations Phase 1 work).
- **Reopening decisions:** ADR-072/073/074/075/076, F-05/ARCH-012 closure, F-01/F-03/F-04/F-05 availability guardrails, Reservations/FO ownership split — **preserved, not reopened**; no contradictory evidence found.

---

## 8. Verification results and unresolved limitations

**Performed:** targeted file reads and greps (§1.2, §2); `git status` snapshot before/after; api+web `pnpm typecheck` (PASS); web full `pnpm test` (FAIL, diagnosed); four single-file jest runs (2 PASS/2 FAIL, diagnoses above); `--listTests` census.

**Not performed (and therefore NOT claimed):**
- Full `apps/api` test suite — timed out; baseline unknown (223 spec files).
- All Postgres-gated suites (`AVAILABILITY_TEST_DATABASE_URL`) — would require a database; prohibited.
- HTTP/e2e tests, API boot, any runtime confirmation of R4 route shadowing and crash paths.
- `pnpm build` / lint / coverage — omitted per 4 GB RAM constraint (api `lint` is a no-op `echo 'ok'` anyway `[V]`).
- **Zero database contact:** no connection, query, migration, `db push`, `migrate deploy`, or ledger operation against `xylo_cloud` or any other database.

**Working-tree confirmation:** the only filesystem change made by this phase is the new directory `docs/reservations/phase-1/` and this file. No staged, unstaged, or untracked pre-existing work was reset, discarded, or overwritten; no application code, test, schema, or migration was modified.

**Unresolved limitations:** items tagged `[R]` (route-reachability of some 404s, dead-component list, handleAction gaps, FO guard metadata) need spot-checks during Phase 2/3 before being treated as verified; runtime behavior of R1 (cross-property read) is inferred from code, not observed — it must be converted into a failing-then-passing test in F6.

---

*End of Phase 1. No implementation started. Awaiting explicit approval of findings and minimum foundation scope (F1–F6).*

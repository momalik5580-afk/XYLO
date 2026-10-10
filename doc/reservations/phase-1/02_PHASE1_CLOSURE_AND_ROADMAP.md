# Reservations Phase 1 — Closure, Findings Register & Four-Phase Roadmap

**Plan:** XYLO Reservations Improvement Plan (4 phases: 1 Assessment & Foundation · 2 Core Reservations Backend · 3 API & Frontend · 4 Verification & Legacy Cleanup)
**Date:** 2026-10-09
**Status:** Phase 1 CLOSED — planning outputs complete; implementation of F1–F5 and all Phase 2+ work NOT started
**Authority:** Consolidates `01_PHASE1_ASSESSMENT_REPORT.md` (findings R1–R8, proposals F1–F6), ADR-076, F-05/ARCH-012 closure, roadmap v2 exit gates, and the implemented+verified F6 slice. No decision is reopened; no finding is re-audited.

**Evidence tags:** `[V]` = verified by direct file read/grep on 2026-10-09 (this finalization session) unless another date is stated; `[V-F6]` = verified in the F6 implementation/verification session (2026-10-09); `[R]` = reported by earlier exploration, spot-check pending; `[D]` = from an authoritative document.

---

## 1. Consolidated findings register

Status vocabulary: **CONFIRMED** (code/evidence inspected) · **SUSPECTED** (reported, plausible) · **UNVERIFIED** (no first-hand evidence yet) · **RESOLVED** (fix shipped and verified).

| ID | Finding (current → expected) | Evidence | Status | Decision / contract | Resolution + required tests | Deps | Phase / batch |
|---|---|---|---|---|---|---|---|
| **R1** | `findAll`/`findById`/`delete` tenant scope was conditional (`getHotelId() \|\| params.propertyId`, `if (hotelId)`) → must fail closed on ALS-absence and never honor client `propertyId` as scope. **Now fixed.** | `reservation.repository.ts:93-94,131,138,358-361,877-879` `[V]`; P1/ARCH-007 `[D]` | **RESOLVED** (`F6`, unit-verified `[V-F6]`; DB-gated proof pending → FND-10) | ARCH-007 / P1; assessment §5 F6 | Done: 400 on missing scope; `propertyId` removed from scope resolution; DTO field retained for compat. Tests: `reservation-tenant-scope.spec.ts` 10/10 `[V-F6]`. | — | Closed; residual → R1b |
| **R1b** | `update` still resolves scope conditionally: `ctxHotelId = getHotelId(); if (ctxHotelId) where.hotel_id` — absent ALS ⇒ **unscoped row lookup on the write path** (then writes proceed with `reservation.hotel_id`). | `reservation.repository.ts:576-579` `[V]` | CONFIRMED (residual of R1; explicitly out of F6 scope) | ARCH-007 / P1 (same basis as R1) | Make `update` fail closed (`requireHotelId()`); tests: extend tenant-scope spec with update cases (no-scope throws, cross-hotel isolation, ALS-scoped success). No schema change. | F6 pattern (done) | **Phase 2 / B2.6** |
| **R2** | Verification blindness: action-engine fixture `res()` never spreads overrides (suite largely meaningless); stale source-pin `t542` expects removed `inventoryDomain`; full api suite baseline unknown. | `reservation-actions.test.ts:15-30` (no `...over`) `[V]`; `t542-dead-artifacts.test.ts:112` `[V-01]`; api full run timed out `[V-01]` | CONFIRMED (fixture + pin); UNVERIFIED (api baseline) | Assessment §4.2/§5 F1; R2 | Fix fixture spread; re-run 15 tests; fix only genuine product defects exposed; update stale pin to current repository content. Tests: `reservation-actions.test.ts` 15/15 green; `t542` green. | none | **Phase 2 / B2.2** |
| **R3** | Frontend breaks lifecycle/contract: no-show via `PUT {reservation_status:'NO-SHOW'}` → DTO whitelist **400**; share-payee calls non-existent `/select-share-payee\|/unselect-share-payee` → **404** (reachable from live detail workspace); linked POSTs `/link` → **404** (panel believed unmounted, latent). | `reservation.api.ts:233` and `reservationStore.ts:385` `[V]`; `shares-payee.api.ts:22,29` vs backend `:id/shares/:gid/select-payee|unselect-payee` `[V]`; `linked-reservations.api.ts:45` vs backend `POST :id/linked` `[V]`; reachability `[R]` | CONFIRMED (code); SUSPECTED (share-payee reachability, `[R]`) | ADR-004/005/006 (explicit lifecycle commands); 12_API_CONTRACT | Wire no-show → `POST :id/no-show`; fix share-payee and linked paths; keep store action names. Tests: targeted web tests per mutation + `pnpm typecheck` (web); add request-shape assertions (contract tests land B3.4). | none | **Phase 2 / B2.5** |
| **R4** | Two dead/broken backend endpoints: `GET :id/linked` crashes via `(this as any).repo` (no such member); static `GET room-moves` (254) and `GET display-sets` (512) declared after single-segment `GET :id` (69) ⇒ expected shadowed (runtime `[R]`). Sales screen `getDisplaySets` consumer affected; coupon POSTs unaffected. | `reservations.service.ts:585-589` `[V]`; controller line order `[V]`; `sales-screen.api.ts:23` `[V]` | CONFIRMED (code); SUSPECTED (shadowing runtime effect) | Assessment §5 F4/F5; ADR-013 boundary | F4: dispatch a registered query/repository instead of `(this as any).repo` + unit test. F5: move static GETs above `GET :id` + first HTTP-level supertest spec proving reachability of `room-moves`, `display-sets` and guard/permission behavior. | HTTP test harness (supertest) introduced in F5 | **Phase 2 / B2.3, B2.4** |
| **R5** | Status vocabulary/storage drift: backend canonical `SCREAMING_SNAKE`; `toStorage()` maps only `NO_SHOW→'NO-SHOW'` (so `CHECKED_IN` stays `CHECKED_IN`) while the integration spec expects `'CHECKED-IN'`; shared `normalizeStatus('checked-in')` **throws**; web types are kebab-case and `normalizeApiStatus` uppercases then maps. 3 red tests today. | `reservation-status.service.ts:12-14,58-62` `[V]`; `reservation-integration.spec.ts:79-87` `[V]`; `reservation-state-machine.ts:65` `[V-01]`; `reservation.types.ts:43-51` `[V]`; `reservation.api.ts:59-66` `[V]` | CONFIRMED | F2 (assessment §5): owner decision on canonical storage/status vocabulary before any lifecycle work; ADR-005/006 | Owner decision (spec ↔ service ↔ shared machine ↔ web mapping aligned by choice, not assumption); implement minimal change covered by the same spec; record decision note. Tests: `reservation-integration.spec.ts` green; web `normalizeApiStatus` tests green. Third failure "rejects unknown statuses" root cause to be confirmed inside this batch (FND-13). | **owner decision** | **Phase 2 / B2.1 (first)** |
| **R6** | 7 Front Office sites write `reservations` directly vs ARCH-012 guardrail "new reservation writes only through ReservationRepository / registered commands". Inspected sites include `hotel_id` in `where` (boundary divergence, not a verified tenant leak). | `check-in-commit.service.ts:80`, `check-out.handler.ts:235`, `transfer-room.handler.ts:49`, `upgrade-room.handler.ts:113`, `change-rate-code.handler.ts:71`, `reverse-check-in.handler.ts:70`, `front-office.service.ts:126` `[V-01]`; ARCH-012 `[D]` | CONFIRMED (code); guardrail conflict OPEN | ARCH-012 guardrails; Reservations=Lifecycle / FO=CheckInOut ownership `[D]` | Owner ruling required (converge writers vs accept documented divergence). No code change without ruling. Tests: only if convergence ordered (Phase 4 batch). | **owner ruling** | Ruling: **Phase 2 / B2.9**; convergence (if ordered): **Phase 4 / B4.4** |
| **R7** | `apps/web/store/reservationStore.ts` holds server state in Zustand (reservations/profiles/rateCodes/KPIs via direct `api.*`), still mounted through `CreateProfileModal` and `CheckoutFlowModal`; duplicate query-key registries; dual reinstate implementations. | assessment §2.17 `[R]` (pattern + mounts) | SUSPECTED (spot-check pending) | "TanStack Query owns server state" convention | Migrate reads/writes to Query hooks (`reservationKeys`); unmount store; single key registry; converge reinstate. Tests: web suite green; grep proving no store-side fetchers remain. | R3 fixes land first (same files) | **Phase 3 / B3.2, B3.3** |
| **R8** | Dead/legacy candidates: `components/reservations/events/` (23 files), `activities/`, 2 modals, `ReservationToolbar`, 9 orphan config endpoints that would 404, duplicate nav; ARCH-042: 82 zero-importer files repo-wide (incl. Reservations Phase-6 port adapters). | assessment §2.18 `[R]`; ARCH-042 `[D]` | SUSPECTED / classify-only | Roadmap Phase 11 (legacy removal with cutover rehearsal); ARCH-042 | Classify → RETAIN/DELETE ledger; deletion only in Phase 4 after zero-reference evidence + authorization. Tests: importer-count evidence per item at deletion time. | B3.5 classification | Classify: **Phase 3 / B3.5**; delete: **Phase 4 / B4.3** |
| **FND-09** | Reported frontend contract deltas (spot-check pending): `ChangeRateDialog` sends `nightly_rate/total_rate` instead of `PUT :id/rate`; `MassUpdateDialog` status casing rejected by status validation; `ReservationsHomeView.handleAction` missing `noShow/reinstate/guarantee/modify` cases; detail workspace `noShow` opens Cancellation dialog; `/reservations/new` and `/reservations/import` links 404. | assessment §2.16 `[R]` | UNVERIFIED (each item) | 12_API_CONTRACT; ADR-004 | Spot-check each; fix per contract; add request-shape contract test per mutation (B3.4). | B2.5 (F3) patterns | **Phase 3 / B3.1** |
| **FND-10** | F6 DB-gated test updates not runtime-verified: `reservation-delete.postgres.spec.ts` and the postgres block of `t560-delete-never-releases.spec.ts` were wrapped in `requestStorage.run({hotelId})` but never executed (no DB). | files `[V-F6]`; `describePostgres` gate `[V-F6]` | UNVERIFIED (runtime) | F6 acceptance (cross-property negative under real DB) | Run both files against an **approved, isolated test database** (never `xylo_cloud`); fix any failures. Tests: those two files green. | **explicit DB approval** | **Phase 2 / B2.7** |
| **FND-11** | F6 changes (repository + 4 test files) live in the **untracked** `apps/api/src/modules/reservations/infrastructure/` tree — invisible to `git diff`; 750 status entries, 0 stashes preserved throughout. | `git status` `[V-F6, V]` | CONFIRMED (hygiene) | Repo preservation rule; commit only on explicit owner request | Owner decision to commit/stage; agents must not commit unprompted. | owner | Owner action (any time); tracked in §5 |
| **FND-12** | Sales payload difference: frontend `validateCoupon` sends `{couponCode, rateCode}`; backend route binds `{couponCode}` only (`rateCode` silently ignored — no class-validator on inline body type, so whitelist does not strip). `display-sets` GET additionally affected by R4 shadowing. | `sales-screen.api.ts:26-34` `[V]`; `reservations.controller.ts:519-524` `[V]` | CONFIRMED (code delta); impact UNVERIFIED | 12_API_CONTRACT sales surface | Decide: accept-and-ignore (document) or extend contract; cover with contract test; display-sets fixed by B2.4. | B2.4 | **Phase 3 / B3.6** |
| **FND-13** | Third F2 failure ("rejects unknown statuses", `reservation-integration.spec.ts:68-71`) appears logically correct against current `isValidStatus`; root cause of its recorded failure not re-diagnosed. | spec `[V]`; failure recorded `[V-01]` | UNVERIFIED (root cause) | F2 batch scope | Re-run the file in B2.1; diagnose and resolve within the F2 decision scope. | B2.1 | **Phase 2 / B2.1** |

**Coverage-gap closures since the assessment:** tenant-scoping tests now exist (R1); the first repository-level cross-hotel isolation tests exist (FND-01 spec). Still missing: HTTP-level route/guard tests (B2.4), web↔api contract tests per mutation (B3.4), lifecycle e2e (B4.1), api full-suite baseline (B4.2).

---

## 2. Foundation items F1–F6 — implementation status

| Item | Scope | Status (2026-10-09) | Evidence |
|---|---|---|---|
| **F1** | Fix `res()` fixture spread; re-run action-engine tests; fix only real defects | **NOT IMPLEMENTED** | `reservation-actions.test.ts:15-30` returns literal without `...over` `[V]` |
| **F2** | Status-vocabulary owner decision + spec/service/machine alignment | **NOT IMPLEMENTED** (decision not recorded; 3 tests still red by recorded baseline) | spec expectations unchanged `[V]`; `toStorage` unchanged `[V]` |
| **F3** | Frontend no-show / share-payee / linked contract fixes | **NOT IMPLEMENTED** | `reservation.api.ts:233`, `reservationStore.ts:385`, `shares-payee.api.ts:22,29`, `linked-reservations.api.ts:45` all unchanged `[V]` |
| **F4** | Fix `getLinkedReservations` crash | **NOT IMPLEMENTED** | `reservations.service.ts:588` still `(this as any).repo` `[V]` |
| **F5** | Move static GETs above `GET :id`; first HTTP route test | **NOT IMPLEMENTED** | controller order 69 → 254 → 512 unchanged `[V]`; no supertest spec `[V-01]` |
| **F6** | Repository tenant scope fail-closed (`findAll`/`findById`/`delete`); drop client `propertyId` as scope | **IMPLEMENTED + UNIT-VERIFIED** (see §5) | repository lines 93-94,131,138,358-361,877-879 `[V]`; tests `[V-F6]` |

No PATCH route exists anywhere in the module `[V]` — correct: ADR-076 authorizes it for the first amendment slice (Phase 2 / B2.8); it is not part of F1–F6.

---

## 3. Four-phase roadmap (ordered batches)

Constraints applying to all batches: no schema/migration changes; no `xylo_cloud` contact; no package installs; no legacy deletion before Phase 4 ledger; sequential low-memory test execution (`-w 1`, jest `isolatedModules` workaround where the heap caps); preserve all existing work.

### Phase 1 — Assessment & Foundation → **CLOSED** (this document)

Outputs: assessment report (01), findings register + roadmap + closure (02), F6 shipped. Exit criteria met — see §6.

### Phase 2 — Core Reservations Backend

| Batch | Work | Acceptance criteria | Focused tests | Deps |
|---|---|---|---|---|
| **B2.1** | **F2 + FND-13**: owner decision on canonical status vocabulary; align spec ↔ `reservation-status.service` ↔ shared machine; diagnose third failure | Decision note filed; `reservation-integration.spec.ts` green (`-w 1`); no other suite newly red | `reservation-integration.spec.ts` 17/17 | **owner decision** |
| **B2.2** | **F2→F1**: fix `res()` fixture spread; triage newly exposed failures (product vs test); fix stale `t542` pin | `reservation-actions.test.ts` 15/15; `t542` green; any model change justified per case | both files `-w 1` | B2.1 optional (independent) |
| **B2.3** | **F4**: replace `(this as any).repo` with a registered query/repository dispatch for `getLinkedReservations` | `GET :id/linked` no longer throws TypeError (unit-proven); typecheck green | new unit test for the service path | none |
| **B2.4** | **F5**: move `GET room-moves` + `GET display-sets` above `GET :id`; add first HTTP-level spec | supertest proves both routes reachable and not shadowed; `@Permission`/`@PropertyScope` behavior locked; R4 runtime suspicion resolved | new supertest route spec | supertest harness introduced here |
| **B2.5** | **F3**: no-show → `POST :id/no-show` (api + store); share-payee → `:id/shares/:gid/select-payee\|unselect-payee`; linked → `POST :id/linked` | targeted web tests green; typecheck green; no backend change; request shapes match 12_API_CONTRACT | web tests per mutation | none (B2.4 independent) |
| **B2.6** | **R1b**: fail-closed `update` (`requireHotelId()`); keep write behavior otherwise identical | tenant-scope spec extended: update no-scope throws, cross-hotel isolation, ALS-scoped success | extend `reservation-tenant-scope.spec.ts` | F6 (done) |
| **B2.7** | **FND-10**: execute the two DB-gated delete specs | both files green against approved isolated DB; zero contact with `xylo_cloud` | `reservation-delete.postgres.spec.ts`, `t560` postgres block | **explicit DB approval** |
| **B2.8** | **ADR-076 PATCH slice**: `PATCH /:id` same DTO/path/validation as `PUT /:id`; PUT retained; no lifecycle fields | one shared code path; test proves `reservation_status` rejected on both verbs; contract text unchanged | controller-level test + validation test | none |
| **B2.9** | **R5/R6 owner rulings** recorded (storage-form outcome from B2.1; FO-writer disposition) | decision notes filed; no FO behavior change without ruling | — | B2.1 (for R5 final wording) |

**Phase 2 exit gate:** batches B2.1–B2.6 green under their focused commands (`-w 1`) + `pnpm typecheck` green in `apps/api` and `apps/web`; B2.7 green or explicitly deferred with blocker named; B2.8 contract test green; B2.9 notes filed. Per roadmap v2, command-handler concurrency/idempototency/authorization tests remain a Phase 4 (their Phase 10) sweep responsibility — not silently waived here.

**B2.8 contract-test disposition (2026-10-10):** `apps/api/test/reservations-amendment-verbs.e2e-spec.ts` was surfaced red (3 of 6) during B3.6 and root-caused here. The 403s originated in `ProtectedAmendFieldsGuard` (field-level authorization): the fixture's `VALID_BODY` carries `room_rate`, which the guard gates behind `RESERVATION_CHANGE_RATE`, while `GRANTS['u-editor']` held only `RESERVATION_UPDATE` — so PUT/PATCH were denied before validation (captured HTTP body: `FORBIDDEN / "Insufficient permissions"`, guard `canActivate` final throw; all other guards passed for the same token, proven by the tests that expect 400 succeeding). Contract basis: guard unit spec (T2), 12_API_CONTRACT §33/§34/§37, ADR-076. **Fix applied:** fixture grants now `[RESERVATION_UPDATE, RESERVATION_CHANGE_RATE]`; guard, DTO, controller, and all six assertions unchanged (no relaxation of authorization). **Verification:** focused run 6/6 (`TARGET_EXIT=0`); full api e2e 3/3 suites 23/23 (`E2E_EXIT=0`); guard unit spec + `t1-rate-api-contract` + `reservation-tenant-scope` green; `pnpm typecheck` 0. B2.8's own acceptance criteria are met; the **Phase 2 exit gate is not asserted closed here** (B2.7 remains DB-gated; other rows unchanged). Incidental, pre-existing, unrelated to B2.8: a 4-spec unit run also surfaced 6 reds in `reservation-update-wiring.spec.ts` (`No property context. Send x-property-id header.` thrown from `requireHotelId()` at `reservation.repository.ts:577` on `update`) — consistent with the B2.6 fail-closed update requiring ALS context this spec never populates (spec was green 6/6 before that change, per availability phase-5/6 evidence); logged for B4.2 baseline ledger or a B2.6 follow-up, not fixed in this pass.

**Update-wiring disposition (2026-10-10, follow-up to the incidental finding above):** the six reds in `apps/api/src/modules/reservations/infrastructure/__tests__/reservation-update-wiring.spec.ts` are root-caused and fixed. **Failing tests** (all under `ReservationRepository.update (T-13)`): "replaces the assertion for the date delta inside one transaction"; "aligns the DTO payload keys with the repository contract"; "issues a cross-room-type replacement when only the room type changes (T-14)"; "skips the port when no current assertion exists"; "reports no modification when the payload carries no aligned keys"; "T5-12 (INV-P5-24): CHECKED_OUT holds — checkout issues no release and no assert". **Verified root cause:** the spec predates B2.6/R1b — before that change `update` scoped conditionally (`getHotelId() || row.hotel_id`, the R1b finding row), so the fixture legitimately passed with no ALS property context; B2.6 replaced it with fail-closed `requireHotelId()` at `reservation.repository.ts:577` (lookup scoped `{id, hotel_id}`), so each of the six `repository.update` calls threw `BadRequestException: No property context. Send x-property-id header.` from `request-context.ts:66` (identical mechanism for all six, each at its own call site — spec lines 69, 100, 150, 171, 181, 192; pre-fix run 6 failed/6, exit 1). Ruled out: production-context gap (global `TenantInterceptor` establishes ALS per request, `app.module.ts:110`; schedulers use `runAsServicePrincipal`; `runWithTransaction` uses a **separate** AsyncLocalStorage at `ambient-transaction.ts:17` and cannot clobber property context) and repository/mock defects (tenant-scope `update (R1b)` tests green; the harness mocks are correct for wiring isolation). Prior green evidence: availability phase-5 (`06_EXECUTION_EVIDENCE.md` update-wiring **6/6**) and phase-6 (`04_EXECUTION_EVIDENCE.md` 7/7 battery) — spec broken only by the later B2.6 change, whose focused test list (`reservation-tenant-scope.spec.ts` only) never included this wiring spec. **Fix (test-only — production code, schema, migrations, seeds, DB state unchanged):** mirrored the established sibling pattern (`reservation-delete-wiring.spec.ts:51`, `reservation-batch-status-wiring.spec.ts:46`, `reservation-tenant-scope.spec.ts:71`) in `reservation-update-wiring.spec.ts` — added `import { storage as requestStorage } from '@/core/context/request-context'` plus helper `asHotel = (fn) => requestStorage.run({ hotelId: 'H1', userId: 'u1' }, fn)`, wrapping all six `runWithTransaction(...)` call sites. **Fail-closed and tenant isolation intact** — `reservation-tenant-scope.spec.ts` `update (R1b)` remains green (no-scope throws + `findFirst` never called; cross-hotel row rejected; ALS-scoped read/write). **Verification (low-memory temp jest configs, `--runInBand`, §5 workaround):** single-file run → **6/6, exit 0** (baseline was 6 failed/6, exit 1); context/regression set (tenant-scope, delete-wiring, batch-status-wiring, property-scope-c01, service-principal) → **62/62, exit 0**; sibling wiring sweep (cancel/no-show/reinstate/waitlist) → **15/15, exit 0**; `pnpm typecheck` → **exit 0**. **Gate relationship:** B2.6's own acceptance criteria (tenant-scope extension) were satisfied then and remain satisfied — the defect was the unlisted dependent spec, now covered; B2.8 unaffected (amendment-verbs remains 6/6); no Phase 2 or Phase 3 gate status changed by this fix; B4.2's full-suite baseline ledger will start without these six reds. **Remaining follow-up:** the full api unit baseline still belongs to B4.2 (not run here); no database/schema work was authorized or performed.

### Phase 3 — API & Frontend

| Batch | Work | Acceptance criteria | Focused tests | Deps |
|---|---|---|---|---|
| **B3.1** | **FND-09** spot-checks + fixes (rate dialog, mass-update casing, handleAction cases, missing `/new` `/import` routes) | each item verified then fixed or reclassified with evidence | per-item web tests | B2.5 patterns |
| **B3.2** | **R7**: migrate reservation server state from Zustand to TanStack Query; unmount `reservationStore` consumers | no store-side fetchers for reservations; web suite green | web suite `-w 1` + grep evidence | B2.5 (shared files) |
| **B3.3** | Single query-key registry; converge dual reinstate paths | one registry in use; one reinstate path | web tests | B3.2 |
| **B3.4** | Web↔api contract tests per mutation (request-shape assertions) | no-show, share-payee, linked, amend, rate, room-type each locked | new contract spec(s) | B2.5, B2.8 |
| **B3.5** | **R8 classification** (no deletion): RETAIN/DELETE ledger for dead components, orphan endpoints, ARCH-042 zero-importer list | ledger drafted with importer-count evidence; owner review queued | evidence recorded, not a test | B3.1 |
| **B3.6** | **FND-12**: sales payload alignment decision (document-and-ignore or extend contract) | decision recorded; display-sets confirmed working after B2.4; contract test added | contract test | B2.4 |

**Phase 3 exit gate:** B3.1–B3.4 green; contract tests green; no Zustand server-state fetchers for reservations (grep proof); B3.5 ledger exists (deletion NOT performed).

### Phase 4 — Verification & Legacy Cleanup

| Batch | Work | Acceptance criteria | Focused tests | Deps |
|---|---|---|---|---|
| **B4.1** | Lifecycle e2e sweep: create → confirm → amend (PATCH) → cancel → reinstate → no-show | sweep green; each transition asserted through explicit commands (ADR-004/006) | new e2e spec | B2.4 harness, B2.8 |
| **B4.2** | Full-suite baseline + green: api + web under documented low-memory commands; postgres suites with approved DB | documented pass/fail ledger; reds fixed or explicitly waived | full runs (approved) | B2.7 DB approval |
| **B4.3** | Execute RETAIN/DELETE ledger (R8) with authorization; deletion only after zero-reference re-verification at deletion time | deletions only for ledger-approved, zero-importer items; post-deletion suite green | importer-count re-check + suite | B3.5, owner auth |
| **B4.4** | FO direct-writer convergence **only if** R6 ruling ordered it | writers converge through registered commands; FO behavior unchanged per handoff contract | FO + reservations suites | B2.9 ruling |

**Phase 4 exit gate:** e2e sweep green; full-suite ledger documented; authorized deletions complete with rollback notes; roadmap v2 Phase 10 (operational validation) satisfied for the Reservations boundary. Roadmap items 2b/6/7/8/11/12 remain **outside** these four phases (unchanged ownership).

---

## 4. Outstanding verification items and blockers

| Item | Type | Blocker? | Resolves in |
|---|---|---|---|
| F6 DB-gated specs (FND-10) | Runtime verification | **Yes for B2.7** — requires explicitly approved, isolated test database (never `xylo_cloud`) | B2.7 |
| F2 owner decision (R5) | Owner decision | **Yes for B2.1** (first Phase 2 batch) | owner, B2.1 |
| R6 owner ruling | Owner decision | Yes before any FO writer change | owner, B2.9 |
| F1–F5 implementation | Not started | No (Phase 2 work) | B2.2–B2.5 |
| R4 route-shadowing runtime proof | Verification | No (suspicion recorded; B2.4 proves) | B2.4 |
| R3 share-payee reachability, R7 mounts, FND-09 items | Spot-checks | No | B2.5, B3.1–B3.2 |
| FND-13 third F2 failure root cause | Diagnosis | No (inside B2.1) | B2.1 |
| Full api suite baseline (223 files) | Verification | No (RAM-constrained; Phase 4) | B4.2 |
| FND-11 untracked `infrastructure/` tree | Repo hygiene | No for planning; **owner commit decision needed** before Phase 2 diffs build on it | owner |
| HTTP/e2e coverage (route, guard, lifecycle) | Coverage gap | No (built in B2.4/B3.4/B4.1) | Phases 2–4 |

**No planning blocker remains for Phase 1 closure.**

---

## 5. F6 implementation & verification status

**Implemented (2026-10-09):** `apps/api/src/modules/reservations/infrastructure/repositories/reservation.repository.ts`
- `findAll` (93-94): scope = `getHotelId() || params.hotelId`, 400 `BadRequestException` if neither; `hotel_id` filter unconditional (131); guests sub-query unconditional (138). Client `propertyId` no longer used as scope (DTO field retained).
- `findById` (358-361): explicit arg falls back to ALS, 400 if neither, `where` always `{id, hotel_id}`.
- `delete` (877-879): `requireHotelId()` only; `where` always scoped.

**Guard-chain rationale (verified):** `SearchReservationDto` has no `hotelId` field and global `ValidationPipe({whitelist, forbidNonWhitelisted})` rejects stray query params; ALS is filled by global `TenantInterceptor` via `PropertyScopeResolver.resolveForAls` (header is a *proposal*, accepted only on identity match or cross-property grant; REFUSED ⇒ 403 via `PermissionGuard`, which also requires an authenticated user on every affected route). `search`/view handlers pass `getHotelId()`-derived values; command handlers pass ALS-derived `command.hotelId`; no `IReservationRepository` injectors exist outside the reservations module.

**Verification evidence (`-w 1`, sequential, temp jest config with ts-jest `isolatedModules: true` — package.json jest config OOMs at 4 GB even with one worker):**
- `reservation-tenant-scope.spec.ts` (new, 10 tests): **10/10 pass** — fail-closed ×3, client-`propertyId`-ignored, explicit-internal path ×2, cross-hotel isolation ×2, ALS-scoped read/delete.
- `reservation-delete-wiring.spec.ts` (8 call sites wrapped in `requestStorage.run`): **13/13 pass**.
- `t560-delete-never-releases.spec.ts`: **2 pass** (static pins on `async delete(id: string)` signature + ordering still hold) + **3 skipped** (postgres block).
- `pnpm typecheck` (`apps/api`): **PASS**.
- Focused verification review: PASS on diff/caller-trace/no-override/fail-closed checks; read-only; 750 status entries and 0 stashes preserved.

**Remaining concerns:**
1. **R1b `update` conditional scope** (`reservation.repository.ts:576-579`) — same class of hole as R1 on the write path; deliberately untouched by F6; scheduled B2.6.
2. **FND-10**: the two DB-gated delete specs were updated (ALS wrappers) but **not executed** — no runtime claim made; blocked on approved DB (B2.7).
3. **FND-11**: all F6 files sit in the untracked `infrastructure/` tree; owner commit decision pending. Nothing was staged, committed, reset, or stashed by any session.
4. Jest low-memory workaround (temp `isolatedModules` config) is required for any future single-file api test run on this machine; the package.json jest config remains OOM-prone (recorded for B4.2 planning).

---

## 6. Phase 1 closure checklist

- [x] Assessment report exists at the established path (`01_PHASE1_ASSESSMENT_REPORT.md`) and is not duplicated.
- [x] Decisions preserved, not reopened: ADR-072/073/074/075/**076** (PATCH canonical, PUT retained, no lifecycle via amendment `[D]`); **F-05/ARCH-012** closed "GO WITH GUARDRAILS" (`reservations.id` UUID authoritative; writes through repository/commands; three deferred guest-identity decisions `[D]`).
- [x] Consolidated findings register produced (§1) with stable IDs, evidence, current-vs-expected, confirmation status, governing decision, proposed resolution + tests, dependencies, and phase mapping for every known finding (R1–R8, R1b, FND-09…13).
- [x] Four-phase plan finalized (§3) — exactly four phases; every finding mapped to a batch; dependencies, acceptance criteria, focused tests, and exit criteria defined; backend contract/lifecycle work sequenced ahead of dependent frontend work; legacy deletion placed only after zero-reference evidence.
- [x] F1–F6 implementation status recorded (§2): only F6 implemented; F1–F5 explicitly pending — no false "done" claims.
- [x] F6 implementation/verification status documented with actual test results (§5), including the `update` (R1b) residual and the unverified DB-gated tests.
- [x] Safety boundaries restated (§7): identity guardrails, PATCH/PUT rules, lifecycle via explicit commands, Reservations/FO ownership split, no shadow database, no `xylo_cloud`, no schema/migration/reset/ledger-repair/package-install/service-startup, full work preservation.
- [x] Outstanding verification items and blockers enumerated (§4); unverified runtime behavior explicitly delegated to implementation/verification batches, never marked verified.
- [x] No application code, tests, schemas, migrations, or docs other than this deliverable and the minimal 01 status update were modified during finalization.

**Phase 1 closure status: CLOSED.** All required planning outputs are complete; every known finding is accounted for; no planning blocker remains. Phase 2 must not begin until the owner reviews this document and (a) approves the batch order, (b) supplies the F2 vocabulary decision, and (c) rules on R6 — with B2.7 additionally gated on explicit DB approval.

---

## 7. Safety boundaries (binding on all subsequent phases)

1. `reservations.id` (UUID) is the authoritative reservation identity; `reservation_name`/`resv_name_id` remain a write-through legacy projection + allocator; no reliance on `reservation_guests.profileId` semantics, `reservation_name.reservation_status` sync, or the `guest_profiles` raw-SQL table (ARCH-012 guardrails).
2. `PATCH /:id` is canonical for permitted attribute amendments; `PUT /:id` remains temporarily compatible (ADR-076); neither may carry lifecycle transitions — lifecycle changes go through explicit registered commands (ADR-004/005/006).
3. Reservations owns reservation lifecycle; Front Office owns operational check-in/check-out/transfer/upgrade (unchanged split).
4. No shadow database, no parallel source of truth, no schema changes in these phases (2b/6/11 remain roadmap-owned).
5. Never connect to, query, or modify `xylo_cloud`; any database-touching verification (B2.7, B4.2) requires explicit approval of an isolated test database.
6. Preserve every staged, unstaged, deleted, and untracked file; no reset, clean, stash, overwrite, or removal of existing work; no commits unless explicitly requested.
7. 4 GB RAM: sequential execution, one worker, targeted reads/searches; no full suites/builds without Phase 4 approval; jest `isolatedModules` workaround for api single-file runs.

---

*End of Phase 1 closure document. Planning only — no Phase 2 implementation started. Awaiting owner review.*

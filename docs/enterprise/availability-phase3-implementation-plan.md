# XYLO Availability Phase 3 — Stage D Readiness Review

**Review subject:** `docs/enterprise/availability-phase3-implementation-plan.md` (v0.1, 1,172 lines, 79,179 bytes, status `DRAFT — AWAITING STAGE D READINESS REVIEW`)

---

## 1. Document Control and Executive Verdict

| Field | Value |
|---|---|
| Document | XYLO Availability Phase 3 — Stage D Readiness Review |
| Review type | Readiness review of the Implementation Plan (no plan edits applied in this pass) |
| Version | 1.0 |
| Date | 2026-09-29 |
| Plan reviewed | `availability-phase3-implementation-plan.md` v0.1 (SHA-unmodified by this review) |
| **Verdict** | **READY WITH REQUIRED PLAN CLARIFICATIONS** |
| Required clarifications | **15** (RC-01 … RC-15, §19) — listed, **not applied** |
| Recommended improvements | 2 (REC-01, REC-02, §19) — non-blocking |
| Execution authority granted | **None.** No task may be implemented until RC-01 … RC-15 are applied to the plan, the plan is re-issued as v0.2, and the amended plan is approved. |

**Executive summary.** The plan is structurally sound, follows the mandated source hierarchy, classifies every change with the prescribed key, and maps all nine Stage B rulings to owning tasks. Its per-task format, wave structure, conflict register, gates and deferred-item discipline are the right shape for execution. It is **not yet executable** for four independent reasons:

1. **Lifecycle coverage gaps against the locked specification** — Confirm/Guarantee (spec §11 rows 203/204, §28 "Confirm / Guarantee / Promote" command), the ratified D-6 defect branch `check-out.handler.ts:94-111`, scheduled room moves (`scheduled_room_moves.to_room_type`), and hold creation/hold expiry (F-03) are required by the specification but absent from §2 Scope, §4.2, §7, §9, §16.1 and the task list (RC-01, RC-02, RC-03, RC-04).
2. **Internal cross-reference drift** — approximately 35 wrong task numbers in §4.4, §5, §10, §11, §12, the R5 contract note and four task `Deps`/`Tests` lines, while §21 `Deps` and §22 Traceability are correct (RC-05).
3. **Two dependency cycles and one undefined wave** — T-19/T-20 → T-33 → T-32 → T-19/T-20; T-04 → T-31 → T-27 → T-04; wave R4 is referenced by §17 and by T-12 `Deps` but contains no task rows (RC-06, RC-07).
4. **Under-specified mechanisms** — D-21 first-touch versus the D-24 lock order; HTTP `Idempotency-Key` → `Command.idempotencyKey` propagation and three-layer dedup precedence; frontend key minting point; T-01 apply command and drift-reconciliation mechanism; T-11 status-classification completeness; D-23 Front Office payload mapping (RC-08 … RC-13).

Nothing in the plan reopens a Stage B decision, edits the locked specification, or introduces Reservations-track scope. One scoping statement (T-18) and one status-mapping statement (T-11) would, as written, leave or create divergence from the locked specification; both are resolved by amending the plan, not the specification (RC-02, RC-12).

---

## 2. Review Scope, Authority, Method and Evidence Labels

**Authority order applied (no deviations):** 1) `availability-phase3-domain-specification.md` — LOCKED (2026-09-28), v0.3, 613 lines, 62,320 bytes, 35 sections; 2) `availability-phase3-stage-b-ratification.md`; 3) `availability-phase3-business-rules-decision-sheet.md` (13 rows, RATIFIED); 4) `availability-phase1-2-contract-extract.md` (derived); 5) `availability-phase3-stage-c-lock-review.md`; 6) `availability-phase3-reservation-lifecycle-forensic-audit.md`; 7) repository code/tests/migrations as **evidence only**.

**Method.** (a) Full read of the plan's 22 sections, 37 tasks and self-review; (b) line-by-line verification of plan citations against source; (c) read-only repository commands (`npx prisma migrate status`, `rg`, file reads, one baseline `jest` run). **No file in the repository was modified by this review except this report.** The Implementation Plan was not edited. The locked specification was not edited. No production code, schema, migration, test, route or frontend change was made.

**Evidence labels used below:** `[VERIFIED FACT]` = confirmed by direct file/line/command inspection during this review · `[OBSERVED BEHAVIOR]` = result of an executed command · `[INFERENCE]` = reasoned conclusion from verified facts · `[BUSINESS DECISION REQUIRED]` = the locked specification does not decide it · `[ARCHITECTURAL DECISION REQUIRED]` = mechanism choice the plan must make and state.

---

## 3. Plan Integrity and Self-Review Verification

| Plan claim | Review verification | Label |
|---|---|---|
| 37 tasks T-01 … T-37 | Confirmed: 37 task headings present, no gaps, no duplicates | `[VERIFIED FACT]` |
| 22 conflicts X-1 … X-22 | Confirmed in §4.4 | `[VERIFIED FACT]` |
| 12 deferred items | Confirmed: 12 rows in §20 | `[VERIFIED FACT]` |
| 22 change-inventory entries C-01 … C-22 | Confirmed in §6 | `[VERIFIED FACT]` |
| 10 gates G1 … G10 | Confirmed in §18 | `[VERIFIED FACT]` |
| 21 functional cases F-01 … F-21 | Confirmed in §16.1 | `[VERIFIED FACT]` |
| 0 authored migrations; 18 pending; 4 drift | Confirmed: 45 migration folders in `packages/db/migrations`, 18 pending, last common `20260819000000_add_inv_audit_log` | `[OBSERVED BEHAVIOR]` |
| 8 pre-existing test failures | Reproduced: **50 suites — 44 passed, 3 failed, 3 skipped; 445 tests — 417 passed, 8 failed, 20 skipped** (Postgres-gated suites skipped without `AVAILABILITY_TEST_DATABASE_URL`) | `[OBSERVED BEHAVIOR]` |
| 1,124 live Reservations | Previously verified against `xylo_cloud.public` | `[OBSERVED BEHAVIOR]` |
| 3 web files (`lib/api/client.ts`, `services/api.ts`, `components/Providers.tsx`) | Confirmed present; `apps/web` has zero idempotency references today | `[VERIFIED FACT]` |
| Source hierarchy, classification key, "Execution authority: None" | Present in §1 and correct | `[VERIFIED FACT]` |

**Self-review defects found** (must be corrected with the RC amendment):

- "of the 22 change-inventory entries, 20 are MUST IMPLEMENT … 2 are DEFERRED" contradicts §6, where all **22** numbered entries are MUST IMPLEMENT and DEFER appears only on unnumbered rows. `[VERIFIED FACT]`
- "6 components … are **not** converted into tasks" contradicts T-18, which is a task for an EXISTING-AND-CPLIANT component (normal checkout, verify-only). `[VERIFIED FACT]`
- "Unresolved dependencies: 3" omits the two dependency cycles in RC-06 and the undefined wave R4 (RC-07). `[VERIFIED FACT]`

---

## 4. 13-Row Decision Matrix Coverage (Decision Sheet rows 1–13)

Verdict key: **BRIEFED** = row's rule is carried into scope, tasks, tests and gates · **PARTIALLY BRIEFED** = carried but with a named gap · **NOT BRIEFED**.

| # | Decision (Stage B verdict) | Plan coverage | Verdict |
|---:|---|---|---|
| 1 | PENDING consumption — hold consumes, no-availability PENDING does not (RATIFIED; spec §10.1/§10.9/§32.10) | §7 row 1, F-02, F-03, T-12 — **but** hold creation has no implementing code path and hold **expiry** has no owner/mechanism (RC-04); the PENDING→Confirm transition (§11 r203/204) has no task (RC-01) | **PARTIALLY BRIEFED** |
| 2 | No-show timing — release unelapsed nights only (RATIFIED; D-3, code is defect) | §7 row 5, T-16, F-07, §22 "§10.6 → T-16" | **BRIEFED** |
| 3 | Early departure (RATIFIED; spec §10.4/§13/§32.27) | §7 row 7, T-17, F-08 | **BRIEFED** |
| 4 | Checkout release — scheduled exclusive departure is the boundary, no release (RATIFIED; code already compliant) | §7 row 8, T-18, F-09 | **BRIEFED** (subject to RC-02 scoping of T-18) |
| 5 | Overstay — secure added night **before** committing extended dates, fail closed (RATIFIED; D-6 names `check-out.handler.ts:94-111`) | §7 row 9 + T-19 cover the **extend commands only**; the D-6 defect location is absent from §4.2, §4.4, §7, §9, §16.1 and every task's file scope; T-18's "same handler, `earlyDeparture=false` → Change: none" would leave it untouched | **PARTIALLY BRIEFED** (RC-02) |
| 6 | Reinstatement (RATIFIED; code defect P-6) | §7 row 10, T-20, F-12/F-13, X-18 | **BRIEFED** |
| 7 | Multi-room grouping / quantity = 1 (RATIFIED WITH DATA; D-0) | T-11 `quantity = candidates.length`, T-12 "MUST NOT add a quantity field", §22 "§7 quantity = 1 → T-11/T-12" | **BRIEFED** |
| 8 | Assertion linkage — one active assertion per Reservation, exact reference (RATIFIED; encoded in migration + port) | T-03/T-05/T-06, §22 "§9 → T-05/T-06", F-06/F-12, G4 | **BRIEFED** |
| 9 | Idempotency ownership — spec §22 adopted, frontend funded (RATIFIED + SCOPE ADDED; D-15) | §11, T-04, T-27 … T-31, §16.2, G6 — **but** HTTP header → `Command.idempotencyKey` propagation is undefined and the three dedup layers have no stated precedence; frontend key minting point unspecified (RC-09, RC-10) | **PARTIALLY BRIEFED** |
| 10 | Legacy counter cutover (RATIFIED; D-20 → Phase 6) | §13.2/§13.3/§13.4, §20 rows 1–3, G10 | **BRIEFED** |
| 11 | Batch atomicity (RATIFIED; spec §32.23; code defects P-4/P-5) | §7 rows 11/12/17, T-22/T-23/T-26, F-17/F-18/F-21 | **BRIEFED** |
| 12 | Active Reservation delete prohibited (RATIFIED; spec §10.2) | §7 row 13, T-24, F-20, §22 "§10.2 → T-15/T-24" | **BRIEFED** |
| 13 | Complimentary/operational upgrade (RATIFIED; D-8, spec §10.8/§11/§19) | §7 rows 3 and 16, T-14/T-25, F-05/F-15/F-16 — **cross-note:** the *scheduled room-move* room-type path is unbriefed (RC-03) | **BRIEFED** |

**Nine Stage B rulings → owning tasks** (all present, none reopened):

| Ruling | Plan location | Verdict |
|---|---|---|
| D-0 quantity = 1 | T-11, T-12, §5 principle, §22 | **BRIEFED** |
| D-1 normative status table (§10.9) | T-09, T-11, §22 | **PARTIALLY BRIEFED** — T-11's classification list is incomplete (RC-12) |
| D-3 no-show release scope | T-16, F-07 | **BRIEFED** |
| D-6 overstay assert-before-commit | T-19 only (extend commands) | **PARTIALLY BRIEFED** (RC-02) |
| D-9 option A — one transaction, all paths | §9 "Paths to convert", T-02, §22 | **BRIEFED** (lists do not name Confirm/Guarantee or scheduled room move — folded into RC-01/RC-03) |
| D-15 idempotency + funded frontend | §11, T-04, T-27 … T-31 | **PARTIALLY BRIEFED** (RC-09, RC-10) |
| D-21 lazy first-touch population | §12, §13.1, T-07, T-10 | **PARTIALLY BRIEFED** (RC-08) |
| D-23 Reservations owns extend + reinstate | §15, T-32, T-33, T-34 | **PARTIALLY BRIEFED** — payload mapping undefined, dependency cycle (RC-13, RC-06) |
| D-24 lock order | §10, T-03/T-05/T-06/T-36, §22 | **BRIEFED** (task numbers in §10 itself are wrong — RC-05) |

D-18 correctly recorded as superseded by D-23 (plan T-32) `[VERIFIED FACT]`.

---

## 5. Task Inventory Coverage T-01 → T-37

All 37 tasks are present with the mandated 13-field format (`File · State · Change · Why · Spec ref · Deps · Tx · Locking · Idempotency · Tests · Migration · Risk`). File and line citations spot-checked and found accurate: `pipes.ts:88-104` (TransactionPipe), `cqrs.module.ts:37-41`, `command-bus.ts:20-27` (last-wins with warning), `availability-assertion.service.ts:50-196 / 199-249 / 251-297 / 299-417 / 419-448 / 450-470` (exact method boundaries), `:546-547` balance sort, `reservations.module.ts:262/263`, `front-office.module.ts:210/214`, `front-office.controller.ts:93/103`, `reservations.controller.ts:171/178`, `check-in-commit.service.ts:112` (SERIALIZABLE). `[VERIFIED FACT]`

**Coverage gaps (no task exists):**

- **Confirm / Guarantee** — routes `POST /reservations/:id/confirm` (`reservations.controller.ts:156`) and `:id/guarantee` (`:163`), handlers registered `reservations.module.ts:260/261`, both silent no-ops flagged by X-2 but assigned to no task; absent from §2, §4.2, §7, §9, §16.1, T-31 adopter list (RC-01). `[VERIFIED FACT]`
- **Checkout overstay branch** — `check-out.handler.ts:94-111` commits `departure_date = today` inside `withTransaction` with warn-only catch, no assertion; not in any task's file scope (RC-02). `[VERIFIED FACT]`
- **Scheduled room moves** — `schedule-room-move`, `execute-scheduled-room-move` (writes `roomType: move.to_room_type`), `cancel-scheduled-room-move`, `sweep-scheduled-room-moves`; plan contains **zero** occurrences of "room move" (RC-03). `[VERIFIED FACT]`
- **Hold creation / hold expiry** — F-03 and §7 row 1 require hold assertions; `InventoryReservationPort.holdInventory/confirmHold/releaseHold` has no consumer, no availability-hold table or column exists in the Phase 3 migrations, and no scheduled job exists (`@Cron` appears only in `common/outbox/outbox-processor.ts`) (RC-04). `[VERIFIED FACT]`

**Task-level citation errors** (folded into RC-05): T-03 `Tests: F-06 replay` and T-04 `Tests: F-05 idempotency rows` cite functional cases that contain no such assertions; T-05 `Tests: T-37 lock-order probe` should be T-36; T-04 `Deps: T-31 (scheduled leg)` and T-23 `Deps: T-31` should be T-30; T-12 `Deps: R2, R3, R4` references an undefined wave.

---

## 6. Reservation Lifecycle Matrix Review (spec §11 / §12 / §21 vs plan §7 / §16.1)

Spec §11 is a 23-row matrix (lines 199–221). Plan §7 has 18 rows. Row-by-row reconciliation:

| Spec §11 rows | Plan §7 row(s) | Status |
|---|---|---|
| 199/200 create confirmed / guaranteed | 1 Create | Covered |
| 201 create active hold | 1 Create (clause "hold ⇒ assert hold set") | **Clause present, mechanism missing** (RC-04) |
| 202 create no-availability PENDING | 1 Create (clause) | Covered |
| **203 PENDING hold → Confirm/convert** | — | **Missing** (RC-01) |
| **204 PENDING no-availability → Confirm** | — | **Missing** (RC-01) |
| 205 WAITLIST → Promote | 15 | Covered |
| 206/207 check-in (CONFIRMED/GUARANTEED) | 6 | Covered |
| 208/209 cancel | 4 | Covered (non-consuming cancel carve-out stated only in F-06) |
| **210 PENDING hold → Cancel or expire** | — | **Missing** (RC-04) |
| 211/212/213/214 cancel of non-consuming states | 4 (implicitly) | Covered implicitly |
| 215 normal checkout | 8 | Covered |
| 216 early departure | 7 | Covered |
| 217 extension | 9 | Covered for extend commands; **checkout-overstay missing** (RC-02) |
| 218 room-type upgrade | 3, 16 | Covered for `ChangeRoomTypeCommand` and FO upgrade; **scheduled room move missing** (RC-03) |
| 219/220 reinstatement | 10 | Covered |
| 221 auto-cancel sweep | 12 | Covered |

**Citation defect:** §7 row 14 (Waitlist join) cites `§11 r212`, which is the **WAITLIST → Cancel** row; spec §11 has no waitlist-join row (join is governed by §10.1/§32.9 non-consumption). X-9's "rows 205/212" has the same defect for the join half. `[VERIFIED FACT]` (RC-15)

**Spec §21 synchronicity rule** ("no path exempt because it is scheduled, batched, automated, or operationally initiated") is quoted in the plan's spirit but violated in coverage by RC-02/RC-03.

---

## 7. Transaction Boundary Review (D-9)

- §9 correctly states one interactive transaction per mutation path, `TransactionPipe` must pass `tx`, handlers must stop opening a second `$transaction`, isolation stays `READ_COMMITTED` (verified default at `transaction-manager.ts`), deadlock retry present (3 attempts, exponential backoff), events after commit, outbox excluded from inventory. `[VERIFIED FACT]`
- X-20/F-9 evidence verified: `TransactionPipe` calls `transactionManager.execute(async () => next())` and passes no `tx`; handlers open their own transactions. `[VERIFIED FACT]`
- "Paths to convert" (§9) omits Confirm/Guarantee and scheduled room move (RC-01/RC-03).
- **Verdict: BRIEFED**, with the two path-list omissions above.

## 8. Locking Order Review (D-24)

- Required order in §10 (`journal claim → reservation_availability_state → sorted balance locks → reservations row`) matches D-24 and spec §23 exactly; balance sort order verified compliant (`lockBalanceKeys` sorts `hotelId → roomType → stayDate`, then `FOR UPDATE`); `FOR UPDATE` today exists only on balances (`availability-assertion.service.ts:106,361,561,604`); `reservation_availability_state` is plain-read and `reservations` is never locked by the engine. `[VERIFIED FACT]`
- `assert()` (lines 50-196) has no production caller — T-08's retire/bypass decision is supported. `[VERIFIED FACT]`
- The known-violations row "concurrency spec `:120-129` claims outside the tx" is accurate; its fix is assigned to **T-37** (full matrix execution) where **T-35/T-36** own test edits (RC-05).
- **Verdict: BRIEFED.**

## 9. Idempotency Review (D-15)

Verified state: zero `IdempotentCommand` implementers; `IdempotencyPipe` marks **before** execute with 86400 s TTL and 409 on duplicate; `IdempotencyInterceptor` exists but is unregistered and reads `x-idempotency-key`; `idempotency_keys` keyed `(hotel_id, idempotency_key)`; `apps/web` sends no key while `Providers.tsx:21-23` sets `mutations: { retry: 1 }`. Plan tasks T-27 … T-31 and §11 correctly describe the required end state, and §16.2 states the replay/conflict/unknown-outcome expectations. `[VERIFIED FACT]`

**Gaps:**
- **Key propagation chain undefined** — no mechanism is specified for how the HTTP header reaches `command.idempotencyKey` (`request-context.ts` carries only tenant/property/user), and the three dedup layers (HTTP `idempotency_keys`, Redis `cmd:dedup` inside `IdempotencyPipe`, operation journal) have no stated responsibility split or precedence. `[ARCHITECTURAL DECISION REQUIRED]` (RC-09)
- **Frontend key minting point unspecified** — "stable across React Query retries" cannot be satisfied by minting inside `mutationFn` (re-run per attempt); the plan must name where the key is minted and carried. `[INFERENCE]` (RC-10)
- Batch/scheduled key composition is stated only for scheduled items (`hotelId:businessDate:jobType:itemId`); batch command keys are not stated (RC-09).
- **Verdict: PARTIALLY BRIEFED.**

## 10. State Population Review (D-21)

- §12/§13.1 correctly implement the ratified rule: assign on first port call, sticky, no backfill, "row absent ⇒ assign, do not throw"; T-10 partitions the legacy source by population (`LEGACY` or unassigned) **before** any assertion lands (§17 ordering constraint 2); X-13 dual-count risk is registered; gate G3/G5 exist. Verified that `setPopulationInTransaction` has zero production callers and that all 1,124 live Reservations have no state row. `[VERIFIED FACT]`
- **Ordering gap:** T-07 states "assignment occurs after step ②", but step ② is `SELECT … FOR UPDATE` on `reservation_availability_state`, a row that does not yet exist on first touch. The plan does not specify create-then-lock (or insert-as-lock), nor handling of the concurrent first-touch primary-key race (`PRIMARY KEY (reservation_id, hotel_id)`), although T-07 claims "idempotent by uniqueness of the state row". `[VERIFIED FACT]` (RC-08)
- **LEGACY semantics undefined:** T-07/§12 say "`ASSERTION_MANAGED` for port calls, `LEGACY` otherwise" but never state which code emits `LEGACY` (today: none), nor what a port call must do when it meets an existing `LEGACY` row (schema CHECK `population <> 'LEGACY' OR current_assertion_id IS NULL` means such a row can never carry an assertion). Refusing is fail-closed on lifecycle operations such as cancel; routing to legacy counters preserves §27's "legacy population stays on legacy authority". `[ARCHITECTURAL DECISION REQUIRED]` (RC-08)
- **Verdict: PARTIALLY BRIEFED.**

## 11. Legacy Coexistence Review

§13 correctly separates stop-use (Phase 3) from retirement (Phase 6): legacy `availability`/`inventory` counters, reconciliation view, GBA sources and every untouched LEGACY path remain in service; no table dropped, no column removed (G10 enforces it). Verified citations: `crs.releaseInventory` called from inside repository transactions at `reservation.repository.ts:609/640/673`; `crs.modifyReservation` at `:546`; `inventoryDomain.isAvailable` check-only at FO extend (`:58`) and upgrade (`:61`); mass-update raw `inventory` read; snapshot `reservations` source retained but partitioned. `[VERIFIED FACT]` **Verdict: BRIEFED.**

## 12. Migration and Data Readiness Review

- Drift state verified: **45 migrations, 18 pending, 4 drift**, last common `20260819000000_add_inv_audit_log`; both `20260927000000_availability_assertion_engine` and `20260928000000_availability_phase3_reservation_foundation` unapplied; `one_active_reservation_uq` present in the Phase 3 foundation migration. `[OBSERVED BEHAVIOR]`
- **The four drift migrations exist nowhere in the working tree** — `packages/db/prisma/migrations/` no longer exists and the four folders are absent from `packages/db/migrations/`; they survive only in git history (worktree deletions). "Reconcile" therefore requires restoring those four migration folders from git into `packages/db/migrations/` before `migrate status` can be clean. T-01 names the four files but not the mechanism or source. `[VERIFIED FACT]` (RC-11)
- **T-01's apply command is unsafe as written:** `pnpm db:push` runs `prisma db push` (schema-sync that can reset/drop under drift) and root `pnpm db:migrate` runs `prisma migrate dev` (may prompt a reset when drift is detected). The safe command is `prisma migrate deploy` (available as `packages/db` → `migrate:deploy`; there is **no** root `db:deploy` script). `[VERIFIED FACT]` (RC-11)
- Gates G1 (status clean, 5 tables present) and risk R-1 (maintenance window + snapshot) are appropriate. **Verdict: PARTIALLY BRIEFED.**

## 13. Command Ownership Review (D-23)

- §15 and T-32 correctly identify the live collision: `ExtendStayCommand` and `ReinstateReservationCommand` are each registered twice (`reservations.module.ts:262/263`, `front-office.module.ts:210/214`), `CommandBus.register` warns then overwrites (last-wins), so Front Office wins and the Reservations routes dispatch an incompatible payload (`command.id`/`hotelId` undefined in the FO handler). Permission names in T-33 match the decorators actually used (`RESERVATION_EXTEND` at `reservations.controller.ts:178`, `RESERVATION_REINSTATE` at `:171`); FO routes `:93`/`:103` have no `@Permission`. `[VERIFIED FACT]`
- **Payload mapping undefined:** Reservations `ExtendStayCommand(id, hotelId, newDepartureDate, userId)` / `ReinstateReservationCommand(id, hotelId, userId)` versus FO `ExtendStayCommand(reservationId, extraNights, reason?, userId)` / `ReinstateReservationCommand(reservationId, newRoomNumber, reason, userId)`. "FO service maps its DTO onto the Reservations payload" does not state where `hotelId` comes from, how `extraNights` becomes `newDepartureDate` (requires reading current departure + date arithmetic), or what happens to `reason` and `newRoomNumber` (both currently feed `logReservationChange` / room reassignment). Silent dropping changes Front Office behaviour. `[VERIFIED FACT]` (RC-13)
- Dependency cycle with T-19/T-20 → T-33 → T-32 (RC-06). **Verdict: PARTIALLY BRIEFED.**

## 14. Automated, Batch and Scheduled Path Review (auto-cancel included)

- **Auto-cancel (T-23):** evidence verified — raw `$queryRawUnsafe` selection, separate `$executeRawUnsafe` status write, no transaction, bypasses `repo.cancel`, no status guard, `take: 1000` truncation, `?? 'SYSTEM'` unreachable because `AuthorizationPipe` requires `userId`. T-23's required end state (per-item transaction, exact-assertion release + `CANCELLED` in one tx, status guard for re-run idempotence, selection policy kept outside the domain contract) matches spec §11 auto-cancel row and D-9. `[VERIFIED FACT]` **Briefed.**
- **Batch / mass update (T-22, T-26):** `batchUpdateStatus` is findMany+updateMany with no transaction; mass-update is check-then-act with 1,000-row truncation. Plan requires per-item interactive transactions, explicit partial results, and surfacing truncation rather than silent dropping — consistent with spec §32.23/§20. `[VERIFIED FACT]` **Briefed.**
- **Scheduled identity (T-30):** correctly scoped to a service principal + deterministic per-item key; no cron/scheduler is introduced (Night Audit orchestration stays deferred). `[VERIFIED FACT]` **Briefed.**
- **Scheduled room moves:** not covered (RC-03). **Scheduled confirm/guarantee:** no scheduled job exists. `@Cron` occurs only in outbox plumbing. **Verdict: BRIEFED except RC-03.**

## 15. Testing Strategy Review

Against the required case list:

| Required case | Plan coverage |
|---|---|
| create / modify / cancel / no-show | F-01, F-04, F-06, F-07 ✓ |
| early departure / checkout / overstay | F-08, F-09, F-10/F-11 (extend commands only — **checkout overstay missing**) |
| reinstatement success/failure | F-12, F-13 ✓ |
| check-in / room transfer / upgrade | F-14, F-15, F-16 ✓ |
| auto-cancel / batch / mass update | F-17, F-18, F-21 ✓ |
| waitlist join / promote / delete | F-19, F-20 ✓ |
| **confirm / guarantee** | **missing** (RC-01) |
| **scheduled room move** | **missing** (RC-03) |
| **hold creation + hold expiry** | F-03 states it, but no mechanism/task owns expiry (RC-04) |
| concurrency (Case C, cancel+extend, extend+modify, reinstatement, uniqueness, lock order) | §16.3 ✓ (Case C target `:253-303` verified correct) |
| idempotency (duplicate/retry/failure/payload/scheduled/command) | §16.2 ✓ but **no case IDs** (caused the T-03/T-04 miscites) |
| tenant/property isolation | §16.4 ✓ (spec §26) |
| fail-closed suite | §16.5 ✓ incl. missing-state must assign |
| baseline discipline | §16.6 ✓ — 8 failures / 3 suites reproduced |
| §16 runner line | `pnpm test` "with `--maxWorkers=1 --max-old-space-size=8192`" is not directly runnable: `pnpm test --maxWorkers=1` is rejected by pnpm, and `--max-old-space-size` is a Node flag, not a Jest option. Verified working invocation: `npx jest --maxWorkers=1` from `apps/api` with `NODE_OPTIONS=--max-old-space-size=8192`, plus `AVAILABILITY_TEST_DATABASE_URL` for Postgres-gated suites. `[OBSERVED BEHAVIOR]` (REC-02) |

**Verdict: PARTIALLY BRIEFED.**

## 16. Conflict Register Disposition (X-1 → X-22)

No conflict contradicts the locked specification; none was silently adapted; each is either resolved by a task, deferred by rule, or mis-referenced. Disposition:

| ID | Disposition | Task cited | Correct task |
|---|---|---|---|
| X-1 | Resolved | T-12 | T-12 ✓ |
| X-2 | **Requires clarification** — confirm/guarantee have no task | T-13/14/20/21 | T-14, T-19, T-20 + **missing tasks** (RC-01) |
| X-3 | Resolved | T-13 | T-13 ✓ |
| X-4 | Resolved | T-15/16/24 | ✓ |
| X-5 | Resolved | T-22 | ✓ |
| X-6 | Resolved | T-23 | ✓ |
| X-7 | Resolved (early departure) | T-18 | **T-17** (RC-05) |
| X-8 | Resolved | T-20 | **T-19** (RC-05) |
| X-9 | Resolved | T-25 | **T-21**; spec citation r212 wrong (RC-05/RC-15) |
| X-10 | Resolved | T-05/06/07 | **T-03, T-05, T-06** (RC-05) |
| X-11 | Resolved | T-02 | port consumers = R5 lifecycle tasks / R4 wiring (RC-05) |
| X-12 | Resolved | T-09 | **T-07** (RC-05) |
| X-13 | Resolved | T-10 | T-10 ✓ |
| X-14 | Resolved | T-11 | T-11 ✓ |
| X-15 | Resolved | T-28/29/30/32 | T-32 → **T-31** (RC-05) |
| X-16 | Resolved | T-33/34 | **T-32, T-34** (RC-05) |
| X-17 | Resolved | T-31 | **T-30** (RC-05) |
| X-18 | Resolved | T-21 | **T-20** (RC-05) |
| X-19 | Resolved | T-24 | T-24 ✓ |
| X-20 | Resolved | T-03 | **T-02** (RC-05) |
| X-21 | Resolved (IN_HOUSE) | "T-11b" — **task does not exist** | **T-09** (RC-05) |
| X-22 | Deferred by rule (spec §3) | — | ✓ |

**Missing register entries:** checkout-overstay branch (D-6 location) and scheduled room moves should be added as conflicts (RC-02, RC-03).

**Contradiction check:** T-18's scope statement ("same handler, `earlyDeparture=false` → Change: none") as written covers the overstay extension branch, which the locked specification (§10.5, ratified D-6) requires to assert before committing dates. As scoped, T-17/T-18 would leave a non-conforming path in place. `[INFERENCE]` — resolved by RC-02.

## 17. Dependency and Wave Review (R0 → R8)

§17 defines R0–R8 with five ordering constraints, all sound. §21 groups tasks under wave headers **R2, R3, R5, R6, R7, R8 only**.

- **Cycle A:** T-19 `Deps: T-02…T-06, T-33` → T-33 `Deps: T-32` → T-32 `Deps: T-19 and T-20` (same cycle via T-20). `[VERIFIED FACT]`
- **Cycle B:** T-04 `Deps: T-03, T-31` → T-31 `Deps: T-27` → T-27 `Deps: T-04`; T-23 `Deps: T-31` also enters this cycle. `[VERIFIED FACT]`
- **Undefined wave R4:** referenced by §17 (R4 — Wiring, C-10/C-11) and by T-12 `Deps: R2, R3, R4`, but §21 contains no "Wave R4" header and no task declares R4. Similarly R0 and R1 have no task rows (R1's content, migration deployment, is T-01, which sits under the "Wave R2 — Foundation" header although §17 assigns it to R1). `[VERIFIED FACT]`
- Resolved deps verified correct: T-01 none, T-05→T-03, T-06→T-02/T-05, T-10→T-01/T-07, T-11→T-09, T-13→T-02/T-04, T-14→T-13, T-15/16/17/21/22/24/25/26 consistent with §17 R5 order, T-28→R1, T-29→T-28, T-33/T-34→T-32, T-35→T-05/T-06/T-13/T-15, T-37→all.

**Verdict: NOT BRIEFED as an executable sequence** until RC-06 and RC-07 re-sequence the two cycles and define R4 (and R0/R1 task ownership).

## 18. Deferred Items Review (§20)

All 12 deferred items were checked against the task list and remain deferred — none has been pulled into scope: legacy counter retirement (Phase 6), population cutover (Phase 6), `crs-engine` removal from untouched paths (Phase 6), GBA/allotment (Phase 4), `PROSPECT` reference row (no task; T-09 handles `IN_HOUSE` only), `changeRate` (X-22 DEFER, no task), Night Audit orchestration beyond durable identity (T-30 is identity only), direct `CONFIRMED → CHECKED_OUT` discrepancy, hotel-local cutoff policy, C-6 (recorded, not resolved, Reservations track), cross-property transfer (ADR-072, no task), frontend amendment dialogs/Quick Book (T-29 is key generation only). §13.2 correctly distinguishes "stop-use" (in scope) from "retirement" (deferred). G10 enforces the non-goals. `[VERIFIED FACT]`

**Verdict: BRIEFED.**

---

## 19. Required Plan Clarifications (listed — **not applied**)

### Blocking (must be applied to produce plan v0.2 before any task executes)

| ID | Clarification required | Affected plan sections / tasks | Why it matters | Label |
|---|---|---|---|---|
| **RC-01** | Add Confirm and Guarantee to scope, lifecycle map, paths-to-convert, task list, F-matrix and T-31 adopters (assert-before-consuming per spec §11 r203/204 and §28 "Confirm / Guarantee / Promote Reservation"); fix X-2's task mapping | §2, §4.2, §7 (new rows), §9, §16.1 (new F-cases), X-2, T-31, §22 "§11 → T-12…T-26" | Two live routes dispatch silent no-ops today; spec requires assertion before the consuming state commits. Without a task they ship unimplemented | `[VERIFIED FACT]` |
| **RC-02** | Bring `check-out.handler.ts:94-111` (overstay auto-extend — the exact D-6 ratified defect) into scope: add an X-register entry, name the owning task (extend T-17/T-19), restrict T-18's "Change: none" to the true normal-checkout path, and add an F-case (assert added night before `departure_date` commit; failure leaves dates unchanged) | §4.4, §7 row 8/9, T-17, T-18, T-19, §16.1, §9 | As scoped, the plan would leave a spec §10.5/D-6 conflict unaddressed; also auto-posts a ROOM_CHARGE with no inventory assertion | `[VERIFIED FACT]` + `[INFERENCE]` |
| **RC-03** | Add scheduled room moves (`schedule-room-move`, `execute-scheduled-room-move` — writes `roomType: move.to_room_type`, `cancel-…`, `sweep-…`) to scope, §7, a task and an F-case as an atomic old-to-new room-type replacement (spec §10.8/§11 r218, §343 "no path exempt because it is scheduled") | §2, §7, §4.4, §21 (new task), §16.1, §9 | A live path mutates committed `room_type` with no replacement; plan has zero "room move" occurrences | `[VERIFIED FACT]` |
| **RC-04** | State the hold position: (a) whether any Phase 3 path creates an availability hold today (`InventoryReservationPort.holdInventory` has no consumer; no hold table/column exists), (b) who owns hold **expiry** (no scheduled job exists; T-30 is identity only), (c) re-scope or supply F-03's "expiry releases it" clause, (d) cover §11 r210 (PENDING hold → Cancel or expire) | §7 rows 1/4, F-03, F-19, T-12, §21 (possible new task), §16.1 | F-03 as written tests a mechanism that does not exist; hold expiry timing is not defined by the locked specification | `[VERIFIED FACT]` + `[BUSINESS DECISION REQUIRED]` (expiry policy) |
| **RC-05** | Correct the cross-reference drift (~35 items): §4.4 X-7→T-17, X-8→T-19, X-9→T-21, X-10→T-03/05/06, X-11→port consumers, X-12→T-07, X-15→T-31 (not T-32), X-16→T-32/T-34, X-17→T-30, X-18→T-20, X-20→T-02, X-21→T-09 (delete "T-11b"), X-2→T-14/T-19/T-20 + new tasks; §5 diagram ①→T-28/29/30, command→T-31, ②→T-02, ③→T-03, ④→T-05, ⑥→T-06, population→T-07; §10 ①→T-03, ②→T-05, ④→T-06, test-fix→T-35/36; §11 HTTP→T-28, retry→T-29, command→T-27, adopters→T-31, scheduled→T-30; §12 + §14 first-touch "T-09"→T-07; R5 contract note "IdempotentCommand (T-21)"→T-31; T-04/T-23 `Deps` T-31→T-30; T-03/T-04/T-05 `Tests` lines | §4.4, §5, §10, §11, §12, §14, §21 (task lines) | An implementer following §4.4/§5/§10/§11 would execute the wrong tasks; §21 `Deps` and §22 are correct and must become the single source | `[VERIFIED FACT]` |
| **RC-06** | Break the two dependency cycles by re-sequencing: (A) T-19/T-20 ↔ T-33 ↔ T-32 — e.g. make T-32 depend on the *payload mapping contract* rather than on completed T-19/T-20, or invert T-33's dependency; (B) T-04 → T-31 → T-27 → T-04 — e.g. T-04 (derivation) must precede T-27/T-31 | §21 T-04, T-19, T-20, T-23, T-27, T-31, T-32, T-33; §17 | No topological execution order exists; waves cannot be run as written | `[VERIFIED FACT]` |
| **RC-07** | Align §17 and §21 waves: define the task rows for R4 (wiring, C-10/C-11), state that R0/R1 are process gates, and move/relabel T-01 under R1 (or state that T-01 is R1 executed at the head of the R2 header). Reconcile T-12 `Deps: R2, R3, R4` accordingly | §17, §21 headers, T-01, T-12 | Tasks reference a wave that does not exist; §17 ordering constraint 1 cannot be traced to a task | `[VERIFIED FACT]` |
| **RC-08** | Specify D-21 first-touch mechanics: create-then-lock (or insert-as-lock) so step ② has a row to lock, ordering versus ① journal claim and ⑥ `reservations` lock, concurrent first-touch PK-race handling, and the LEGACY semantics — (i) state that no Phase 3 code assigns `LEGACY` (no-row is the LEGACY representation) or name the code that does, and (ii) define port behaviour when an existing `LEGACY` row is met (refuse fail-closed vs route to legacy authority per §27/§13.2) | T-05, T-07, §10, §12, §13.1 | First touch is the highest-risk change (T-07 risk HIGH); without this, implementations either deadlock logically or block lifecycle operations | `[VERIFIED FACT]` + `[ARCHITECTURAL DECISION REQUIRED]` |
| **RC-09** | Define the idempotency key chain: how the HTTP `Idempotency-Key` header reaches `Command.idempotencyKey` (RequestContext has no such field), the responsibility/precedence of the three layers (HTTP `idempotency_keys`, Redis `cmd:dedup`, operation journal), and batch command key composition (scheduled key is defined; batch is not) | §11, T-04, T-27, T-28, T-31 | D-15 requires end-to-end dedup; today the HTTP layer and the command layer cannot see each other | `[ARCHITECTURAL DECISION REQUIRED]` |
| **RC-10** | State where the frontend mints the key so it survives a React Query retry (mint per logical mutation — e.g. in the mutation variable or `onMutate` — not inside `mutationFn`), and how it is discarded after completion | T-29, §11, §16.2 | With `mutations.retry: 1`, a per-attempt key defeats dedup, silently invalidating the funded D-15 frontend leg | `[INFERENCE]` |
| **RC-11** | Pin T-01's commands: apply with `prisma migrate deploy` only (root `pnpm db:push` = `db push`, root `pnpm db:migrate` = `migrate dev` — both unsafe with 18 pending + 4 drift); forbid `db:push`; specify the drift-reconciliation mechanism (restore the four migration folders from git into `packages/db/migrations/`, since they exist nowhere in the worktree) and the pre/post verification (`migrate status` clean, 5 tables present) | T-01, §14, §17 R1, G1, R-1 | `db:push`/`migrate dev` against a drifted database can reset or destroy data; "reconcile" has no stated mechanism | `[VERIFIED FACT]` |
| **RC-12** | Complete T-11's status classification against spec §10.9: state explicitly that `PROSPECT` is canonical **non-consuming** (rule: it never appears as "unclassifiable"), how a hold-context `PENDING` is distinguished from no-availability `PENDING` in the consumption source (current adapter counts **all** `PENDING` as consuming), that `NO_SHOW` retains the elapsed arrival night (§10.6) for that date, and that `CHECKED_OUT` **holds** its committed range without releasing (§10.9 rule 4) — i.e. whether such rows are counted in the legacy source and how that reconciles with assertion balances for the same status | T-11, §16.1 F-cases, §22 "§10.9" | Current code returns UNRESOLVED for `NO_SHOW`/`CHECKED_OUT`; T-11's list resolves them without stating the retained-commitment rules, risking a silent under-count versus balances | `[VERIFIED FACT]` + `[INFERENCE]` |
| **RC-13** | Specify the D-23 payload mapping: source of `hotelId` for the Reservations commands, `extraNights` → `newDepartureDate` (which read performs the date arithmetic), and the disposition of `reason` and `newRoomNumber` (drop, preserve in the command, or route to `logReservationChange`) | §15, T-32 | Silent field dropping changes Front Office behaviour (change-log reason, room reassignment) and the FO routes cannot construct the owner command without `hotelId` | `[VERIFIED FACT]` |
| **RC-14** | Extend the test matrix: add F-cases for Confirm, Guarantee, hold expiry (once RC-04 resolves its owner), scheduled room move and checkout overstay; add identifiers to §16.2/§16.3/§16.4/§16.5 cases so tasks cite them correctly | §16.1, §16.2, §16.3, §16.4, §16.5, T-03, T-04 | G8 requires §16 "fully green" — gaps mean untested locked clauses; missing IDs already caused two miscites | `[VERIFIED FACT]` |
| **RC-15** | Correct evidence/metric defects: F-1 says the port is "imported by ReservationsModule" — `reservations.module.ts` does **not** import `RESERVATION_AVAILABILITY_PORT` (substance "zero production consumers" unchanged); §7 row 14 and X-9 cite spec r212 (WAITLIST→Cancel) for waitlist **join**; Self-Review rows "20 MUST + 2 DEFERRED of 22", "EXISTING … not converted into tasks" vs T-18, and "Unresolved dependencies: 3" | §4.1 F-1, §4.4 X-9, §7 row 14, Self-Review | The plan's own audit trail must be internally true before it can gate implementation | `[VERIFIED FACT]` |

### Recommended (non-blocking)

- **REC-01** — Assign stable IDs to the §16.2/§16.3/§16.4/§16.5 cases (e.g. I-01…I-06, CN-01…, ISO-01…, FC-01…) so task `Tests:` lines can cite them without misciting F-cases.
- **REC-02** — Replace §16's runner line with the verified invocation: from `apps/api`, `set NODE_OPTIONS=--max-old-space-size=8192 && npx jest --maxWorkers=1`; Postgres-gated suites additionally require `AVAILABILITY_TEST_DATABASE_URL`.

---

## 20. Confirmations and Final Verdict

**Confirmations (explicit):**

1. **No Stage B decision was reopened.** All 13 decision-sheet rows and all nine rulings (D-0, D-1, D-3, D-6, D-9, D-15, D-21, D-23, D-24; D-18 recorded as superseded) are treated by the plan as ratified inputs. The clarifications in §19 are plan-level only; none proposes a different business rule. `[VERIFIED FACT]`
2. **Stage C remains LOCKED.** `availability-phase3-domain-specification.md` (v0.3, LOCKED 2026-09-28) was not modified by this review and must not be modified to accommodate RC-01 … RC-15. Where the plan under-covers a locked clause (RC-01, RC-03, RC-04, RC-12), the specification stands and the plan must be extended. `[VERIFIED FACT]`
3. **No implementation was performed.** No production code, schema, migration, test, route, frontend file or Prisma model was created or changed. `prisma migrate status`, file reads and one baseline `jest` run were the only repository commands executed. `[VERIFIED FACT]`
4. **The Implementation Plan was not modified during this first review pass.** RC-01 … RC-15 are listed for a separate amendment pass. `[VERIFIED FACT]`
5. **No Reservations-track scope was introduced.** C-6 remains recorded-not-resolved; no reservation state machine, DTO, or Reservations domain redesign is proposed; ADR-072 cross-property transfer remains deferred. `[VERIFIED FACT]`
6. **Scope control held.** Non-goals (§3) intact: no schema authoring, no legacy retirement, no Phase 1/2 spec authoring, no new quantity/status field, no new authority. G10 enforces this at gate time. `[VERIFIED FACT]`

**Conditions for the verdict:**

- Apply **RC-01 … RC-15** to `availability-phase3-implementation-plan.md`, re-issue as **v0.2** with an updated Self-Review (corrected metrics, corrected cross-references, cycles broken, R4 defined), and re-submit for approval.
- Fold **REC-01/REC-02** into the same amendment if convenient; they do not block.
- Re-verify the amended plan's §21 `Deps` graph is acyclic and that every spec §11 row maps to at least one task and one F-case.
- Only then may R0 begin.

**FINAL VERDICT:**

> **READY WITH REQUIRED PLAN CLARIFICATIONS.**
>
> The Phase 3 Implementation Plan is decision-faithful, authority-correct and structurally fit for execution, but it is **not yet approved for implementation**. Fifteen required clarifications (RC-01 … RC-15) must be applied and the plan re-approved. No task in the plan may be executed before that approval.
>
> **NOT READY FOR IMPLEMENTATION** would apply if any conflict with the locked specification could not be resolved inside the plan; no such unresolvable conflict was found — RC-02 and RC-12 are scoping/completeness defects in the plan that the plan itself can correct.

---

*Stage D Readiness Review complete. Next step: plan amendment (v0.2) applying RC-01 … RC-15, then re-approval. Implementation remains prohibited until then.*

# XYLO Availability Phase 4 — Stage 5: Implementation Readiness Review

**Phase:** 4 — GBA / Allotment Integration
**Stage:** 5 — Implementation Readiness Review (pre-implementation gate)
**Date:** 2026-09-30
**Document type:** Readiness review (READ-ONLY — no code, schema, migration, API, test, or data changes were made)
**Output file:** `docs/availability/phase-4/15_READINESS_REVIEW.md`
**Path note:** the stage instruction names `doc/avability/phase04/…`; the repository's actual path is `docs/availability/phase-4/` (verified — `doc/avability/phase04` does not exist). The instruction's intent is honored at the real path.

| Metric | Result |
|---|---|
| **Final verdict** | **READY WITH NON-BLOCKING NOTES** |
| Blockers | **0** |
| Non-blocking notes | **6** |
| Implementation details | **13** (DEF-1…DEF-10 + BLK-1/2/3 confirmations) |
| False positives / already covered | **10** |
| Task / rule / decision / confirmation validation | **70/70 · 97/97 · 22/22 · 6/6** (all independently verified; 2 matrix-wording gaps noted) |
| Stage 6 entry | **MAY BEGIN**, subject to §27 entry conditions |

---

## 1. Executive Summary

**Central question answered:** *Can an implementation agent begin executing the 70-task Phase 4 Implementation Plan without encountering an unresolved architectural, domain, dependency, data-model, transaction, API, migration, or integration blocker?*

**Answer: Yes.** No genuine evidence-based blocker was found. Every concern encountered during this review classified as one of: (a) a non-blocking documentation/wording/impact note, (b) an implementation detail the plan itself already routes to a scheduled Readiness-Review decision point or to implementation (DEF-1…DEF-10, BLK-1/2/3), or (c) an already-covered concern (the plan, or the locked decisions behind it, explicitly address it).

Basis of the verdict:

1. **No unresolved business decisions.** 22/22 decisions resolved and 6/6 confirmations recorded (D-2=A, D-4=B/C, D-6a=A, D-10=B, S-1=A, S-3=A) — independently re-verified against `12_DECISION_STATUS.md` §2b/§3/§4 and the spec header, not trusted from counters.
2. **97/97 target rules traceable** (95 DECIDED → tasks/tests; 2 DEFERRED → plan §13 DEF-1/DEF-2), with a matrix-wording gap on TR-10.3/TR-4.3 (NB-3) whose semantic coverage is intact.
3. **All 70 tasks (T-01…T-70) audited**: no missing/duplicate/orphan task, no circular dependency, no wrong ordering, no task depending on a non-existent component (every referenced path/handler/adapter/harness/interceptor/consumer spec spot-verified on disk), no task depending on an unresolved business decision, and every task carries verifiable acceptance criteria (Files · Prereq · Action · Expected · Tests · Verify · Rollback — one field-level nit, NB-6).
4. **Repository and live-DB reality re-verified** (read-only): GBA tables live, `block_type`/wash/attrition stores absent, `shoulder_days_*` live-but-undeclared, `allotment_pickups` live-and-undeclared — exactly the states the plan documents as BLK-1/2/3, FIND-1/4. Two new hygiene findings were added to the register (NB-4 migration bookkeeping, NB-5 dormant PUT caller).
5. **The plan closes the audit's risk findings**: partial commits (T-56/57/60), missing version predicates (T-56 seam), divergent availability numbers (T-38/39/40 + single-authority rules), unscoped SQL (T-59 + T-24), fabricated ids (T-22), missing `reservation.checked_out` (T-30/33), consumer `default: warn` for GBA events (T-40/63), D-11 fuzzy guest binding (T-25).
6. **Deferred items stayed deferred.** All 10 plan deferrals (scheduler transport, lock mechanism, assertion-port shape, ledger ownership, guest caller contract, confirmation semantics, idempotency-header mechanics, migration CI wiring, EXPIRED mechanism, metrics vehicle) are mechanism/tooling-shaped, constrained by the Final Domain Specification, and resolvable without another business decision — they are **not** converted to blockers (§23).

**The 6 non-blocking notes** (NB-1…NB-6) are: plan current-state count deltas; a spec self-check wording inconsistency; a §14 matrix 95/97 literal-naming gap; pre-existing `_prisma_migrations` duplicate records; a dormant frontend PUT release caller (no active production caller); and plan-internal cross-reference nits (3 omitted graph edges + T-55 Files field + event-class create-ownership overlap). None prevents an implementation agent from starting Phase A.

---

## 2. Review Scope

**Performed:** a structured pre-implementation gate across the domain contract, the implementation plan, the repository, and the live database, per the Stage 5 instruction's sections 4–24.

- **Validated:** `13_FINAL_DOMAIN_SPECIFICATION.md` internal consistency and decision/confirmation/rule representation; all 70 tasks of `14_IMPLEMENTATION_PLAN.md` (fields, dependencies, ordering, implementability); all 11 workstreams (WS-01…WS-11); repository paths/modules/handlers/adapters/hooks/controllers cited by the plan; `schema.prisma`, migration history, and live-DB structures; hotel isolation; reservation, availability, pickup/voucher, wash/release, shoulder, attrition, concurrency, event/outbox, legacy-containment, API/frontend, test, and migration/cutover/rollback readiness; deferred items; full traceability (97/22/6/70).
- **Not performed (locked stages):** no forensic audit repeated; no business decision reopened; no spec or plan rewritten; no fixes implemented; no code/schema/migration/API/test/data changes; no new rollout or architecture designed.
- **Method:** document read-through (spec 854 lines, plan 1114 lines, decision register 112 lines, rules doc), independent re-counting of TRs/decisions/confirmations/tasks rather than trusting completion reports, three parallel repository verification passes (GBA module, availability/events/FO, frontend/schema/migrations), targeted spot-checks on every load-bearing citation, and read-only live-database inspection via `information_schema`/`pg_indexes`/`_prisma_migrations` (SELECT only).

---

## 3. Authoritative Inputs

**Primary contract:** `docs/availability/phase-4/13_FINAL_DOMAIN_SPECIFICATION.md` (FINAL — treated as authoritative; source-of-truth order per its §0 header).

**Implementation contract:** `docs/availability/phase-4/14_IMPLEMENTATION_PLAN.md` (COMPLETE — the object under review).

**Supporting evidence (read/verified as needed):** `01_FORENSIC_AUDIT.md`, `02_CURRENT_ARCHITECTURE.md`, `03_DATA_MODEL_AUDIT.md`, `04_AVAILABILITY_INTEGRATION_MATRIX.md`, `05_BUSINESS_RULES_CURRENT_STATE.md`, `06_CONCURRENCY_AND_TRANSACTION_AUDIT.md`, `07_LEGACY_DEPENDENCY_MAP.md`, `08_OPEN_DECISIONS.md`, `09_PHASE_4_AUDIT_SUMMARY.md`, `10_DECISION_RESOLUTION.md`, `11_TARGET_BUSINESS_RULES.md` (97 TRs), `12_DECISION_STATUS.md` (22 items, 6 confirmations).

**Implementation reality inspected:**

- `apps/api/src/modules/group-allotment/` (aggregates, entities, value objects, 25 command handlers, 5 query handlers, controllers, adapters, repositories, events, permissions, ports).
- `apps/api/src/modules/availability/` (source adapter, snapshot/assertion services, reconciliation service, lock-order spec).
- `apps/api/src/modules/front-office/…/check-out/check-out.handler.ts`, `apps/api/src/modules/activities/availability-sales.controller.ts`, `apps/api/src/modules/shared/events.consumer.ts`, `common/database/unit-of-work/`, `common/events/event-bus.ts`, `common/outbox/`, `platform/audit/audit-subscriber.service.ts` (path confirmed at `src/platform/audit/`, not under `modules/`), `core/interceptors/idempotency.interceptor.ts`.
- `apps/web/features/group-allotment/` (api, hooks, views, components) and `apps/web/components/AvailabilitySales.tsx`.
- `packages/db/schema.prisma`, `packages/db/migrations/` (49 directories), live Postgres `xylo_cloud` (read-only).

---

## 4. Stage Gate Status

| # | Stage | Status | Evidence | Carried into Stage 5 |
|---|---|---|---|---|
| 1 | Phase 4 Forensic Audit | **CLOSED** | `01`–`09` documents | Findings F-x/C-x/A-x/L-x/B-x treated as fixed evidence |
| 2 | Business Rules / Decision Resolution | **CLOSED** | `10`–`12`; 22/22 decisions, 6/6 confirmations, 0 `USER DECISION REQUIRED` | Re-verified independently (§6): holds |
| 3 | Final Domain Specification | **FINAL** | `13_…` §21.3 declaration; 0 contradictions per §21.2 | Re-verified (§6): one wording inconsistency found (NB-2), zero rule contradictions |
| 4 | Implementation Plan | **COMPLETE** | `14_…` Final Quality Gate A–H; 70 tasks | Re-verified (§7): holds with 6 non-blocking notes |
| 5 | **Implementation Readiness Review** | **COMPLETE — this document** | Verdict §26 | — |
| 6 | Implementation | **NOT STARTED — may begin** per §26/§27 | — | Stage 6 will be a separate instruction |

No stage was restarted, repeated, or re-litigated.

---

## 5. Repository Reality Check

Goal: confirm the plan is executable against the *actual* codebase — nonexistent paths, renamed/deleted components, stale references, wrong ownership, wrong API surface, wrong schema model, missing dependencies, architecture mismatch.

### 5.1 Verified-exist (plan citation → on-disk result)

| Plan citation | Result |
|---|---|
| GBA module: 3 aggregates, 4 entities, 7 VOs, 22 event classes, 3 repositories, 2 controllers, `GROUP_BLOCK_WASH` permission | ✅ exact counts verified (`domain/aggregates`×3, `entities`×4, `value-objects`×7, `group-allotment.events.ts` = 22 `export class`, `infrastructure/repositories`×3, `permissions/group-allotment.permissions.ts:14`) |
| Command/query handler inventory | ⚠ **NB-1**: 25 command handlers exist (plan §1.1 says 24); 5 query handlers exist (plan says 6); `infrastructure/adapters` = 3 (plan §1.1 says 4) |
| Frontend `features/group-allotment/` hooks ×40, views ×5, components ×4 (plan §3 row 11) | ⚠ **NB-1**: 36 exported hooks (1 file), 4 views, 4 components (components counts match; hooks/views do not) |
| Handlers cited by name (create/cancel-allotment-pickup, consume/cancel-allotment-voucher, create-group-pickup, add/remove-shoulder-days, release-*, set-quota, set-daily-allocation, delete-*, lifecycle/attrition services, `prisma-reservation-association.adapter.ts`, `prisma-folio.port.ts`, `prisma-billing-instruction.adapter.ts`) | ✅ all exist at cited locations |
| Availability: `availability-source.adapter.ts` (filters `:49-52`, eligibility `:66`, overbooking `:52`, `allotmentStopSaleActive: false` `:113`, clamp `:116`), `availability-snapshot.service.ts`, `availability-assertion.service.ts`, `availability-reconciliation.service.ts`, `availability-lock-order.postgres.spec.ts` | ✅ all exist; adapter facts re-verified |
| Events/outbox: `event-bus.ts` (`publish(event, tx)`, `maxRetries`), `common/outbox/*`, `events.consumer.ts` (job.id dedup `:40-43`, rethrow `:61-63`, `default: warn`), `modules/shared/__tests__/events.consumer.spec.ts` | ✅ exists (spec file confirmed — plan's "extend existing" is valid) |
| Infra primitives: `common/database/unit-of-work/` (`unit-of-work.ts` etc.), `core/interceptors/idempotency.interceptor.ts`, `infrastructure/bullmq/`, `platform/audit/audit-subscriber.service.ts` | ✅ all exist (audit subscriber at `src/platform/audit/` — plan's relative path resolves correctly) |
| Dead port replacement target `inventory-commitment.port.ts` | ✅ exists at `group-allotment/domain/ports/` (plan calls it dead-port — correct target for T-26/T-41) |
| Reservation/FO: `reservation-domain.events.ts`, `reservation.repository.ts`, `check-out.handler.ts` (raw pickup UPDATEs `:392`/`:481`, events `:254`/`:263`), `reservation-status.service.ts` (DEF-6) | ✅ all exist |
| Frontend: `group-allotment.api.ts:29` (GET available-rooms), `:146` (PUT release), `use-group-allotment.ts` hook lines `:101/:112/:200/:213/:305/:381/:401/:422`, views `AllotmentDetailView.tsx:663-720`, `GroupBookingDetailView.tsx`, `AvailabilitySales.tsx`, web `jest.config.js` | ✅ all verified (hook line numbers exact) |
| Test harness `availability-postgres.harness.ts` (`splitSqlStatements`, ReadCommitted, env-gated) | ✅ exists; `AVAILABILITY_TEST_DATABASE_URL` currently unset (Stage 6 setup item, §21) |
| Non-existent audit-era files (`AllotmentDetail.tsx`, `AllocationStatus.tsx`, `GroupBookingPanel.tsx`) confirmed removed and untargeted | ✅ absent; no task references them |
| Empty feature dirs `components/allotment|group-block|group-booking` exist (intentional targets) | ✅ present |

### 5.2 Live-database re-verification (read-only, 2026-09-30)

| Check | Result |
|---|---|
| 9 new-world GBA tables + analytics in live DB | ✅ present: `group_bookings`, `group_blocks`, `group_block_daily_allocations`, `group_pickups`, `allotment_contracts`, `allotment_daily_quotas`, `allotment_vouchers`, `allotment_stop_sales`, + `group_block_analytics`, `allotment_analytics` (D-2 verification PASS re-confirmed) |
| `group_blocks.block_type` | ❌ absent → BLK-1 reality re-confirmed |
| Wash/release/attrition tables (`%wash%`, `%attrition%`) | ❌ zero → BLK-2 reality re-confirmed (`penalty_due` exists but is reservation-scoped, not a substitute) |
| `group_blocks.shoulder_days_before/after` | ✅ live, **not** in `schema.prisma` → FIND-1 re-confirmed |
| `allotment_pickups` live, undeclared (no model, no migration) | ✅ re-confirmed → BLK-3 reality; **16 rows, all `ACTIVE`, all with `reservation_id`**; legacy `allotment_pickup` = 0 rows; `group_pickups` = 61 rows; `reservations` = 1124 rows (backfill scope is tiny — informs T-03 RR decision) |
| `group_pickups.group_block_id` NOT NULL, no allotment reference, `room_number` present | ✅ re-confirmed (ALTER requirement for T-03 real) |
| `attrition_threshold` default 80 declared | ✅ `schema.prisma` confirms (`Int?` default 80 — NULL-handling = implementation detail under T-52) |
| `information_schema` column inventory of `group_blocks` (25 cols incl. `version`, `cutoff_date`, `inventory_policy`, `contracted_nights`/`picked_nights`/`released_nights`) | ✅ matches plan §4 WS-01 claims |

### 5.3 New findings from this pass

- **NB-4 (migration bookkeeping):** live `_prisma_migrations` has **51 rows for 49 migration directories**; two names each have a failed-then-retried pair (`20260817000000_add_checkin_sessions_and_guest_documents`, `20260910_b1_trx_codes` — second attempt succeeded for both), and the table has only a primary key (**no unique index on `migration_name`**). Every migration ultimately applied; substantive claims of the plan (zero migrations create GBA tables — re-verified including `20260920_groups_blocks_allotments`, which creates only `reservation_groups`) remain true. Plan wording "51 applied migrations" (§1.1, §1.6 BLK-1 row, WS-01) should read "49 migrations / 51 records". Pre-existing; tracked, not blocking.
- **NB-5 (dormant PUT caller):** `groupBookingApi.releaseAllocation` (the PUT defect at `group-allotment.api.ts:141-146`) is wired only through `useReleaseBlockAllocation` (`use-group-allotment.ts:200-204`), which is **imported but never invoked** in `GroupBookingDetailView.tsx:4`. The only active release UI is `AllotmentDetailView.tsx:677` → `useReleaseAllotmentAllocation` (`:342` → POST path `api.ts:317`). No other caller of either function exists in `apps/web`. FIND-3/D-12 remains a real contract fix (T-44) but is dormant in the current UI — impact lower than the plan implies.

**Conclusion:** no nonexistent path, no renamed/deleted target, no stale reference, no wrong module ownership, no wrong API surface, no wrong schema model, no missing dependency, no architecture mismatch in any task target. The only deltas are the count/cross-reference notes above.

---

## 6. Domain Contract Validation

### 6.1 Internal consistency of `13_FINAL_DOMAIN_SPECIFICATION.md`

- **Rule-level consistency:** re-checked the spec's own contradiction pass (§21.2) claims: counter-timing table vs voucher table vs pickup table; wash `released += unpicked` vs remaining formula vs A1 inputs; S-2 guard vs `Math.max` clamp; D-4 consumer vs S-5 delegation; stop-sale vs guard order; D-6a vs §18 A3 row. **No rule contradictions found** — §21.2's "0" holds.
- **NB-2 (wording inconsistency in the spec's self-check):** spec §21.1 check #2 lists the 6 confirmations as **"(S-2, S-5, D-6a, D-10, S-1, S-3)"** (line 819), while the spec's own header (line 13), `12_DECISION_STATUS.md` §2b (line 49), §3 (line 57, bold items), and §4 (lines 75–77) all record the authoritative set as **"(D-2, D-4, D-6a, D-10, S-1, S-3)"**. S-2 and S-5 are user-confirmed *decisions* (2026-09-30), not confirmation-pass items. Classification: **NON-BLOCKING NOTE** — pure labeling error inside a self-verification table; it changes no rule, and **all eight distinct items (D-2, D-4, D-6a, D-10, S-1, S-3, S-2, S-5) are fully represented** in spec §20.1 and plan §14, so 6/6 confirmation traceability is unaffected. The spec is FINAL and is not edited by this review.
- **Provenance discipline:** `[DECIDED]` / `[DEFERRED]` / `[SPEC-CARRIED]` / `SOURCE-SILENT` tags observed throughout; deferred mechanisms (TR-6.6, TR-11.7) explicitly labeled in §8.2/§12.1/§20.1 (spec §21.1 check 13 ✅ re-verified). No lock syntax, no SQL, no index strategy stated as policy (check 14 ✅). No schema change asserted — only declared-schema gap language (check 15 ✅).

### 6.2 22/22 decisions represented — independently verified

Re-derived from `12_DECISION_STATUS.md` (not from counters): items = D-1…D-15 + D-6a + S-1…S-6 = 22. Every ID appears in spec §20.1 and in plan §14 rows: D-1, D-2, D-3, D-4, D-5, D-6, D-6a, D-7, D-8, D-9, D-10, D-11, D-12, D-13, D-14, D-15, S-1, S-2, S-3, S-4, S-5, S-6. **22/22 ✅.** Decision substance vs plan behavior spot-checked for the highest-risk items:

| Decision | Plan behavior conforms? |
|---|---|
| D-10 = B (real reservation on consume) | ✅ T-22 rewrites `consume-allotment-voucher.handler.ts:26` fabrication; API contract §8 changes response to real-id-or-null; scenario 4 |
| S-5 = A (USED cancel restores iff reservation cancelled) | ✅ T-23 implements both branches + delegation to reservation-cancel semantics; scenario 6/7 |
| D-11 = C amended (hotel-scoped exact order; no fuzzy/LIMIT 1) | ✅ T-25 targets the exact live violations (`prisma-reservation-association.adapter.ts:83/:246` `full_name ILIKE … LIMIT 1` re-verified this session); scenario 24 |
| D-4 = B/C (consumer-derived pickup state; FO raw writes removed) | ✅ T-30/32/33 before T-31 removal — hard activation ordering in §11 flag rules |
| D-2 = A (forward migration; declared = executed) | ✅ T-01/T-02/T-06 + §7 reconciliation matrix; live-DB reality honored |
| S-2 = A (quota ≤ physical; `overbooking_limits` sole facility) | ✅ T-13/T-42/T-66(c); adapter `:52/:116` re-verified — no second mechanism planned |
| D-3 / D-13 / D-14 (wash / shoulder uniformity / attrition) | ✅ T-43…T-55 implement exactly the decided semantics (§14–§16 of this review) |
| D-15 / D-12 / D-5 / D-9 / D-8 / D-7 / S-1 / S-3 / S-4 / S-6 | ✅ T-08, T-44 (+no PUT alias), T-40/62/63/64, T-56…T-60, T-26/41, T-21/28/68, T-11, T-12/14/37, T-59 — each anchored |

### 6.3 6/6 confirmations — independently verified

Authoritative set (evidence: `12_DECISION_STATUS.md` §2b line 49, §4 lines 75–77; spec header line 13): **D-2=A, D-4=B/C, D-6a=A, D-10=B, S-1=A, S-3=A** — each re-checked against its recorded evidence (live-DB PASS; checkout raw writes present; A3 independent at `availability-sales.controller.ts:459`; no active `reservationId` caller; both pickup mechanisms live; both vocabularies present). Plan §14 carries a dedicated confirmation row for each of the six (its confirmation-block rows). **6/6 ✅** (NB-2 records the spec §21.1 mislabeling of two of them).

### 6.4 97/97 target rules — independently verified

- `11_TARGET_BUSINESS_RULES.md` contains exactly **97 unique TR IDs** (re-counted): **95 `[DECIDED]` + 2 `[DEFERRED]`** (TR-6.6 scheduler transport, TR-11.7 lock mechanism — both row-inspected, genuinely deferred; TR-9.7/10.6/15.7 carry `[DECIDED]` tags with mechanism-language inside their text and are decided).
- Plan §14 names **95 of 97 literally** (shorthand `TR-5.1…5.5` ranges expand with ellipsis). **NB-3:** TR-10.3 and TR-4.3 are absent from §14 matrix rows. Semantic coverage is intact: TR-10.3 → plan WS-05 §4 item 4 + T-40 action text + WS-10 consumer matrix; TR-4.3 (concurrent pickup serialization) → §14's TR-11.x row (D-8 invariants) + §9 scenarios 8/9 → T-56…T-58. Classification: **NON-BLOCKING NOTE** (matrix completeness wording; coverage claim holds semantically). 2 deferred TRs map to §13 DEF-1/DEF-2 as required. **97/97 ✅** with NB-3.

### 6.5 No task silently changes or contradicts a rule

All 70 task bodies checked against the contract: silence preserved where the spec is silent (T-34/T-35 stop-and-document discipline; T-27 anomaly surfacing without repair; T-11 raises storage round-trip to RR instead of inventing a column; T-44 forbids the PUT alias; T-52 removes rejected VO variants rather than adopting them; wash reads `picked` untouched per §8.2). **Zero silent rule changes. Zero "implementation detail" items found to be hidden unresolved business decisions** (each of DEF-1…DEF-10 answers the four §23 questions in §23 of this review).

---

## 7. Implementation Plan Validation

### 7.1 Structural audit of all 70 tasks

| Check | Result |
|---|---|
| Task IDs T-01…T-70, no gaps, no duplicates | ✅ re-counted (T-19 present — closes the WS-02 close-out for TR-2.4 `picked ≤ contracted − released` + create-idempotency; the only historically missing slot) |
| Every task has: ID, workstream, purpose, source requirement, dependencies, files, schema/API/event impact, tests, verification command, rollback | ✅ with one field-level nit: **NB-6** — T-55 lacks a `Files:` line (store is TBD by RR, mirroring T-04's explicit "Files: TBD by RR"); all other 69 tasks carry the full field set |
| Missing task / duplicate task / orphan task | ✅ none — no two tasks own the same exclusive outcome; every task is reachable from §6 edges and appears in §15 order |
| Circular dependency | ✅ none — all §6.1 edges point forward (schema → guards → seam → rewrites → consumers → authority → wash → hardening → cutover → T-70); §15 phases are a topological sort |
| Incorrect dependency / wrong order | ✅ none material: consumers before FO raw-SQL removal (T-32/33 → T-31) ✔; seam before rewrites (T-56 → T-20…T-24) ✔; A3 rebuild before F-18 removal (T-38 → T-39) ✔; RR gates before blocked tasks (T-03/04/05 → T-21/45/46/55/68) ✔; guards before intake rewrites (T-08…T-19 → T-20+) ✔. **NB-6:** §6.1 omits three edges stated in task text (T-42→T-39, T-52→T-45, T-62→T-67) — §15 order already respects all three, so no ordering violation results |
| Task depending on an unresolved business decision | ✅ none — 0 open decisions; RR-gated tasks depend only on mechanism confirmations (§23) |
| Task depending on a non-existent component | ✅ none — every Files-path spot-verified (§5.1), including `inventory-commitment.port.ts` (dead-port replacement target), the consumer spec file, the harness, unit-of-work, interceptor, bullmq infra, audit subscriber |
| Task referencing obsolete architecture | ✅ none — plan's current→target map (§3) matches the repo; audit-era components confirmed absent and untargeted |
| Task whose acceptance criteria cannot be verified | ✅ none — every task names a concrete Verify command (jest patterns, `npx prisma validate`, grep gates, typecheck/lint, harness) |
| Event-class create-ownership overlap | **NB-6** (part): T-62 (Phase A) and T-54 (Phase D) both list creation of `AttritionAssessed`; T-62 and T-46 both list `AllotmentWashExecuted` — cross-references, resolvable as create-once + emit; noted so T-62's static "every spec name mapped" test is scheduled after (or tolerant of) later class creation |

### 7.2 Dependency graph & parallel tracks (§6.3 / §15)

Track decomposition is sound: Track A (schema/RR) ∥ B (domain guards) ∥ C (authority) ∥ D (events) ∥ E (isolation) all precede the intake-rewrite convergence; wash needs B + A(T-04/05) + T-52/53; cutover (T-68) needs everything + T-66 clean; **T-70 is the single exit gate**. Widest gate correctly identified as T-56 (15 downstream tasks). No track assigns a task before a prerequisite phase.

### 7.3 Task-level domain spot-checks (highest-risk rewrites)

| Task | Implementable as written? | Evidence |
|---|---|---|
| T-01/T-02/T-06 | ✅ | Live DDL verified matchable with `CREATE TABLE IF NOT EXISTS`; shoulder live columns match plan's `Int @default(0)` intent; drift gate command valid |
| T-03 (⛔ BLK-3) | ✅ once RR confirms (plan-gated) | `group_pickups` reality re-verified; 16 legacy rows make backfill trivially feasible; both options documented |
| T-20/T-21/T-22/T-24 | ✅ | Defects re-verified at cited lines; unit-of-work vehicle exists; scenario coverage attached |
| T-25 | ✅ | Live violations re-verified at `adapter:83/:246`; conflict error code enumerated in T-61 |
| T-30/T-31/T-32/T-33 | ✅ | `check-out.handler.ts` event-emission pattern `:254` present; consumer extension point + spec file exist; activation order enforced by §11 flags |
| T-38/T-39 | ✅ | A3 controller route `:237` + computation region verified; F-18 route/hook/api callers enumerated and verified |
| T-43/T-44 | ✅ | Controllers already `@Post`; PUT defect verified (NB-5: dormant) |
| T-45/T-46/T-47 | ✅ for mechanics; exclusion/log lines gated (intended) | `washAllocation :332`, `canWash/wash` VOs exist; no wash handler/scheduler exists today (net-new, as plan states) |
| T-49/T-50 | ✅ | Raw SQL + undeclared-column writes re-verified in shoulder handlers |
| T-52/T-53/T-54 | ✅ | `attrition-calculation.service.ts` + policy VO exist; folio posting port exists; `ATTRITION_FEE` code check correctly written as verification (not a rule) |
| T-56…T-61 | ✅ | Seam pattern (unit-of-work + `publish(evt, tx)` + version columns) all exist; isolation sweep targets enumerated |
| T-62/T-63/T-64 | ✅ | Consumer spec exists to extend; 22 events enumerated; INV-19 static test testable |
| T-65…T-69 | ✅ | Legacy tables/paths verified; reconciliation service exists; flag wiring pattern (§11) is repo-standard |
| T-70 | ✅ | Commands valid; harness env var = documented Stage 6 setup (§27) |

**§7 conclusion:** plan validated as executable; 6 non-blocking notes and 13 implementation details attached; **no plan-rewrite required.**

---

## 8. Workstream Readiness

| WS | Prereqs exist | Architecture exists | Referenced files exist | DB structures exist or correctly planned | Dependencies executable | Hidden blocker? | Verdict |
|---|---|---|---|---|---|---|---|
| **WS-01** Data Model | none (first) | Prisma + migrate standard | ✅ schema/migrations/harness | Live tables ✅; shoulder declare = T-02; `allotment_pickups` collapse = T-03 ⛔; `block_type` = T-04 ⛔; wash/attrition store = T-05 ⛔ | ✅ (IF NOT EXISTS discipline) | None — the ⛔ items are plan-documented RR confirmations (§23), not surprises | **READY** (3 RR-gated tasks flagged) |
| **WS-02** Core GBA | WS-01 | Aggregates/services/VOs ✅ | ✅ (25 handlers, 3 aggregates, 7 VOs, lifecycle + attrition services) | Reads only after WS-01; type-dependent behavior ⛔ BLK-1 (exclusion ships later — intended) | ✅ | None | **READY** |
| **WS-03** Pickup/Voucher | WS-01(⛔T-03), WS-02, WS-09, WS-05 | Ports/adapters/entities ✅ | ✅ all 4 intake handlers + association adapter | Canonical writes gated by T-03 (flag path documented) | ✅ | None | **READY** (canonical switch RR-gated as designed) |
| **WS-04** Reservation Integration | WS-03, WS-09, WS-10 | Reservation events + consumer + FO handler ✅ | ✅ (`reservation-domain.events.ts`, `events.consumer.ts`, `check-out.handler.ts`) | None | ✅ | None — missing `reservation.checked_out` is FIND-2 → created by T-30 | **READY** |
| **WS-05** Availability | WS-02, WS-10 | A1 adapter + snapshot/assertion + reconciliation ✅ | ✅ all cited files incl. F-18 callers | None | ✅ | None — assertion-port wiring deferred as DEF-3 with interim read-time consult documented | **READY** |
| **WS-06** Wash/Release | WS-01(⛔1/2), WS-02, WS-08, WS-09, WS-10 | Aggregate wash primitive + release handlers + permission ✅; scheduler net-new (port only) | ✅ | ⛔ wash-log store (BLK-2 → T-05) and block-type (BLK-1 → T-04) | ✅ | None — wash-by-date mechanics + counters + events ship without the gated lines; scheduler transport = DEF-1 | **READY** (2 RR-gated lines flagged) |
| **WS-07** Shoulder | WS-01, WS-02 | Handlers + controller routes + live columns ✅ | ✅ | Declare columns = T-02 (already decided under D-2) | ✅ | None — FIND-1 is decided work | **READY** |
| **WS-08** Attrition | WS-06, WS-01(⛔BLK-2), financial port | Calculation service + policy VO + folio port ✅ | ✅ | Threshold column ✅; durable attrition record ⛔ BLK-2 (T-05/T-55) — assessment + posting still ship | ✅ | None — ledger ownership = DEF-4 (dependency note) | **READY** (record persistence RR-gated as designed) |
| **WS-09** Concurrency | seam first (none) | unit-of-work + outbox-in-tx + idempotency service + version columns ✅ | ✅ | Version fields exist (WS-01 verified) | ✅ | None — lock mechanism = DEF-2 (seam ships pluggable; invariant tests don't depend on the pick) | **READY** |
| **WS-10** Events | WS-04/06/08 logic | outbox → BullMQ → consumer + consumer spec ✅ | ✅ | None | ✅ | None — today's `default: warn` gap is the work, not a blocker (T-40/63) | **READY** |
| **WS-11** Legacy/Reconciliation | WS-03/04/01 | legacy models + reconciliation service + audit subscriber ✅ | ✅ | BLK-3 switch only | ✅ | None — no legacy HTTP paths were found to remain as blockers; legacy tables frozen read-only by tests (T-65) | **READY** |

**11/11 workstreams checked.** Each prerequisite named by a workstream exists or is itself a scheduled task; no hidden blocker beyond the three plan-documented ⛔ items routed to §23.

---

## 9. Database / Schema Readiness

### 9.1 Declared / migrated / live reconciliation (plan §7.1 re-verified)

| Item | schema.prisma | migrations history | Live DB | Plan action | Verified? |
|---|---|---|---|---|---|
| 8 new-world GBA tables | ✅ `:17046`–`:17252` | ❌ | ✅ | T-01 baseline `IF NOT EXISTS` | ✅ all three states re-verified |
| Analytics tables (2) | ✅ `:17275/:17293` | ❌ | ✅ | T-01 (include) | ✅ |
| `shoulder_days_before/after` | ❌ FIND-1 | ❌ | ✅ | T-02 declare + fold into T-01 | ✅ |
| `group_blocks.block_type` | ❌ | ❌ | ❌ | ⛔ BLK-1 → T-04 (RR) | ✅ |
| wash/release/attrition stores | ❌ | ❌ | ❌ | ⛔ BLK-2 → T-05 (RR) | ✅ |
| `allotment_pickups` (live ledger, 16 rows) | ❌ undeclared | ❌ | ✅ | T-03 collapse (S-1) + T-68 retire; backfill stance = RR | ✅ (row count captured for the RR data decision) |
| `overbooking_limits` | ✅ `:7434` | ✅ pre-existing | ✅ | none — sole overbooking facility (S-2) | ✅ (agent pass noted the table was created by two historical migrations with differing PK naming — pre-existing quirk outside Phase 4 scope; T-06 drift gate is scoped to GBA models, so it is unaffected) |
| Legacy `allotment` / `allotment_pickup` / `allotment_room_types` / `allotment_ledger` / `reservation_groups` | ✅ models (`:105/:137/:152/:16068/:9100`) | ✅/partial | ✅ | read-only (WS-11), no change | ✅ |
| `reservations` GBA secondary columns | ✅ | ✅ | ✅ (1124 rows) | none — secondary only (TR-15.7), pinned by T-69 | ✅ |
| `penalty_due` | ✅ `:9718` | ✅ | ✅ | none — reservation-scoped, correctly *not* used as BLK-2 substitute | ✅ |

### 9.2 Constraints, indexes, keys, version fields

- Natural keys/uniques the plan relies on exist: `uq_group_blocks_hotel_code`, `uq_group_alloc_hotel_block_date_rt`, `uq_allotment_quota_hotel_allot_date_rt`, `uq_allotment_voucher_hotel_allot_code` (duplicate-voucher prevention at DB level), `(hotel_id, block_code)` — declared per WS-01; live `group_blocks`/`group_pickups` carry `version` columns consumed by the WS-09 seam.
- Every GBA table has `hotel_id` with `(hotel_id, …)` indexes (WS-01 verified) — supports §10 isolation checks; RLS correctly deferred to Phase 2b (out of scope).
- `group_pickups.group_block_id` NOT NULL without allotment ref → the T-03 ALTER (nullable + `allotment_id` + FK + hotel-scoped index) is the correct minimum, consistent with S-1.
- Optional INV-14 CHECK constraints (T-07) correctly marked Phase-2b, not required to exit Phase 4.

### 9.3 Migration state, ordering, and D-2 handling

- **D-2 = A (forward migration) correctly implemented by the plan:** declared-then-executed gap (FIND-4) closed by T-01/T-02 with re-run-safe `IF NOT EXISTS`; fresh-env reproducibility asserted by T-06 (`prisma migrate diff` = empty for GBA models); **no data rewrite** except the explicit RR-gated T-03 backfill (§7.3 backfill stance: required none, conditional reviewed, prohibited counter rewrites). The plan honors "declared schema is the source of truth" without inventing a new migration strategy.
- **Ordering (§7.2):** baseline → conditional ALTERs → optional Phase-2b checks; each listed migration family maps to a task; `pnpm db:generate && npx prisma validate` + harness gates attached. **Migration ordering executable as written.**
- **NB-4 applies here:** plan's "51 applied migrations" phrasing vs reality (49 dirs / 51 records incl. 2 failed-then-retried duplicates; no unique index on `migration_name`). T-06/T-70's `prisma migrate status/deploy` runs should expect the duplicate-name rows (pre-existing; may need `--` waiver or cleanup as an implementation-time chore). Non-blocking.
- **No planned migration violates the domain contract:** all statements are additive/declarative or the S-1-sanctioned nullable/`allotment_id` ALTER; nothing destructive (drop = Phase 11 only); prohibited backfills (counters, `quota > physical`, legacy drift) explicitly excluded by §7.3.

**§9 conclusion: READY** — required structures exist or are correctly planned; blocked items are RR-routed with data evidence gathered (16-row backfill scope).

---

## 10. Property Isolation Readiness

Enterprise multi-hotel rule (INV-1, TR-14.1…14.5, S-6): every statement — including raw SQL — carries `hotel_id`; no bare-id mutation; no cross-tenant guest resolution; no unsafe fallback.

### 10.1 Current-state defects located (evidence) → plan closure

| Defect (re-verified this session) | Closing task(s) | Test anchor |
|---|---|---|
| `cancel-allotment-pickup.handler.ts:66` — `UPDATE reservations … WHERE id = $1` with **no `hotel_id` predicate** | T-24 (scoped `WHERE id=$1 AND hotel_id=$2`) + T-59 sweep + static gate | isolation matrix negatives (T-59), scenario 18 |
| `prisma-reservation-association.adapter.ts:83/:246` — `full_name ILIKE … LIMIT 1` (hotel-scoped but **name-fuzzy**, D-11-prohibited) | T-25 exact-order rewrite + `GUEST_MATCH_CONFLICT` | scenario 24, foreign-hotel guest negative |
| `LIMIT 1` reads in `prisma-folio.port.ts:48`, `prisma-billing-instruction.adapter.ts:24/:70` | T-59 file list explicitly includes both adapters | static gate + isolation matrix |
| Shoulder handlers' raw statements (`add-shoulder-days.handler.ts` `:50/:65/:96/:113`) and other GBA raw SQL enumerated in T-59 | T-59 predicate sweep + repo test gate | static gate enumerating raw statements (whitelist + predicate assertion) |
| `check-out.handler.ts` raw pickup UPDATEs | removed entirely by T-31 (FO performs no GBA writes) | grep gate `UPDATE group_pickups\|UPDATE allotment_pickups` empty under front-office |
| Potential cross-hotel events in consumers | T-32 foreign-hotel event → no-op test; T-62 payload `hotelId` presence test; T-62 static payload-shape test | consumer routing matrix (T-63), scenario 18 |

### 10.2 Isolation coverage across the twelve required surfaces

GBA records ✅ (every model carries `hotel_id`; T-59/T-65 gates) · allotment blocks ✅ (`uq…(hotel_id, …)` keys; handlers hotel-scoped post-sweep) · pickups ✅ (T-20/21/24 hotel-scoped rewrites + canonical read key `(hotel_id, reservation_id)`) · vouchers ✅ (unique key hotel-scoped; consume keyed `(hotel_id, allotment_id, voucher_code)`) · reservations ✅ (all reservation writes in T-20/22/24 scoped; FO removal T-31) · guest resolution ✅ (D-11 hotel-scoped exact order; T-25; no fuzzy/LIMIT 1 name binding; conflict explicit — the *current* code's violations are the task target, not an unplanned gap) · availability ✅ (A1 reads hotel-scoped; invalidation keyed `availability:${hotelId}:*`) · shoulder ✅ (T-49/50 unit includes scope; static gate) · attrition ✅ (wash unit loads block by hotel-scoped id; posting via hotel-scoped folio port after T-59) · master folios ✅ (existing routes behind property-scope guard; T-59 covers folio port SQL) · events ✅ (all new/changed events require `hotelId` — T-62 payload test; consumers no-op on foreign hotel) · reconciliation ✅ (detectors per hotel; report-only).

**Guard against header-only override:** plan §2 principle 1 explicitly prohibits header-only scope override; routes already sit behind `property-scope.guard.ts`/partition middleware (repo architecture), with the **raw-SQL layer as the actual defect surface — precisely what T-59 closes.**

**§10 conclusion: READY** — isolation defects are enumerated with tasks, static gates, and negative tests; no surface lacks a plan item. (NB-1-class line drift does not apply here: cited defect lines were re-verified exactly.)

---

## 11. Reservation Integration Readiness (WS-04)

### 11.1 Required coverage checklist (Stage 5 §10)

| Required behavior | Plan coverage | Verdict |
|---|---|---|
| Reservation creation from pickup consumption | T-22 (real id, in-unit, association port); T-20 block path | ✅ |
| pickup → reservation linkage | T-20/T-21/T-22 (real `reservation_id`); T-27 anomaly surfacing for null-linkage USED vouchers; T-69 association non-authority tests | ✅ |
| Reservation cancellation | T-32 cascade consumer (restore exactly once, unlink TR-9.2, invalidate TR-12.6) | ✅ |
| Voucher cancellation after intake | T-23 (USED → delegate to reservation-cancel semantics; ISSUED → direct restore) | ✅ |
| Quota restoration | single restore authority + T-36 interlock (cancel-command vs cascade double-restore) + preconditions | ✅ |
| Reservation = consumption source of truth | S-5 delegation + TR-12.1 family + INV-4/5 (plan principle 5) | ✅ |
| Check-in | T-34 invariance verification (no counter change; voucher redemption = check-in) | ✅ |
| Check-out | T-30 event + T-33 consumer (`CHECKED_OUT`, counters unchanged TR-4.8) + T-31 FO raw-SQL removal | ✅ |
| No-show | T-34: **no restore** (§7.3) + post-no-show pickup status SOURCE-SILENT → no invented behavior | ✅ |
| Cancellation (GBA-initiated) | T-24 hotel-scoped restore-once; existing reservation-cancel side effect preserved (not re-decided) | ✅ |
| Modification | T-35 SOURCE-SILENT regression guard (counters untouched) | ✅ |
| Extension / overstay | T-35 same (contract silence preserved) | ✅ |
| Room changes | T-35 same | ✅ |
| Batch operations | BatchUpdateStatus et al. are reservation-side; no pickup/counter interaction exists → covered by T-35 family guard scope | ✅ |
| Automatic cancellation | No automatic-cancellation mechanism exists in-repo for GBA-linked reservations; if emitted, it surfaces as `reservation.cancelled` → same T-32 path (event-driven design absorbs it) | ✅ |
| Waitlist promotion where applicable | Waitlist promotion creates/updates reservations only — no GBA counter path (SOURCE-SILENT); would ride the same reservation events if it cancels | ✅ |

### 11.2 Rejected-alternatives check (D-10 = B, S-5 = A)

- **D-10:** the rejected alternative A (caller-supplied `reservationId`) and the fabricated-id status quo are both excluded: T-22 removes `RES-${Date.now()}`; consume response becomes real-id-or-null; **the plan does not invent a frontend contract** — the confirmed fact (no active caller supplies `reservationId`, re-verified: the consume hook is imported-but-uninvoked at the view layer) is exactly why B was chosen (recorded in `12_DECISION_STATUS.md` §2b). ✅
- **S-5:** rejected "always restore" and "never restore" both absent — T-23 implements restore-iff-cancelled with the no-second-restore precondition (TR-11.4), matching spec §6.4/INV-4/5. ✅
- **D-4:** checkout path is consumer-derived; no residual FO ownership of pickup counters remains after T-31 (master-folio auto-post retained as financial-owner behavior — explicitly allowed). ✅

**§11 conclusion: READY.**

---

## 12. Availability Integration Readiness (WS-05)

### 12.1 Contract checks

| Requirement | Evidence | Verdict |
|---|---|---|
| Canonical assertion path / A1 sole authority | plan principle 4; adapter is the one eligible-filters home; no new availability computation anywhere in 70 tasks | ✅ |
| **No parallel authoritative counter** | F-18 route removal (T-39) + A3 rebuild (T-38) + scenario 17 parity + "no second number" tests; GBA never publishes an availability number (principle 4) | ✅ |
| A3 rebuild requirement (D-6a = A) | T-38 sources matrix values from authority; response shape preserved; interim option-B labeling asserted by test | ✅ |
| Pickup guards | T-12/T-13/T-18 + two-layer consult T-26/T-41 (DEF-3 seam) | ✅ |
| Quota constraints | T-12 (`picked + released ≤ quota`), T-19 (`picked ≤ contracted − released`), guard ordering eligibility→stop-sale→remaining | ✅ |
| **Overbooking boundaries — `overbooking_limits` sole facility** | adapter `:52` reads it transiently; clamp `:116` consumes only that input; T-42 asserts sole-facility + no contract-overcommit path + `quota > physical` flagged UNRESOLVED not clamped. **No second overbooking mechanism planned** (S-2 = A honored) | ✅ |
| Reservation assertion lifecycle | TR-4.1/10.6 → T-26 (interim read-time consult against `availability-snapshot.service.ts`) + T-41 (final wiring per DEF-3 RR) — pickup-created reservations route through assertion | ✅ (RR-shaped, §23) |
| Release behavior | T-43 emits `group_block.released`/`allotment.released` in-unit with invalidation effect (TR-5.2) | ✅ |
| Same-key conflict handling | stop-sale flag sync T-14 keeps A1 conflict detection (`availability-source.adapter.ts:84-87`) silent; UNRESOLVED promotion guard `:98-102` kept | ✅ |
| Idempotency | invalidation = cache delete (idempotent); consumer dedup; wash/pickup keys | ✅ |
| Transaction boundaries | §12.2 units (see §17); read-side work is non-transactional by design | ✅ |
| Property safety | §10 | ✅ |
| Fail-closed where required | TR-10.4 flagged-UNRESOLVED preserved (T-13/T-42/T-66c); clamp never becomes enforcer (spec §21.2) | ✅ |
| Invalidation (TR-10.3) | T-40 registers GBA event → `cache.delPattern('availability:${hotelId}:*')` + snapshot/assertion hooks; T-63 enforces INV-19 (no `default: warn` for Phase-4 types) | ✅ (NB-3: TR-10.3 absent from §14 rows only) |
| S-2 = A (quota ≤ physical) | T-13 rejects at create/set-quota using A1's physical definition; historical violations surfaced only (T-66c) — never rewritten | ✅ |

### 12.2 A1 adapter conformance (re-verified)

Filters already decided-conformant: `deleted_at: null` + status whitelist `:49-52`; DEDUCT/eligibility `:66`; integrity flags `:68-88`; validity-window/expired-zero `:50`; overbooking input `:52` + clamp `:116`. The single non-conformant fact — `allotmentStopSaleActive: false` hard-coded `:113` — is T-37's explicit target (display-only wiring; quantity untouched, S-4 scope). Existing postgres suites (`availability-isolation`, `fail-closed`, `lock-order`, `assertion*`) must stay green (WS-05 verification + §21 regression).

**§12 conclusion: READY.**

---

## 13. Pickup / Voucher Readiness (WS-03)

| Requirement | Plan coverage | Verdict |
|---|---|---|
| Canonical pickup record (S-1) | ONE `group_pickups` row for both paths (T-21 write, T-28 read, T-68 flag switch, T-69 non-authority); legacy `allotment_pickups` retireable after T-03 | ✅ ⛔-gated as designed |
| Voucher lifecycle + ISSUED/USED/CANCELLED semantics | T-15 enforces §6.3 matrix; `EXPIRED` mechanism = DEF-9 (read-derived vs sweep — semantics unchanged) | ✅ |
| Reservation linkage | T-22 real id / null; anomaly surfaced not repaired (T-27, SOURCE-SILENT per D-10 note 4) | ✅ |
| Quota consumption | issue = `picked++` hold (spec §6.3); consume = no counter change; guards T-12/T-19 | ✅ |
| Quota restoration | ISSUED cancel → restore (T-23/TR-12.5); reservation-cancel cascade → restore once (T-32); checkout/no-show never restore (T-33/T-34) | ✅ |
| **USED voucher cancellation (the resolved issue)** | Current code defect confirmed in scope: `cancel-allotment-voucher.handler.ts` state machine permits cancel-from-ISSUED only. **The plan explicitly covers the correction**: WS-03 target line "S-5: cancelling USED voucher → quota restores iff linked reservation cancelled; delegates to reservation-cancel semantics" + **T-23** with the exact both-branch tests (USED w/ live reservation → no restore; USED w/ cancelled reservation → restore once; ISSUED → restore; duplicate-cancel no-op) + §9 scenarios 6/7. | ✅ **explicitly covered** |
| Idempotency | T-29 business keys (voucher unique constraint exists; status preconditions) + optional Idempotency-Key interceptor (DEF-7) + consumer dedup | ✅ |
| Duplicate consumption protection | second consume of USED → deterministic rejection/idempotent return (T-22, INV-3); DB unique key `uq_allotment_voucher_hotel_allot_code` | ✅ |
| Invalid state transitions | T-15 full matrix rejection; T-61 deterministic error codes (`VOUCHER_ALREADY_USED`, `GUEST_MATCH_CONFLICT`, …) | ✅ |
| Direct pickup mechanism | retained (S-1 confirmed direct intake stays live — `AllotmentDetailView.tsx` caller verified) but converges on canonical record | ✅ |
| Canonical pickup API/service | `GET /allotments/:id/pickups` → canonical behind flag, response shape preserved (T-28 field-parity test) | ✅ |
| Legacy mechanism containment | legacy table writes allowed only behind the cutover flag until T-68 (T-65 gate: GBA never writes `allotment`/`allotment_pickup`/…) | ✅ |

**§13 conclusion: READY.**

---

## 14. Wash / Release Readiness (WS-06) — full D-3 model

| D-3 element | Plan coverage | Verdict |
|---|---|---|
| **T-1** block cut-off = single wash event | T-45 `ExecuteCutOffWash`: `cutoff_date <= today` blocks, one unit per block, idempotency key `(block_id, cutoff_date)`, re-run no-op, scenario 11 | ✅ (date mechanics + counters + events ship unblocked) |
| **T-2** rolling N-days for `ROLLING_RELEASE` | T-46 `ExecuteRollingRelease`: window `today..today + release_days_before` (existing default 14), key `(allotment_id, windowStart, windowEnd)`, scenario 12 | ✅ |
| **GUARANTEED_BLOCK never washes** | Contract-level exclusion in T-46 (uses S-3 mapping — **not** BLK-1); block-level exclusion line in T-45 gated on T-04 ⛔ (block type storage); scenario 13 with explicit gated marker | ✅ as designed (gated line tracked in §23) |
| Date-based granularity | T-45 applies from start of cutoff day (TR-6.4) | ✅ |
| Manual release coexists as override | T-43 keeps existing release commands (guard `release ≤ held − picked`, state guards, in-unit events); T-44 verb fix | ✅ |
| Wash events follow D-5 consumer model | `group_block.washed` exists (enrich before/after), `AllotmentWashExecuted` created (T-46/T-62), consumed via T-40 invalidation + T-63 INV-19; event-less wash prohibited (spec §8.2) | ✅ |
| Atomicity per D-9 | §12.2 wash row = counters + wash log + attrition + fee posting one commit; T-58 adoption; failure injection (attrition failure → no counter change) in T-45 tests | ✅ |
| Transaction boundaries / idempotency / repeated execution | seam T-56; deterministic keys; scenarios 10/11 (concurrent wash → second no-op; repeated → no-op logged once) | ✅ |
| Scheduler boundary | **DEF-1 — intentionally deferred**: T-47 defines `WashSchedulerPort` only; transport alternatives (BullMQ repeatable jobs / `@nestjs/schedule` / Temporal) documented with in-repo evidence; `gba.wash.schedulerEnabled` flag off until RR pick; TR-6.5 data-honesty labeling via T-48 while inactive | ✅ **deferred stays deferred** |
| Manual override / rolling release / guaranteed block / historical already-washed blocks | override ✅ (T-43); rolling ✅ (T-46); guaranteed ✅; historical: wash idempotency keys make already-washed ranges no-ops; `cutoff_date`-driven metadata labeled non-authoritative until scheduler on (T-48) | ✅ |
| Shoulder dates in wash | T-45 "no exclusion filter" (TR-8.6b) + scenario 14 | ✅ |

**§14 conclusion: READY** — wash date/counter/event mechanics implementable now; only the two storage lines (T-04 type column, T-05 wash log) are RR-routed (§23), and the scheduler transport remains the plan-documented deferral.

---

## 15. Shoulder-Day Readiness (WS-07) — D-13 = A

| D-13 element | Plan coverage | Verdict |
|---|---|---|
| Shoulder rooms count in attrition base (P1) | T-52 base = wash range incl. shoulder; T-51 cross-suite proves it; scenario 14/15 | ✅ |
| Shoulder days follow same cut-off/wash behavior (P2) | T-45 covers extended range (TR-8.6b); T-51 | ✅ |
| Unrestricted pickup (P3) | T-49 creates allocations through aggregate with **core-day guards** (identical guards, TR-8.1); T-51 parity test | ✅ |
| Identical guards / no differentiated buffer | T-49 guard parity tests; rejected buffer variants absent (D-13 rejected them) | ✅ |
| Integration: availability | shoulder allocations are ordinary allocation rows → A1 reads them via existing filters; mutation events → T-40 invalidation (mapped `group_block.allocation_changed` / `ShoulderAllocationChanged`) | ✅ |
| Integration: pickup eligibility | T-49 same guards as core days | ✅ |
| Integration: wash/release | T-45 (extended range) + T-43 (release on shoulder allocations follows same bounds) | ✅ |
| Integration: attrition | T-52/T-51 (base includes shoulder) | ✅ |
| Integration: reconciliation | T-66 detectors operate per hotel/date/room-type — shoulder dates are ordinary dates in those grids | ✅ |
| Declared quantities only (TR-8.2) | T-49 rejects silent qty-0 inserts (request-shape change, §8 API row); removal guard + rollup (T-50, TR-8.3) kills unguarded deletes (F-11) | ✅ |
| Column declaration (TR-8.5) | T-01/T-02 (FIND-1 — live-verified undeclared columns) | ✅ |

**§15 conclusion: READY** — the only shoulder gap is the already-decared schema declaration (T-02), not an open question.

---

## 16. Attrition Readiness (WS-08) — D-14 = A

| D-14 element | Plan coverage | Verdict |
|---|---|---|
| Attrition activated | wash invokes assessment (T-53 call site in T-45/T-46) | ✅ |
| Default threshold = 80% | `group_blocks.attrition_threshold` default 80 declared (re-verified `:17095`); T-52 pins default-80 test + bounds 50–100 `[SPEC-CARRIED]` | ✅ |
| Uniform per block | single percentage per block (T-52 policy conformance; rejected VO variants — 85 default / per-day / cumulative / room-type — removed or normalized) | ✅ |
| Assessed at wash / exactly once / same wash transaction | TR-6.7 → T-53 inside wash unit; wash idempotency ⇒ single assessment (INV-11; scenario 15); never async (scenario 16 failure injection: posting fails → no wash) | ✅ |
| Shoulder days included in base | T-52/T-51 | ✅ |
| Shortfall × rate math | `minimumRequired = ceil(contracted × threshold / 100)`; `shortfall = max(0, minimumRequired − picked)`; `liabilityDue = shortfall × negotiatedRate` — T-52 tests incl. exact-threshold boundary | ✅ |
| Master folio + `ATTRITION_FEE` | posting via existing `prisma-folio.port.ts` to the block's master folio in the wash unit; `ATTRITION_FEE` trx-code existence = verification step (create only if code table legitimately lacks it — flagged in PR, not invented as policy) | ✅ |
| Inventory read-only / no pickup-guard mutation / pre-wash report read-side | T-52 "no inventory mutation" proof test; assessment never evaluated at pickup/booking/availability/expiry (WS-08 target 1); pre-wash reports carry no authority | ✅ |
| Ledger ownership deferred (spec open Q2) | **DEF-4** — posting uses existing folio structures; owning module = RR dependency note. Not an unresolved business decision (mechanism/architecture boundary) | ✅ (§23) |
| Durable attrition record | ⛔ BLK-2 → T-05/T-55 RR store; interim audit trail via T-54/T-67 event subscription | ✅ as designed |

**§16 conclusion: READY** — calculation, posting, and idempotency ship independently; only the durable record store is RR-routed.

---

## 17. Concurrency / Transaction Readiness (WS-09)

### 17.1 Spec §12 contract → plan mapping (re-verified line-by-line)

Spec §12.1 standing invariants (D-8): (1) no lost updates → T-56 seam (exactly-one-wins, version mismatch → `CONFLICT`); (2) conflict is an error → T-61 maps to `409 CONFLICT` + deterministic codes; (3) core inequality after commit → T-19/T-12 post-commit guards + T-56 `expectedVersions` + T-60 fault injection; (4) exactly-once per business event → T-29/T-32/T-64 preconditions + event-id dedup; (5) version monotonicity → seam tests. **All 5 closed by plan.**

Spec §12.2 has **13 operation rows** (recounted: pickup-block, pickup-allotment, voucher issue, voucher consume, reservation-cancel cascade, voucher cancellation, ISSUED cancel, wash, manual release, attrition posting, block status transitions, quota changes, shoulder add/remove). Plan T-57 covers **rows 1–7**, T-58 covers **rows 8–13** — **exact match; no operation row lacks a home.**

Spec §12.3 cross-cutting: atomicity → T-60 (delete `rollbackQuota` swallow, fault injection each step asserts zero partial rows); idempotency → INV-3/6/10/11 tasks; ordering none across events → consumers order-tolerant (T-64 out-of-order pairs); lock ordering **[DEFERRED]** → documented acquisition order in seam (reservation → pickup → allocation/quota, matching existing `availability-lock-order.postgres.spec.ts` pattern) + DEF-2; hotel isolation → §10 (T-59); duplicate ops → deterministic rejection/no-op; partial failure → unit rollback + surfaced error (D-9).

### 17.2 Mandatory validation dimensions

| Dimension | Plan coverage | Verdict |
|---|---|---|
| Transaction scope | §12.2 rows → one unit-of-work each (T-56 seam over existing `common/database/unit-of-work`) | ✅ |
| Isolation level | harness baseline `ReadCommitted` (re-verified pattern) — reuse unless RR opts otherwise (DEF-2) | ✅ deferred-as-mechanism |
| Row locking / optimistic concurrency | version columns exist on allocations/quotas/contracts; seam exposes `expectedVersions`; strategy pluggable (optimistic retry / `FOR UPDATE` / advisory — alternatives §13) | ✅ deferred-as-mechanism |
| Version predicates | conditional-update pattern in seam; stale → `CONFLICT` (T-56 tests) | ✅ closes "missing version predicate" finding |
| Deterministic lock ordering | documented order + lock-order spec pattern; validated by concurrency tests | ✅ |
| Deadlock risk | fixed acquisition order + bounded retry in seam alternatives; concurrent-suite (concurrent consume/cancel/wash) would surface it | ✅ |
| Idempotency | business keys + status preconditions + event dedup + optional header interceptor (DEF-7) | ✅ |
| Retry behavior | bounded retry on version conflict (mechanism choice) ; consumer retries = existing queue rethrow `:61-63` with outbox `maxRetries: 5` | ✅ |
| Partial-commit prevention | T-60 + fault-injection suite; compensating-swallow prohibited (principle 6) | ✅ |
| Event/outbox ordering | producers publish in-tx (`event-bus.publish(evt, tx)`); delivery at-least-once; consumers order-tolerant by contract §7.2 | ✅ |
| reservation + pickup atomicity | T-20/T-22 single unit | ✅ |
| reservation + quota atomicity | T-21 single unit (quota hold + reservation + record + events) | ✅ |
| wash + attrition atomicity | T-45/T-53 single wash commit (scenario 16) | ✅ |
| cancellation + restoration atomicity | T-32 consumer unit + T-24 command unit (each internally atomic; cross-path exactly-once via preconditions — T-36) | ✅ |

### 17.3 Previously identified audit findings — closed by the plan?

| Audit finding (Stage-5 §16 list) | Closed? |
|---|---|
| Pickup committing reservation before counters (partial-commit ordering) | ✅ T-20/T-21 reorder into one transaction + fault injection |
| Missing version predicate | ✅ T-56 seam `expectedVersions` + stale→`CONFLICT` tests |
| Divergent availability numbers | ✅ single authority (T-38/39) + invalidation (T-40) + no-second-number tests (T-42) + reconciliation (T-66) |
| Concurrent mutation risks | ✅ concurrency suite (simultaneous pickup/cancel/wash, repeated command) + seam + scenario 8/9/10 |
| Partial state transitions | ✅ T-15 voucher matrix + T-08 lifecycle rejection + status-precondition no-ops throughout |

**§17 conclusion: READY** — every critical mutation path has a defined unit, guard, and test; the only open choice (lock mechanism/isolation) is DEF-2, correctly deferred with a swappable seam (§23).

---

## 18. Event / Outbox Readiness (WS-10)

| Requirement | Plan coverage | Verdict |
|---|---|---|
| Event ownership | GBA events published by GBA aggregates/handlers in-tx; reservation events by reservation domain; FO emits checkout event only (no GBA writes post-T-31) | ✅ |
| Producers | 22 existing IntegrationEvents kept; **4 new**: `AllotmentWashExecuted`, `AttritionAssessed`, `ShoulderAllocationChanged` (or documented alias), `reservation.checked_out` (T-62/T-30/T-46/T-54) | ✅ |
| Consumers | T-40 (GBA → invalidation), T-32 (reservation.cancelled → cascade), T-33 (checked_out → pickup status), T-63 routing matrix; INV-19 static test = every published event ≥ 1 consumer; `default: warn` eliminated for Phase-4 types | ✅ |
| Transaction/outbox boundaries | outbox writer accepts `tx`; all producers publish in-unit; outbox → BullMQ `events` queue → consumer (existing pipeline re-verified) | ✅ |
| Duplicate event handling | `EventIdempotencyService` (job.id/event-id dedup `:40-43`) + handler status-precondition no-ops; T-64 duplicate-delivery suite | ✅ |
| Idempotent consumers | required for all new handlers; T-64 tests double delivery + replay after outbox redelivery | ✅ |
| Tenant context | every event payload carries `hotelId` (T-62 payload-shape test); consumers no-op on foreign-hotel (T-32 negative test) | ✅ |
| Ordering requirements | none guaranteed per contract §7.2 — consumers order-tolerant; T-64 out-of-order pairs (cancel+checkout, wash+pickup) | ✅ |
| Wash events | `group_block.washed` (exists, enriched) + `AllotmentWashExecuted` (new) + no event-less wash (spec §8.2; T-45 tests) | ✅ |
| Reservation events | `reservation.cancelled` (exists, in-tx at `reservation.repository.ts:791`) + `reservation.checked_out` (T-30) | ✅ |
| Pickup events | `group_pickup.created/cancelled`, `allotment.voucher_*` published in-tx with real `reservationId` (WS-03 event impact) | ✅ |
| Cancellation/restoration events | cascade publishes/restores within consumer unit; invalidation event effect (TR-12.6) | ✅ |
| Reconciliation events/cadence | T-66 wired to existing reconciliation cadence (no new transport) + audit subscription T-67 | ✅ |
| **No competing event sources** | single outbox bus; `audit-subscriber` and `events.consumer` are consumers, not producers; FO narrowed to reservation events; A3 writes remain A3's own restrictions (not availability numbers) | ✅ |
| Failure handling | consumer throw → queue retry (existing rethrow); failures never swallowed (D-4 rule; plan observability §10.2) | ✅ |
| Replay | outbox PENDING re-drivable; idempotent consumers make replay safe; runbook only, no new transport | ✅ |

**§18 conclusion: READY** — the pipeline exists; the plan's work is adding cases and contracts, not designing transport.

---

## 19. Legacy Containment Readiness (WS-11)

### 19.1 Legacy paths that remain live → disposition per plan

| Legacy path | Plan disposition | Verdict |
|---|---|---|
| `allotment` / `allotment_pickup` / `allotment_room_types` / `reservation_groups` / `reservation_block` / `allotment_ledger` tables | **read-only legacy** — GBA performs zero new writes (T-65 static gate enumerating raw-SQL targets; regression tests) | ✅ retained (contract §18 forbids Phase-4 deletion) |
| Legacy `availability` counters (A4, `crs-engine.service.ts`, `schema.prisma:433`) | never read/written by GBA (TR-15.1); A4 non-read test (T-65) + `reservations` secondary-column non-authority tests (T-69) | ✅ |
| Live undeclared `allotment_pickups` (current write target) | write path switches to canonical under `gba.pickup.canonicalWrite` flag (T-68); table becomes read-only after cutover; retire in Phase 11 | ✅ cutover-staged (⛔ T-03) |
| FO raw `UPDATE pickup` statements | **removed** (T-31) after consumers active | ✅ redirect→event |
| Legacy UI `VouchersModal.tsx` vs new `VoucherPickupModal.tsx` | both retained until Phase 11; new feature views live under `features/group-allotment/` | ✅ explicitly retained |
| A3 self-computed matrix | **redirected** to authority (T-38, shape preserved) | ✅ |
| F-18 `GET available-rooms` + frontend callers | **removed** (T-39) | ✅ |
| Legacy endpoints in `rates-inventory` | none found targeting allotment GBA reads/writes beyond A4 counters (agent pass: no legacy `allotment` singular HTTP endpoints outside GBA module) — retained implicitly until Phase 11 | ✅ |
| Legacy cache keys in `events.consumer` | existing cache invalidation extended (T-40) — legacy keys untouched (out of Phase-4 scope) | ✅ |

### 19.2 "No two authoritative numbers after implementation"

- One pickup ledger: S-1 → T-21/28/68/69 (canonical `group_pickups` only; legacy table frozen).
- One availability number: A1 sole authority → T-38/39/42 (F-18 gone; A3 derives; GBA publishes nothing).
- One consumption truth: the reservation → D-10/S-5 tasks.
- One guard formula: `quota − picked − released` (+ block `picked ≤ contracted − released`) — no variant guards planned.
- Reconciliation (T-66) detects drift between legacy and canonical **as reports only** — never a second authority.

**Divergence watch (Stage-5 §18):** canonical Availability vs legacy counters vs Activities matrix vs other "available" computations — all four are addressed: canonical = A1 (unchanged authority), legacy A4 frozen, Activities rebuilt (T-38), F-18 removed (T-39), parity/absence tests attached (scenarios 17/19).

**§19 conclusion: READY.**

---

## 20. API / Frontend Readiness (§8 contract + web layer)

### 20.1 Planned API changes — all traced, none invented

Every row of plan §8 re-validated against the controllers: route paths/verbs exist as cited (`POST …/release` `group-booking.controller.ts:247` & `allotment.controller.ts:239`; `GET available-rooms` `:95`; voucher issue/consume/cancel `:184/:206/:217`; pickup create/cancel/list `:236/:251/:271/:281`; quota `PUT :174`; stop-sale `:195/:228`; shoulder `:267/:277`; deletes `:297/:304/:321`; master folio `:143-177`; A3 matrix `availability-sales.controller.ts:237`). Changes are: 1 route retirement (T-39), 1 frontend verb fix (T-44), request-shape change for shoulder quantities (T-49), response semantics (real `reservationId`), error-code/`409` mapping (T-61). **0 invented endpoints** — wash deliberately has no HTTP surface (T-47 invokes handlers via port).

### 20.2 DTO/validation/handlers/response/permissions

- Validation: state preconditions + guard errors deterministic (T-08/T-12/T-15/T-18/T-49 request quantities); controller-level request validation for shoulder added (WS-07 files).
- Response contracts preserved where promised (A3 matrix shape, pickup list shape — parity tests T-28/T-38).
- Permissions unchanged; `GROUP_BLOCK_WASH :14` exists and is reused for any manual invocation surface.
- Error states: deterministic codes enumerated (T-61) mapped through existing `exception.filter.ts`; `409 CONFLICT` for TR-11.2.
- Idempotency keys: business-key first (T-29); header interceptor mechanics = DEF-7 (interceptor exists — re-verified).

### 20.3 Frontend consumers / hooks / mutations

- All cited hooks exist at cited lines (`useAddShoulderDays :101`, `useRemoveShoulderDays :112`, `useSetDailyAllocation :200` region, `useAvailableRooms :213`, `useConsumeAllotmentVoucher :305`, `useAllotmentPickups :381`, `useCreateAllotmentPickup :401`, `useCancelAllotmentPickup :422` — line numbers verified exact).
- **Confirmed fact handled correctly:** the reservation-aware consume hook has **no active caller supplying `reservationId`** (re-verified: consume hook imported but not invoked at view level; no caller passes the field). The plan handles this per D-10=B: backend creates/links the real reservation; response `reservationId` real-or-null; **no frontend contract is invented that conflicts with D-10** (WS-03 API impact). ✅
- **NB-5** records that the block-release PUT path is likewise dormant (imported-but-unused `useReleaseBlockAllocation`); T-44 still fixes it (decision D-12), impact lower than stated.
- F-18 callers enumerated (`group-allotment.api.ts:29` + `useAvailableRooms` + views) — T-39 updates each; empty component dirs exist for new shared components.
- Web gates: `pnpm typecheck && pnpm lint && pnpm test` (jest config verified) attached to T-28/T-39/T-44/T-48/T-70.

**§20 conclusion: READY.**

---

## 21. Test Readiness (§9)

### 21.1 Infrastructure (verified)

- jest API (`testRegex .spec.ts$`) and web jest config exist; **Postgres harness exists** (`availability-postgres.harness.ts` — `describePostgres` env-gated, `splitSqlStatements`, ReadCommitted); compose `xylo-postgres` reachable (queried live this session). `AVAILABILITY_TEST_DATABASE_URL` unset today → **Stage 6 setup item (§27), not a plan gap.**
- **GBA has zero tests today** — plan correctly marks the entire §9 strategy as create-scope (not extend-scope) except `events.consumer.spec.ts` (exists — verified) and availability suites (exist — must stay green).

### 21.2 Required coverage → plan mapping

| Required category | Plan coverage | Verdict |
|---|---|---|
| Unit: state machines | T-08 full transition matrix; T-15 voucher matrix; scenario 25 | ✅ |
| Unit: guards | T-12/T-13/T-18/T-19/T-43 bounds + ordering tests (TR-2.2/2.3/2.4/3.2/3.3/3.5/5.1/7.4) | ✅ |
| Unit: calculations | T-52 attrition math (ceil boundary, shortfall 0 at threshold); T-46 window math | ✅ |
| Unit: guest resolution | T-25 order/conflict/isolation/no-fuzzy tests; scenario 24 | ✅ |
| Unit: attrition / wash logic | T-52/T-45/T-46 suites incl. idempotent re-run, shoulder-included, picked-preserved | ✅ |
| Integration: reservation → pickup | T-20/T-22 postgres specs (real FK row asserted); scenarios 2/4 | ✅ |
| Integration: pickup → quota | T-21 + guard tests; scenarios 1/2/3 | ✅ |
| Integration: reservation → availability | T-32 invalidation + A1 adapter suites (existing stay green) | ✅ |
| Integration: voucher cancellation | T-23 both branches; scenarios 6/7 | ✅ |
| Integration: wash/release | T-45/T-46/T-43 postgres wash suite; scenarios 10–13 | ✅ |
| Integration: shoulder days | T-49/50/51; scenario 14 | ✅ |
| Concurrency: simultaneous pickup / cancellation / pickup+cancellation / modification+cancellation / wash+pickup / repeated command | T-56–T-58 concurrency suite + T-36 interlock + T-64 duplicates; scenarios 8/9/10 | ✅ |
| Property isolation: same IDs across hotels, cross-hotel guest conflicts, cross-hotel pickup attempts | T-59 matrix + T-25 foreign-hotel negative + per-suite negatives (scenario 18) | ✅ |
| Regression: legacy consumers, existing reservation flows, existing Availability tests | existing `events.consumer.spec.ts` extended (not replaced); A1/`availability-*` postgres suites must remain green (WS-05/WS-09 verification); FO `check-out.handler.spec.ts` updated (WS-04 files); T-70 full battery | ✅ |
| Migration-from-scratch / drift | T-01 harness test + T-06 `prisma migrate diff` gate | ✅ |
| API contract / route-absence / verb | T-61/T-39/T-44 tests | ✅ |
| E2E workflow chains | §9.1 end-to-end level (issue→consume→checkout→cancel; wash day; shoulder add→pickup→wash→attrition) | ✅ |

### 21.3 Critical business rules without executable coverage?

Cross-referenced the 95 decided TRs against §9.2's 25 mandatory scenarios + per-task test lists: every decision-critical rule family (lifecycle, guards, wash, shoulder, attrition, pickup/voucher, cascade, isolation, legacy, events, authority) has at least one mapped scenario/task test. Rules whose verification is a *static gate* (grep) are paired with runtime negatives. **No critical rule left without planned executable coverage.**

**§21 conclusion: READY** — one environment setup item recorded in §27.

---

## 22. Migration / Cutover / Rollback Readiness

Validated against what the plan already specifies (§7, §11) — no new rollout strategy invented.

| Dimension | Plan specifies | Verdict |
|---|---|---|
| Migration order | baseline (T-01/T-02) → RR-gated ALTERs (T-03/04/05) → optional Phase-2b checks (T-07); drift gate (T-06) **before any code deploy** | ✅ |
| Deployment order (activation sequence) | schema → domain guards → seam → consumers → intake rewrites → authority reads → A3 → F-18 removal → wash (flag) → attrition (with wash) → canonical cutover → FO raw-SQL removal → T-70 | ✅ matches §6 graph; consumers-before-FO-removal is a *hard* rule (§11.2) |
| Backward compatibility | `IF NOT EXISTS` additive migrations apply cleanly over live DB; response shapes preserved (A3, pickup list); no API removal before frontend switch (F-18 retirement last among availability tasks) | ✅ |
| Data compatibility | no data rewrite by default (§7.3); conditional backfills each RR-reviewed with parity report (T-66e) before switch; prohibited rewrites explicit (counters, `quota > physical`, legacy drift) | ✅ |
| Existing live GBA data | live tables + 16 legacy pickup rows + 61 `group_pickups` + 1124 reservations all compatible with additive path; baseline migration must match live DDL exactly (T-01 action states this) | ✅ |
| Forward migration strategy | D-2=A honored verbatim (§9.3) | ✅ |
| Feature activation sequence | 4 named flags in dependency order: `gba.pickup.*` : consumers → wash → a3 — each with off-state semantics documented | ✅ |
| Reconciliation | T-66 detectors soak after consumers-on and canonical-on; **exit gate: zero detector hits attributable to the new path** (legacy drift expected, Phase 11 owns) | ✅ |
| Rollback boundaries | every flag off → previous behavior; migrations non-destructive (down-scripts only for RR additions); route removal rollback = redeploy; FO raw-SQL restore only under emergency flag (documented, not default); **monitored window R-04**: consumers off → cancellations queue in outbox, drain on re-enable | ✅ |
| Partial deployment behavior | single-writer flag model (**no dual-write ever**); dual-*read* only during pickup parity window; mixed old/new consumers tolerated via order-tolerance + preconditions | ✅ |
| Failure recovery | outbox retries + consumer idempotency; wash dormant until flag; posting failure aborts wash unit (fail-closed — no partial wash) | ✅ |
| **NB-4** | `_prisma_migrations` has 2 duplicate failed-then-retried records and no unique index on `migration_name` (49 dirs / 51 rows) — pre-existing; `migrate status/deploy` runs in T-06/T-70 should account for it; plan's "51 applied" wording reads "49 migrations / 51 records" | ✅ tracked |

**§22 conclusion: READY** — rollout/rollback/recovery are specified, ordered, reversible, and honest about the consumers-off window.

---

## 23. Deferred Items

Every item intentionally deferred, evaluated against the four required questions: (1) truly implementation detail? (2) can implementation proceed without another business decision? (3) sufficiently constrained by the Final Domain Specification? (4) is the Implementation Plan responsible for resolving it?

| # | Item | (1) Detail? | (2) No business decision needed? | (3) Spec-constrained? | (4) Plan resolves? | Classification |
|---|---|---|---|---|---|---|
| DEF-1 | Wash scheduler transport (TR-6.6) — BullMQ repeatable vs `@nestjs/schedule` vs Temporal | ✅ transport choice | ✅ wash semantics fully decided (D-3: when/what/idempotency/atomicity/events) | ✅ spec §8.2 fixes behavior; only invocation vehicle open | ✅ T-47 port + RR pick at Stage-6 entry; flag keeps wash dormant until then | IMPLEMENTATION DETAIL |
| DEF-2 | Concurrency lock mechanism (TR-11.7) — optimistic retry vs row locks vs advisory; isolation level | ✅ | ✅ invariants fully decided (D-8/§12.1: no lost updates, `CONFLICT`, inequalities, exactly-once, version monotonicity) | ✅ spec §12 states invariants only, forbids lock syntax | ✅ T-56 seam ships pluggable; RR picks; invariant tests pass either way | IMPLEMENTATION DETAIL |
| DEF-3 | Pickup→assertion port shape (D-7 mechanism) — assert-at-write vs read-time consult | ✅ | ✅ two-layer validation requirement decided | ✅ spec fixes both layers + single authority | ✅ T-26 (interim read-time consult documented) + T-41 (RR confirms final wiring) | IMPLEMENTATION DETAIL |
| DEF-4 | `ATTRITION_FEE` ledger ownership (spec open Q2) | ✅ architecture boundary | ✅ assessment/posting/atomicity decided (D-14/TR-6.9) | ✅ posting mechanics fixed: master folio, in-wash-unit | ✅ posting via existing `prisma-folio.port`; ownership = RR dependency note | IMPLEMENTATION DETAIL |
| DEF-5 | Guest-identity caller contract (D-11 note) | ✅ caller mechanics | ✅ lookup order/conflict rule decided | ✅ TR-4.10/9.6 | ✅ T-25 (conflict UX reviewed, not decided) | IMPLEMENTATION DETAIL |
| DEF-6 | Reservation confirmation semantics for pickup-created reservations (TR-9.7) | ✅ | ✅ "not hardcoded; honors reservation state machine" decided | ✅ | ✅ existing `reservation-status.service.ts` (verified exists) | IMPLEMENTATION DETAIL |
| DEF-7 | Request-level idempotency header mechanics (TR-11.4 note) | ✅ optional scope | ✅ exactly-once counter semantics decided | ✅ interceptor exists | ✅ T-29 + RR optional wiring | IMPLEMENTATION DETAIL |
| DEF-8 | Forward-migration CI wiring detail (D-2 note) | ✅ | ✅ declare-everything decided | ✅ repo standard = Prisma migrate | ✅ T-06 | IMPLEMENTATION DETAIL |
| DEF-9 | `EXPIRED` voucher mechanism (read-derived vs sweep) | ✅ | ✅ states + rejection semantics decided (TR-13.3; expiry never washes TR-13.4) | ✅ §6.3/§13.2 | ✅ T-15 choice at implementation | IMPLEMENTATION DETAIL |
| DEF-10 | Metrics vehicle (none in repo) | ✅ tooling | ✅ metric requirements listed (§10.2) | ✅ (requirements-only by design) | ✅ RR only if a stack appears | IMPLEMENTATION DETAIL |
| **BLK-1** | `group_blocks.block_type` storage for the GUARANTEED_BLOCK exclusion | ✅ storage location | ✅ the *rule* is decided (D-3/TR-6.3: guaranteed never washes; triad vocabulary = S-3) | ✅ **spec §17.1 already declares the column** — the confirmation is "implement as declared vs alternative" | ✅ T-04 (RR session) before T-45's exclusion line; date mechanics ship regardless | IMPLEMENTATION DETAIL |
| **BLK-2** | wash/release/attrition persistence store | ✅ store shape | ✅ audit requirements decided (TR-6.1/6.5/6.9: before/after logged, attrition recorded) | ✅ D-2 constrains any new table to the forward-migration path | ✅ T-05 (RR) → T-55; interim audit trail via T-54/T-67 events; assessment+posting ship without it | IMPLEMENTATION DETAIL |
| **BLK-3** | `group_pickups` canonicalization ALTER + backfill stance | ✅ data mechanics | ✅ collapse target decided (S-1: one canonical record) | ✅ spec §17.1 declares `allotment_pickups` absent / collapse required | ✅ T-03 (RR approves ALTER + migrate-vs-fresh); live evidence gathered this session: **16 legacy rows, 0 in legacy `allotment_pickup`** — trivial either way | IMPLEMENTATION DETAIL |

**Rule honored:** none of the 13 items is converted to a blocker; none may be quoted in code as a decided business rule (plan §13 closing rule); each ships as interface + alternatives + decision record. The plan schedules RR sessions at the exact phases that need them (§15 A2 = RR session #1 for DEF-1/2/3; ⛔ tasks wait at their phase gates).

**Also checked:** items *not* deferred anywhere — none found hiding as "details" (spec §21.1 check 14 holds: no lock syntax/SQL/index strategy stated as policy; §6.5 of this review found no hidden business decision).

---

## 24. Traceability Validation

Independent re-counts (not copied from completion reports). Method: extract all `TR-x.y` IDs from `11_TARGET_BUSINESS_RULES.md` (shorthand families expanded), all decision IDs from `12_DECISION_STATUS.md`, all confirmation evidence lines from §2b/§3/§4, all `T-xx` IDs from `14_…` §5, then match against plan §14 matrix rows (ellipsis-range notation expanded to literal IDs).

### 24.1 Readiness traceability check (condensed — full matrix is plan §14, re-verified here)

| Requirement family | Source | Implementation Task(s) | Test coverage | Ready? |
|---|---|---|---|---|
| TR-1.1–1.6, 10.1–10.6, 15.5 (facts/authority/quota≤physical/mapping) | D-6, D-6a, S-2, S-3, D-7 | T-11, T-13, T-37–T-42, T-63, T-66 | §9.2 rows 1, 17 | ✅ |
| TR-2.1–2.7 (blocks) | D-15, D-1 | T-08, T-09, T-16, T-18, T-19 | 25 | ✅ |
| TR-3.1–3.6, 13.1–13.5 (allotment/expiry) | D-1, D-15, D-10, S-2 | T-10, T-12, T-13, T-15, T-22 | 3, 4, 23, 25 | ✅ |
| TR-4.1–4.10, 9.1–9.7 (pickup) | D-7, D-9, D-10, S-1, D-11, D-13, **D-4** | T-20–T-27, T-29, T-31, T-33, T-56–T-61 | 2, 4, 8, 18, 20, 24 | ✅ (TR-4.3 via NB-3 note — covered by T-56–58 family) |
| TR-5.1–5.5 (release) | D-12, D-3 | T-43, T-44 | 12, 18 | ✅ |
| TR-6.1–6.9 (wash/attrition) | D-3, D-14, D-6; **6.6 [DEFERRED]** | T-45–T-48, T-52–T-55 | 10–13, 15, 16 | ✅ |
| TR-7.1–7.5 (stop sale) | S-4 | T-12, T-14, T-37 | 23 | ✅ |
| TR-8.1–8.6 (shoulder) | D-13, D-2 | T-01, T-02, T-49–T-51 | 14 | ✅ |
| TR-9.x (see pickup row) | S-1, D-10, D-4 | T-21, T-22, T-28, T-32, T-68, T-69 | 4, 19, 20 | ✅ |
| TR-11.1–11.6; **11.7 [DEFERRED]** | D-8, D-9 | T-56–T-61 | 8, 9, 10 | ✅ |
| TR-12.1–12.6 (cancellation) | D-4, S-5 | T-23, T-24, T-32, T-36 | 5, 6, 7, 9 | ✅ |
| TR-13.3/13.4 (expiry) | D-15, D-3 | T-15, T-10, T-62 | 23, 25 | ✅ |
| TR-14.1–14.5 (isolation) | S-6 | T-59, T-65 | 18 | ✅ |
| TR-15.1–15.7 (legacy/schema) | D-1, D-2 | T-01, T-02, T-06, T-65, T-69 | 19 | ✅ |
| D-5 (every event consumed) | D-5 | T-40, T-62, T-63, T-64 | routing matrix + static coverage | ✅ |

### 24.2 Count validation

| Target | Claimed | Independently verified | Result |
|---|---|---|---|
| Target rules | 97 (95 decided + 2 deferred) | **97 unique TR IDs** extracted; **95 DECIDED + 2 DEFERRED** (TR-6.6, TR-11.7 row-inspected); plan §14 names **95 literal + both deferred as `[DEFERRED]` rows** — TR-10.3 and TR-4.3 absent from §14 rows but anchored in plan body (WS-05/T-40; TR-11.x family + scenarios 8/9) | **97/97 ✅** (NB-3) |
| Decisions | 22/22 | D-1…D-15 + D-6a + S-1…S-6 = 22 IDs, each present in spec §20.1 **and** plan §14 | **22/22 ✅** |
| Confirmations | 6/6 | D-2, D-4, D-6a, D-10, S-1, S-3 — each has a dedicated plan §14 confirmation row + spec header row | **6/6 ✅** (NB-2: spec §21.1 mislabels two as S-2/S-5) |
| Implementation tasks | 70/70 | T-01…T-70 contiguous, unique, all mapped to WS + §15 order + ≥1 verification command; no task orphaned from the matrix (every task appears in ≥1 §14 row or its WS/§9 anchor) | **70/70 ✅** (NB-6 nits) |
| Workstreams | 11/11 | WS-01…WS-11 all validated (§8) | **11/11 ✅** |

**If any mapping were incorrect, exact IDs:** two matrix-row omissions are named — **TR-10.3** (§14 row 1 lists TR-1.1/1.2/1.3/10.1/10.2/10.6/15.5 — 10.3 skipped) and **TR-4.3** (§14 pickup row lists 4.1/4.2/4.4/4.5/4.6 — 4.3 skipped). One spec self-check mislabeling is named — **spec §21.1 check #2**. No other incorrect mapping found.

---

## 25. Findings Register

Every finding is classified as exactly one of: **BLOCKER** · **NON-BLOCKING NOTE** · **IMPLEMENTATION DETAIL** · **FALSE POSITIVE / ALREADY COVERED**.

### BLOCKER (0)

*None. No finding met the bar of "implementation cannot safely begin without resolving it" backed by concrete evidence.*

### NON-BLOCKING NOTE (6)

| ID | Finding | Evidence | Affected tasks | Why non-blocking |
|---|---|---|---|---|
| **NB-1** | Plan current-state inventory counts are off: §1.1 says 24 command handlers / 6 query handlers / 4 adapters; §3 row 11 says hooks ×40 / views ×5 | Actual on disk: **25** `*.handler.ts` under `application/commands/`, **5** under `queries/`, **3** adapters, **36** exported hooks, **4** views (aggregates 3, entities 4, VOs 7, events 22, repos 3, components 4 all ✅ match) | none functional — descriptive text only | Every file a task names exists (§5.1); no task depends on the counts; correct the numbers in passing during implementation docs |
| **NB-2** | Spec self-check lists the 6 confirmations as "(S-2, S-5, D-6a, D-10, S-1, S-3)" | `13_…` line 819 vs header line 13, `12_…` §2b line 49 / §4 lines 75–77 = "(D-2, D-4, D-6a, D-10, S-1, S-3)" | none | Wording error in a verification table; all 8 distinct items represented in §20.1/§14; no rule changes; spec stays FINAL/unedited |
| **NB-3** | Plan §14 matrix literally names 95/97 TRs — **TR-10.3** and **TR-4.3** missing from matrix rows | §14 row text vs TR inventory (§24.2); TR-10.3 anchored at plan WS-05 §4 + T-40 action; TR-4.3 anchored at §14 TR-11.x row + §9 scenarios 8/9 → T-56–T-58 | §14 wording (not tasks) | Semantic coverage intact and test-anchored; matrix row add is a documentation edit at implementation time |
| **NB-4** | Migration bookkeeping: live `_prisma_migrations` has 51 rows for 49 directories; 2 failed-then-retried duplicate records; no unique index on `migration_name` | read-only SQL this session: duplicates = `20260817000000_add_checkin_sessions_and_guest_documents`, `20260910_b1_trx_codes` (retried OK); `pg_indexes` = pkey only; plan says "51 applied" (§1.1/§1.6/WS-01) | T-06, T-70 (`migrate status/deploy` runs) | All 49 migrations ultimately applied; pre-existing condition (not introduced by the plan); substantive plan claims (no GBA-table migrations) re-verified true; account for rows in gate runs |
| **NB-5** | FIND-3 PUT-release path is **dormant**: `groupBookingApi.releaseAllocation` (PUT, `api.ts:141-146`) has no active caller; `useReleaseBlockAllocation` imported but never invoked | `GroupBookingDetailView.tsx:4` (import only); only active release UI = `AllotmentDetailView.tsx:677` → allotment POST path (`hook:342` → `api.ts:317`); repo-wide grep shows no other callers | T-44 | D-12 fix still required and one-line; impact/priority lower than plan states; no production path breaks today |
| **NB-6** | Plan-internal cross-reference nits: (a) §6.1 graph omits 3 edges stated in task text (T-42→T-39, T-52→T-45, T-62→T-67); (b) T-55 has no `Files:` line; (c) T-62 vs T-54/T-46 both claim creation of `AttritionAssessed`/`AllotmentWashExecuted` | task-text prereqs vs §6.1 edges; T-55 body; §5.10 vs §5.8/§5.6 files | none functional | §15 order already respects all three edges (no wrong ordering); T-55 Files = store TBD by RR (mirrors T-04's explicit TBD); class create-once ownership resolves at implementation with T-62's static test tolerant/sequenced |

### IMPLEMENTATION DETAIL (13)

| ID | Item | Why implementation detail (not blocker) |
|---|---|---|
| **ID-1…ID-10** | DEF-1…DEF-10 (§23 table) | Each answers all four §23 questions affirmatively: mechanism/tooling-shaped, no business decision required, spec-constrained, plan-owned resolution with alternatives + RR session already scheduled (§15) |
| **ID-11** | BLK-1 — `block_type` storage | The rule (guaranteed never washes) and vocabulary (S-3) are decided; spec §17.1 already declares the column; T-04 confirms location; wash date-mechanics ship before it |
| **ID-12** | BLK-2 — wash/release/attrition store | Audit requirements decided (TR-6.1/6.5/6.9); D-2 constrains the store; T-05/T-55 resolve with interim event/audit trail (T-54/T-67) |
| **ID-13** | BLK-3 — pickup canonicalization ALTER + backfill | S-1 decides the target; ALTER shape is forced by live schema (NOT NULL / no allotment ref — re-verified); backfill is data mechanics with evidence gathered (16 rows); T-03 RR-gated, T-68 flag-staged |

### FALSE POSITIVE / ALREADY COVERED (10)

| ID | Apparent concern | Why already covered |
|---|---|---|
| **FP-1** | "USED-voucher cancel correction missing (current code cancel-from-ISSUED only)" | Plan WS-03 target line + **T-23** implements both S-5 branches + scenarios 6/7 — explicitly required by Stage-5 §15 and present |
| **FP-2** | "No active frontend caller supplies `reservationId`" | This fact *produced* decision D-10=B (recorded `12_…` §2b); T-22 backend creates the real reservation; response real-or-null; no conflicting frontend contract invented |
| **FP-3** | "Zero GBA tests / harness env-gated" | §9 strategy is create-scope for exactly that reason; harness + consumer spec + availability suites verified existing; env var = §27 setup item |
| **FP-4** | "`reservation.checked_out` event does not exist (FIND-2)" | T-30 producer + T-33 consumer + T-62 mapping + grep gate |
| **FP-5** | "Unscoped reservation UPDATE in cancel-pickup (FIND-5)" | T-24 hotel-scoped rewrite + T-59 static gate + scenario 18 isolation matrix |
| **FP-6** | "D-11-violating `ILIKE … LIMIT 1` live in adapter" | Re-verified at `:83/:246` — **T-25** targets exactly that file/lines + scenario 24 |
| **FP-7** | "`events.consumer` has no GBA cases (`default: warn`)" | T-40 + T-63 (INV-19 static test) + T-64 — the plan's core WS-10 work |
| **FP-8** | "No migration creates GBA tables (FIND-4)" | Re-verified across all 49 migrations (incl. `20260920_groups_blocks_allotments` = `reservation_groups` only) — T-01/T-02/T-06 close it; this is decided work (D-2=A), not an open gap |
| **FP-9** | "Plan cites `§12.2` although plan §12 is the Risk Register" | References resolve to **spec §12.2** (13 transaction rows); plan T-57/T-58 split (rows 1–7 / 8–13) matches exactly — wording resolution, no gap |
| **FP-10** | "`shoulder_days_*` undeclared breaks fresh env (FIND-1)" | Already decided work under D-2=A → T-01/T-02; live columns re-verified; no decision needed |

---

## 26. Final Readiness Verdict

# ✅ READY WITH NON-BLOCKING NOTES

Selected of the three permitted verdicts because **zero genuine blockers exist** (a pure "READY FOR IMPLEMENTATION" would require zero findings of any kind) and implementation can **safely begin** while the 6 documented notes are tracked.

### Verdict statement

An implementation agent can begin executing the 70-task Phase 4 Implementation Plan — starting with Phase A (T-01/T-02/T-06 schema, T-56 seam + RR session #1, T-08…T-19 domain guards, T-30/T-62 event contracts, T-59 isolation sweep) — **without encountering an unresolved architectural, domain, dependency, data-model, transaction, API, migration, or integration blocker.** The plan's RR-gated and ⛔ items (§23: DEF-1…DEF-10, BLK-1/2/3) are mechanism/storage/data confirmations with documented alternatives, scheduled at the correct phases, requiring no business decision to be reopened.

### Counts

| Classification | Count | IDs |
|---|---|---|
| **BLOCKER** | **0** | — |
| **NON-BLOCKING NOTE** | **6** | NB-1 inventory counts · NB-2 spec §21.1 confirmation mislabel · NB-3 §14 matrix 95/97 (TR-10.3, TR-4.3) · NB-4 `_prisma_migrations` duplicates/49-vs-51 · NB-5 dormant PUT release caller · NB-6 plan cross-reference nits |
| **IMPLEMENTATION DETAIL** | **13** | ID-1…ID-10 (DEF-1…DEF-10) · ID-11 (BLK-1) · ID-12 (BLK-2) · ID-13 (BLK-3) |
| **FALSE POSITIVE / ALREADY COVERED** | **10** | FP-1…FP-10 (§25) |
| **Total findings** | **29** | |

### Validation summary

| Validation | Result |
|---|---|
| 70-task validation | **PASS** — 70/70 audited: no missing/duplicate/orphan task, no circular/incorrect dependency, no wrong order, no non-existent-component dependency, no unresolved-decision dependency, all acceptance criteria verifiable (NB-6 nits only) |
| 97-rule validation | **PASS** — 97/97 traced (95 decided → tasks/tests; 2 deferred → DEF-1/DEF-2); §14 literally names 95 with TR-10.3/TR-4.3 anchored in plan body (NB-3) |
| 22-decision validation | **PASS** — 22/22 re-derived and matched to §14 rows; high-risk decisions behaviorally spot-checked (§6.2) |
| 6-confirmation validation | **PASS** — 6/6 (D-2, D-4, D-6a, D-10, S-1, S-3) evidence re-checked; spec §21.1 mislabel recorded (NB-2) |
| 11-workstream validation | **PASS** — WS-01…WS-11 (§8) |
| Quality Gate (§27 of the stage instruction) | **PASS** — appendix A below |

### If BLOCKED were the verdict, blocker IDs would be listed here

Not applicable — **0 blockers**. The plan's own ⛔ labels (BLK-1/2/3) are task-level gates routed to scheduled RR confirmations (ID-11…ID-13), not review-level blockers.

**No redesign was performed; no alternative architecture was proposed; no decision was reopened; no fix was implemented.**

---

## 27. Stage 6 Entry Conditions

Stage 6 — Implementation may begin as a separate instruction when:

1. **This document is accepted** as the Stage 5 gate record (`docs/availability/phase-4/15_READINESS_REVIEW.md`, verdict §26).
2. **RR session #1 scheduled at Phase A2** (plan §15) resolves, as *mechanism confirmations*: DEF-1 wash transport, DEF-2 lock mechanism + isolation level, DEF-3 assertion-port shape — alternatives already enumerated in plan §13 with in-repo evidence (BullMQ infra, `ReadCommitted` harness, existing assertion services).
3. **RR confirmations obtained before their phase gates:** T-03 (ALTER + backfill stance for 16-row `allotment_pickups`), T-04 (`block_type` storage — implement as spec §17.1 declares unless a concrete reason surfaces), T-05/T-55 (wash/attrition store within D-2), T-41/T-47 (final consult wiring; wash transport), T-56 mechanism pick. None requires reopening a business decision (§23).
4. **Environment setup:** export `AVAILABILITY_TEST_DATABASE_URL` against compose Postgres `xylo-postgres` for the postgres spec suites; confirm `pnpm db:generate` baseline (harness + compose verified reachable this session).
5. **Track the 6 non-blocking notes** (NB-1…NB-6) as documentation/cross-reference chores during implementation — correct plan counts, spec §21.1 label (if the spec is ever revised), §14 matrix rows for TR-10.3/TR-4.3, `_prisma_migrations` duplicate awareness in T-06/T-70 runs, T-44 caller-list precision, §6.1 edge additions.
6. **Honor the hard ordering rules:** consumers (T-32/33) before FO raw-SQL removal (T-31); seam (T-56) before intake rewrites; A3 rebuild (T-38) before F-18 removal (T-39); wash dormant behind `gba.wash.schedulerEnabled` until DEF-1 pick; canonical read parity (T-28) before canonical write (T-68).
7. **No code/schema/migration/API/test/data changes precede Stage 6** — confirmed: this review was read-only (SELECT-only database access; no repo files modified other than creating this document).

---

## Appendix A — Stage 5 Quality Gate (instruction §27) Verification

| # | Gate item | Result |
|---|---|---|
| 1 | No forensic audit unnecessarily repeated | ✅ audits cited as evidence only |
| 2 | No business decision reopened | ✅ 22/22 treated as locked (§6) |
| 3 | No final-domain rule silently changed | ✅ §6.5 |
| 4 | `13_FINAL_DOMAIN_SPECIFICATION.md` treated as authoritative | ✅ §3, §6 |
| 5 | `14_IMPLEMENTATION_PLAN.md` actually validated | ✅ §7 (all 70 tasks + graph + order) |
| 6 | All 70 tasks checked | ✅ §7.1/§7.3 |
| 7 | All 97 target rules checked | ✅ §6.4/§24 |
| 8 | All 22 decisions checked | ✅ §6.2 |
| 9 | All 6 confirmations checked | ✅ §6.3 |
| 10 | All 11 workstreams checked | ✅ §8 |
| 11 | Repository paths verified where relevant | ✅ §5.1 |
| 12 | Schema/migration reality checked | ✅ §5.2/§9 |
| 13 | Hotel isolation checked | ✅ §10 |
| 14 | Reservation integration checked | ✅ §11 |
| 15 | Availability integration checked | ✅ §12 |
| 16 | Pickup/voucher integration checked | ✅ §13 |
| 17 | Wash/release checked | ✅ §14 |
| 18 | Shoulder days checked | ✅ §15 |
| 19 | Attrition checked | ✅ §16 |
| 20 | Concurrency checked | ✅ §17 |
| 21 | Events/outbox checked | ✅ §18 |
| 22 | Legacy containment checked | ✅ §19 |
| 23 | API/frontend impact checked | ✅ §20 |
| 24 | Test coverage checked | ✅ §21 |
| 25 | Migration/cutover/rollback checked | ✅ §22 |
| 26 | Deferred items not incorrectly converted into blockers | ✅ §23 (13 items classified IMPLEMENTATION DETAIL) |
| 27 | Every finding has a classification | ✅ §25 (29 findings, all four classes used correctly) |
| 28 | Final verdict is evidence-based | ✅ §26 (every verdict claim cites document lines or on-disk/live-DB evidence) |
| 29 | No code modified | ✅ read-only inspection only |
| 30 | No schema modified | ✅ |
| 31 | No migration created or modified | ✅ |
| 32 | No API modified | ✅ |
| 33 | No test modified | ✅ |
| 34 | No database data changed | ✅ SELECT-only statements |

---

**End of `15_READINESS_REVIEW.md`.** Stage 5 complete. **STOP** — Stage 6 (Implementation) will be a separate instruction.

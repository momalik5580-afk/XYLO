# XYLO Availability Phase 4 — Implementation Plan

**Phase:** 4 — GBA / Allotment Integration
**Version:** 1.0 — **Status: COMPLETE / READY FOR READINESS REVIEW**
**Date:** 2026-09-30
**Document type:** Implementation plan (planning only — NO code, schema, migration, API, test, or data changes were made)

**Primary contract:** `docs/availability/phase-4/13_FINAL_DOMAIN_SPECIFICATION.md` (22/22 decisions, 6/6 confirmations, 97 = 95 DECIDED + 2 DEFERRED, 0 contradictions).
**Supporting evidence:** `01`–`12` audit/decision documents + `docs/design/gba-domain-spec.md` — evidence only, never overriding the contract.

**Verified state carried in:** 22/22 decisions resolved · 6/6 confirmations (2026-09-30) · D-2 live-DB PASS · 0 USER DECISION REQUIRED · 2 DEFERRED mechanisms (TR-6.6 scheduler, TR-11.7 lock mechanism).

---

## 1. Executive Summary

### 1.1 Current state (verified against repository)

- The **new-world GBA domain code exists** at `apps/api/src/modules/group-allotment/` — 3 aggregates, 4 entities, 7 value objects, 2 domain services, 4 ports, 22 event classes, 24 command handlers, 6 query handlers, 2 controllers, 3 Prisma repositories, 4 infrastructure adapters — but it does **not** yet conform to the Final Domain Specification in the atomicity, idempotency, guard-ordering, canonicalization, and event-consumer dimensions (findings F-1…F-13, C-1…C-17 remain open in code).
- The **Availability authority exists** at `apps/api/src/modules/availability/` (source adapter already reads GBA inputs with the decided filters: `DEDUCT_INVENTORY`, block-status whitelist, `HARD_COMMITMENT` + `ACTIVE` + validity window, stop-sales, `overbooking_limits`) with a Postgres test harness and assertion engine.
- The **reservation event backbone exists** (outbox → `events` BullMQ queue → `EventsConsumer` with idempotency), but `EventsConsumer` has **no GBA handlers** (`group_*` / `allotment_*` events hit `default: warn`) and no pickup-cancel/checkout consumption.
- **No `reservation.checked_out` event exists**; checkout emits `ReservationUpdatedDomainEvent(reservation_status: 'CHECKED_OUT')` and `FrontOfficeCheckedOut` (`check-out.handler.ts:254`, `:263`).
- **No GBA tests exist** — `apps/api/src/modules/group-allotment/` contains no `__tests__`; no web tests under `features/group-allotment/`.
- The **live database contains all new-world GBA tables** (verified read-only via `information_schema`, 2026-09-30) but **no migration in `packages/db/migrations/` creates any of them** — only `schema.prisma` declares them (`:17046`–`:17308`). This is exactly the D-2 declared-then-executed gap (TR-15.3/15.4).

### 1.2 Target state

Full conformance of GBA/Allotment/Pickup/Voucher/Availability/FO code paths to `13_FINAL_DOMAIN_SPECIFICATION.md`: single pickup ledger, single availability number, reservation-driven consumption lifecycle, atomic + idempotent + deterministic-concurrency operations, hotel-scoped statements, legacy read-only containment, wash/attrition implemented per D-3/D-13/D-14, and every mechanism the contract deferred kept deferred until Readiness Review.

### 1.3 Implementation objective

Translate the domain contract into ordered, verifiable engineering work — **without reopening any resolved decision or creating any new business rule**. Every task traces to a contract section/TR; every contract rule maps to at least one task.

### 1.4 Major implementation streams (workstreams)

| WS | Stream | Nature |
|---|---|---|
| WS-01 | Data Model / Schema | migrate (declare existing) + 2 blockers |
| WS-02 | Core GBA / Allotment Domain | modify |
| WS-03 | Pickup / Voucher | modify (rewrite paths) |
| WS-04 | Reservation Integration | create (consumers/events) + modify |
| WS-05 | Availability Integration | modify + create |
| WS-06 | Cut-off / Wash / Release | create (wash) + modify (release) |
| WS-07 | Shoulder Days | modify |
| WS-08 | Attrition / Financial | modify + 1 blocker |
| WS-09 | Concurrency / Transactions | modify (cross-cutting) |
| WS-10 | Events / Integration | create (handlers/contracts) |
| WS-11 | Legacy Containment / Reconciliation | read-only legacy + create (reconciliation) |

### 1.5 Major dependencies

1. **WS-01 schema declaration** (fresh-env reproducibility) precedes everything that persists.
2. **WS-09 transaction primitives** (unit-of-work adoption) gate WS-03/WS-04/WS-06 atomic rewrites.
3. **WS-10 consumer handlers** gate WS-04 cascade and WS-05 invalidation.
4. **WS-05 authority wiring** gates WS-03 pickup two-layer validation.
5. **Blockers BLK-1/BLK-2/BLK-3** (§1.6) gate parts of WS-02/WS-06/WS-08 — all other work proceeds around them.

### 1.6 Step 1 verification findings — CONTRADICTION / BLOCKER REGISTER

Verification of the domain contract against current code/schema/live DB (read-only). Per plan rules: **documented, not silently resolved. No specification was changed.**

| ID | Contract claim | Repository / live-DB reality | Impact | Status |
|---|---|---|---|---|
| **BLK-1** | Spec §17.1 declares `group_blocks.block_type`; §2.2/§4.6/TR-6.3 require the contract-type triad on blocks ("GUARANTEED_BLOCK never washes") | `block_type` **absent** in `schema.prisma` `group_blocks` (:17079–:17111), **absent in all 51 applied migrations, absent in live `group_blocks`** (verified `information_schema`) | Block-level wash type exclusion (WS-06) and type storage (WS-02) cannot be implemented without a storage location | ⛔ **BLOCKER — Readiness Review must resolve storage** (column vs alternative). Not resolved here. |
| **BLK-2** | Spec §17.1 lists `wash_logs / release_logs / attrition records` as "declared" audit-trail entities; TR-6.1/6.5/6.9 require logged before/after wash + attrition trail | **No wash/release/attrition tables exist** in `schema.prisma` or live DB (verified: zero matches for `wash`/`attrition` table names; only `penalty_due` at `schema.prisma:9718`, reservation-scoped) | Wash logging + attrition record persistence (WS-06/WS-08) have no declared store | ⛔ **BLOCKER — Readiness Review must resolve persistence** (new tables vs events vs logs) within D-2 constraints |
| **BLK-3** | Spec §17.1 declares `allotment_pickups` absent — "splits collapse into `group_pickups`" (S-1 one canonical record) | Live DB **has an undeclared `allotment_pickups` table** (full DDL verified; not in `schema.prisma`, not in any migration); current code writes it (`create-allotment-pickup.handler.ts:129-141`). `group_pickups.group_block_id` is **NOT NULL** and has **no allotment ref** (verified), so the collapse requires an ALTER | Canonical single pickup ledger (WS-03) requires schema ALTER + a backfill decision | ⚠️ **Spec target decided (S-1); ALTER + backfill mechanics flagged for Readiness Review** (data-migration decision, not a business rule) |
| **FIND-1** | TR-8.5 / §17.1: `shoulder_days_before/after` are "declared schema" (reproducible from artifacts) | Columns **exist in live `group_blocks`** (verified) and are written by `add-shoulder-days.handler.ts:113-125`, but are **absent from `schema.prisma` and all migrations** → fresh env breaks (undeclared drift) | WS-01 must declare them (decided work, TR-15.3/15.4) | ✅ Task, not a blocker (resolution already decided by D-2=A) |
| **FIND-2** | Spec §16.2 event contract lists `reservation.checked_out` | Event does not exist; checkout emits `ReservationUpdatedDomainEvent` + `FrontOfficeCheckedOut` (`check-out.handler.ts:254`, `:263`) | WS-04/WS-10 must add the decided event contract | ✅ Task (implementing the contract, not inventing) |
| **FIND-3** | D-12: canonical release verb is `POST`, frontend caller corrected | Controller is already `@Post` (`group-booking.controller.ts:247`, `allotment.controller.ts:239`) but frontend still calls **`api.put`** (`group-allotment.api.ts:146`) | Block release from UI fails | ✅ Task (fix caller to the decided contract) |
| **FIND-4** | TR-15.3/15.4 declared-then-executed | **No migration creates** `group_bookings`, `group_blocks`, `group_block_daily_allocations`, `group_pickups`, `allotment_contracts`, `allotment_daily_quotas`, `allotment_vouchers`, `allotment_stop_sales` (grep across all `migration.sql` = zero) — live DB has them (push out-of-band) | Fresh environment cannot reach the target schema | ✅ WS-01 baseline migration (decided work) |
| **FIND-5** | TR-14.1 hotel isolation on every statement | `cancel-allotment-pickup.handler.ts:66` runs `UPDATE reservations SET reservation_status = 'CANCELLED' WHERE id = $1` — **no `hotel_id` predicate** (C-7 family) | Cross-tenant write hazard | ✅ WS-09 hotel-scoping sweep task |

No other contract↔repository contradictions were found. Areas without contradictions were planned normally; areas with blockers are planned around (downstream tasks marked ⛔).

### 1.7 Major risks (summary — full register §12)

Non-atomic pickup paths today can partially commit (evidence: swallowed `rollbackQuota`, `create-allotment-pickup.handler.ts:114-123/:165-185`); wash implementation is net-new with a blocked type-exclusion; legacy `allotment_pickups` vs canonical `group_pickups` divergence during cutover; A1 invalidation currently misses all GBA events (consumer `default: warn`); hotel-isolation defects in raw SQL; undeclared schema breaks fresh-env reproduction.

---

## 2. Implementation Principles

Translated from `13_FINAL_DOMAIN_SPECIFICATION.md` §3/§14 — binding on every task:

1. **Hotel isolation (INV-1):** every read/write — including raw SQL — carries `hotel_id`; bare-id mutations prohibited; no header-only override of scope.
2. **Authoritative ownership (§3.1):** reservations own stay lifecycle; Availability owns sellable; GBA owns entitlement (quota/held/picked/released); pickup record owns association; FO owns room states and performs **no** GBA writes; financial owner owns the ledger.
3. **No duplicate source of truth (INV-7/INV-18/§19):** one pickup ledger, one availability number, one consumption truth (the reservation), one association truth (the pickup record), one guard formula (`quota − picked − released`).
4. **Availability authority (TR-10.1/10.2):** A1 is the sole sellable producer; GBA never computes/publishes an availability number; all eligibility filters live in the one adapter.
5. **Reservation-driven consumption lifecycle (INV-4/5):** the reservation decides whether consumption stands; voucher/pickup records follow; quota restores only iff the linked reservation is cancelled (S-5).
6. **Atomicity (INV-12):** the §12.2 unit commits wholly or not at all; compensating-swallow (current `rollbackQuota`) is prohibited; partial failure surfaces as an error.
7. **Idempotency (INV-3/6/10/11):** exactly-once counter mutation per business event; duplicate consume/cancel/wash converge to no-op or deterministic rejection; outbox→queue delivery is at-least-once, handlers idempotent.
8. **Deterministic concurrency (INV-13/15):** no lost updates, no silent last-write-wins; stale state → `CONFLICT`; version monotonicity; core inequalities hold after every commit (INV-14).
9. **Fail-closed where specified (INV-8, TR-10.4):** invalid facts are flagged `UNRESOLVED` with provenance and block trust — never silently clamped or rewritten; `quota > physical` flagged, not auto-repaired (S-2).
10. **Legacy containment (TR-15.1/15.2/15.5/15.7):** legacy tables and A4 counters read-only; never authoritative; no new legacy writes; `reservations` GBA columns secondary only.
11. **Observability / auditability (§8.2, §19):** wash/release/status changes emit domain events with before/after; correlation and idempotency keys flow through outbox; reconciliation is read-side only with alerts, never auto-repair.
12. **Deferred stays deferred (§11/§13):** scheduler, lock mechanism, assertion-port wiring, and other mechanism-shaped deferrals are presented as **alternatives requiring Readiness Review confirmation** — never implemented as if they were decided, never stated as business rules.

---

## 3. Current → Target Architecture Map

**Legend:** existing = keep as-is · modify = change behavior · create = new artifact · migrate = schema/migration work · remove = delete per contract · read-only-legacy = frozen · deferred = implementation-plan deferred.

| # | Current component (verified path) | Target component | Change required |
|---|---|---|---|
| 1 | `apps/api/src/modules/group-allotment/` aggregates/entities (`group-block.aggregate.ts`, `allotment.aggregate.ts`, `group-booking.aggregate.ts`; entities `group-pickup`, `daily-allocation`, `allotment-voucher`, `allotment-stop-sale`) | Same aggregates with conformed guards/state machines (§13, INV-14) | **modify** — add lifecycle transitions (D-15 `DRAFT→TENTATIVE`), voucher legal-transition enforcement (§6.3), guard ordering (stop-sale before quantity), guard bounds (`release ≤ held − picked`), rollup recomputation through aggregate (kill raw mutation) |
| 2 | 24 command handlers under `application/commands/` (multi-step, raw SQL, e.g. `create-allotment-pickup.handler.ts`, `consume-allotment-voucher.handler.ts:26`, `cancel-allotment-pickup.handler.ts:66`) | Atomic, idempotent, hotel-scoped handlers implementing §12.2 units | **modify** — rewrite pickup/voucher/release/cancel paths into single transactions; remove fabricated ids; remove unscoped SQL |
| 3 | `infrastructure/adapters/prisma-reservation-association.adapter.ts` (raw `INSERT INTO reservations` `:101`, `:265`) | Reservation-creation port that (a) applies D-11 guest order, (b) participates in the §7.1 two-layer validation path | **modify** — mechanism for assertion wiring `[DEFERRED]` (§11.5) |
| 4 | `infrastructure/adapters/prisma-folio.port.ts` / `prisma-billing-instruction.adapter.ts` (folio/folio_postings writes) | Same ports; used inside wash transaction for `ATTRITION_FEE` (TR-6.9) | **modify** — called within wash unit; ledger ownership `[DEFERRED]` (§13 D-6) |
| 5 | `domain/events/group-allotment.events.ts` (22 event types, all `IntegrationEvent` → outbox) | Same events + **missing contracts**: `AllotmentWashExecuted`, `ShoulderAllocationChanged` (or mapped to `group_block.allocation_changed` — mapping table §10.1), `reservation.checked_out` | **create/mapping** — spec §16 names ↔ existing types published as a mapping table; missing events added |
| 6 | `apps/api/src/modules/shared/events.consumer.ts` (reservation.created email + cache invalidation only; GBA events → `default: warn`) | GBA-aware consumer: (a) GBA events → availability invalidation, (b) `reservation.cancelled` → pickup cascade restore, (c) `reservation.checked_out`/checkout signal → pickup `CHECKED_OUT` | **create** — new handlers, all idempotent via `EventIdempotencyService` |
| 7 | `apps/api/src/modules/front-office/application/commands/check-out/check-out.handler.ts` raw `UPDATE allotment_pickups` `:392` and `UPDATE group_pickups` `:481` | FO checkout emits decided events only; **no GBA writes** (L-13 closed) | **modify (remove)** — pickup status transition moves to GBA consumer; master-folio auto-post (`:322-383`) retained (financial owner behavior, unchanged) |
| 8 | `apps/api/src/modules/availability/infrastructure/adapters/availability-source.adapter.ts` (A1 inputs; `allotmentStopSaleActive: false` constant at `:113`; clamp `:116`) | Same adapter with stop-sale display wiring per TR-7.2/7.3; filters unchanged (already decided-conformant) | **modify (small)** — see WS-05 |
| 9 | `apps/api/src/modules/activities/availability-sales.controller.ts` `@Get('availability/matrix')` `:237` (+ `:549` bulk-update, `:671` interval-update) — A3 computes its own availability | A3 rebuilt **on top of A1** outputs (D-6a = A); interim labeled-view option (B) allowed only while explicitly labeled | **modify** — replace local computation (`:430` area) with authority reads |
| 10 | `group-booking.controller.ts` `@Get('available-rooms')` `:95` (counts `room_status='AVAILABLE'`) + `group-allotment.api.ts:29` + `useAvailableRooms` (`use-group-allotment.ts:213`) | Retired (F-18, D-6) — callers read the Availability API instead | **remove** |
| 11 | Frontend `apps/web/features/group-allotment/` (hooks ×40, views ×5, components ×4, `group-allotment.api.ts` — release via `PUT` `:146` vs decided `POST`) | UI conformed: `POST` release, canonical pickup reads, availability figures from authority, wash/attrition surfacing | **modify** |
| 12 | Legacy tables `allotment` / `allotment_pickup` / `allotment_room_types` / `reservation_groups` / `reservation_block` / `allotment_ledger`; legacy `availability` (`schema.prisma:433`) written by `rates-inventory/crs-engine.service.ts` | **read-only-legacy**, never authoritative (TR-15.1); GBA never reads/writes A4 counters | **read-only-legacy** + reconciliation guards (WS-11) |
| 13 | Live undeclared `allotment_pickups` (current allotment-pickup ledger) | Retired after canonical collapse into `group_pickups` (BLK-3) | **migrate + read-only-legacy** |
| 14 | `reservations` GBA columns (`block_code`, `group_block_id`, `pickup_type`, …) | Retained secondary read-optimization (TR-15.7); pickup record remains truth | **existing** (no change; tests pin non-authority) |
| 15 | `packages/db/schema.prisma` `:17046-17308` + live GBA tables; no migrations create them | Declared **and** executed (TR-15.3/15.4): baseline migration + shoulder declaration + BLK-3 ALTER | **migrate** |
| 16 | `availability/infrastructure/reconciliation/availability-reconciliation.service.ts` + `platform/audit/audit-subscriber.service.ts:123` | Extended reconciliation for GBA counters; GBA audit events subscribed | **modify/create** |
| 17 | `common/database/unit-of-work/` (`ambient-transaction.ts`, `transaction-manager.ts`, `unit-of-work.ts`), `common/outbox/*`, `common/events/event-bus.ts`, `core/interceptors/idempotency.interceptor.ts` | Same primitives as the standard implementation vehicle for §12.2 units | **existing** (adopt, don't replace) |

---

## 4. Implementation Workstreams

Fields per workstream: **Plan ID · Domain area · Current location · Target behavior · Files affected · DB/schema impact · Migration impact · API impact · Event impact · Transaction impact · Concurrency impact · Idempotency · Tests · Dependencies · Ordering · Rollback · Verification.**
Classification tags: `existing` / `modify` / `create` / `migrate` / `remove` / `read-only-legacy` / `implementation-plan deferred` / ⛔ `BLOCKED`.

### WS-01 — Data Model / Schema

- **Plan ID:** WS-01 · **Classification:** `migrate` (+ `existing`, ⛔ blockers)
- **Domain area:** declared schema, relationships, hotel scoping, constraints, version fields, pickup canonicalization, shoulder structures.
- **Current location:** `packages/db/schema.prisma` (`group_bookings :17046`, `group_blocks :17079`, `group_block_daily_allocations :17113`, `group_pickups :17136`, `allotment_contracts :17161`, `allotment_daily_quotas :17198`, `allotment_vouchers :17221`, `allotment_stop_sales :17252`, analytics `:17275/:17293`, `overbooking_limits :7434`); live DB verified via `information_schema` (2026-09-30); `packages/db/migrations/` (51 applied; **none** creates the GBA tables — FIND-4).
- **Target behavior:** target schema = declared schema, executable from version-controlled artifacts (TR-15.3/15.4, D-2=A); shoulder columns declared (TR-8.5); one canonical `group_pickups` able to reference either a block or an allotment (S-1, BLK-3).
- **Required schema changes:**
  1. `migrate` — **baseline declaration migration**: `CREATE TABLE IF NOT EXISTS` (matching live + `schema.prisma` exactly) for the 8 new-world GBA tables + their unique keys/indexes already declared (`uq_group_blocks_hotel_code`, `uq_group_alloc_hotel_block_date_rt`, `uq_allotment_quota_hotel_allot_date_rt`, `uq_allotment_voucher_hotel_allot_code`, …). *Why:* fresh env cannot produce these tables from migrations (FIND-4); TR-15.3 demands reproducibility.
  2. `migrate` — **declare shoulder columns**: add `shoulder_days_before` / `shoulder_days_after` to `schema.prisma group_blocks` (live already has them — FIND-1) + include in baseline (guard `ADD COLUMN IF NOT EXISTS` for envs where absent). *Why:* TR-8.5/15.3; `add-shoulder-days.handler.ts:113` writes them today.
  3. ⛔ `migrate` — **BLK-3 pickup canonicalization ALTER**: `group_pickups.group_block_id` → nullable + add nullable `allotment_id` (+ FK) so allotment pickups can live in the canonical ledger. *Backfill decision (migrate live `allotment_pickups` rows or start fresh) = Readiness Review item — data mechanics, not a business rule.*
  4. ⛔ **BLK-1** `group_blocks.block_type` storage — **BLOCKED** (Readiness Review resolves location).
  5. ⛔ **BLK-2** wash/release/attrition persistence — **BLOCKED** (Readiness Review resolves store choice within D-2).
  6. `optional (Phase 2b scope)` — defense-in-depth CHECK constraints for INV-14 (`picked + released <= quota`, `picked <= contracted - released`, non-negative counters). Business invariant enforcement remains in code (TR-11.3); DB checks are belt-and-braces. Flagged `PHASE-2B`, not required to exit Phase 4.
- **Existing schema needing NO change:** `group_blocks` counters/`attrition_threshold`(:17095 default 80 ✓ D-14)/`cutoff_date`(:17088 ✓)/`inventory_policy`; `group_block_daily_allocations` counters + natural key; `allotment_contracts` (incl. `contract_type :17170`, `is_rolling_release :17175`, `release_days_before :17174`, `version :17184`); `allotment_daily_quotas` (+ `stop_sale_active`, `version`); `allotment_vouchers` (`status` default `ISSUED` ✓, real `reservation_id` ✓, unique `(hotel_id, allotment_id, voucher_code)` ✓); `allotment_stop_sales` (`APPLIED` default ✓); `overbooking_limits`; `reservations` (secondary columns kept — TR-15.7); availability assertion tables (`:17310+`).
- **Relationships / linkage:** pickup ↔ reservation via `reservation_id` (real id, D-10); pickup ↔ block `group_block_id`; pickup ↔ allotment via BLK-3 column; allocations/quotas FKs already declared.
- **Hotel scoping:** every table has `hotel_id` (verified in `schema.prisma` models); indexes `(hotel_id, …)` declared — keep; RLS = Phase 2b, out of scope.
- **Migration impact:** 1 new baseline migration (non-destructive, `IF NOT EXISTS`), 1 conditional ALTER (BLK-3), declared-column sync (FIND-1). **No data rewrite** except optional BLK-3 backfill (RR decision). Backfill otherwise: **not required, not prohibited — each item explicitly reviewed in §7.**
- **API impact:** none directly. · **Event impact:** none directly.
- **Transaction impact:** none (schema only). · **Concurrency impact:** preserves existing `version` columns consumed by WS-09.
- **Idempotency:** migrations must be re-run-safe (`IF NOT EXISTS`).
- **Tests required:** migration-from-scratch test (fresh DB + `pnpm db:migrate` → schema equals `schema.prisma` drift check); declaration tests (columns present). Pattern: SQL-split harness `availability-postgres.harness.ts` (`splitSqlStatements`).
- **Dependencies:** none (first). · **Ordering:** T-01/T-02 before any persistence task; ⛔ tasks wait on RR.
- **Rollback:** baseline migration reversible by `DROP TABLE IF NOT EXISTS` in down-script (only tables not yet in migration history — live data unaffected); BLK-3 ALTER reversible (`DROP COLUMN`/restore NOT NULL after verifying no orphan rows).
- **Verification criteria:** `npx prisma validate` + drift check (`prisma migrate diff --from-migrations --to-schema-datamodel`) reports zero diff for GBA models; fresh-DB migration test green; `pnpm typecheck` green.

### WS-02 — Core GBA / Allotment Domain

- **Plan ID:** WS-02 · **Classification:** `modify` (+ `create`, ⛔ blockers)
- **Domain area:** block lifecycle, allotment lifecycle, quota/picked/remaining, DEDUCT/NON-DEDUCT, HARD/SOFT/FREE-SALE/GUARANTEED_BLOCK/ROLLING_RELEASE, cut-off fields, stop sale, quota ≤ physical.
- **Current location:** `group-allotment/domain/aggregates/group-block.aggregate.ts` (status transitions, `washAllocation :332`, rollup `recalculateNights`), `allotment.aggregate.ts`, `group-booking.aggregate.ts`; `domain/services/lifecycle.service.ts`, `attrition-calculation.service.ts`; `value-objects/allocation-status.value-object.ts`, `daily-room-allocation.value-object.ts` (`canWash :63`, `wash :97`); handlers `create-group-block`, `create-allotment`, `set-allotment-quota`, `set-daily-allocation`, `apply/lift-allotment-stop-sale`, `delete-group-*`; `create-group-block` (F-12 code-generation defect); `add/remove-shoulder-days` raw SQL (F-11, C-9/C-10).
- **Target behavior:** §4/§5 of contract —
  - Block status machine exactly §13.1: `DRAFT → TENTATIVE → DEFINITE → OPEN_FOR_PICKUP → CLOSED`, `→ CANCELLED` from pre-closed; only `DEFINITE`/`OPEN_FOR_PICKUP` hold inventory (TR-2.2); explicit transitions only (TR-2.1); pickup requires eligible state (TR-2.3); block_code derived from real per-hotel count (F-12, TR-2.6).
  - Allotment status machine §13.4: consume only while `ACTIVE` + not expired (TR-3.3); expired/closed contribute zero (TR-13.2); type normalization via published S-3/TR-1.6 mapping (canonical triad + HARD/SOFT commitment) — mapping function only, storage location of canonical value verified against `contract_type`/`is_rolling_release` (⚠ see §13 note).
  - Guards: `picked + released ≤ quota` (TR-3.2), `release ≤ held − picked` (TR-5.1), `picked ≤ contracted − released` (TR-2.4), quota≤physical at create/set-quota (S-2/TR-1.5/3.5) with pre-existing violations **surfaced as flagged facts, never rewritten** (TR-10.4).
  - Guard order everywhere: contract eligibility → **stop-sale** → remaining (TR-7.4/4.5).
  - Stop-sale apply/lift updates `allotment_daily_quotas.stop_sale_active` **in the same transaction** as the `allotment_stop_sales` row (closes F-10 / A1 conflict detection at `availability-source.adapter.ts:84-87`).
  - All mutations flow through aggregates — **no raw counter SQL** (TR-2.5 event emission from the unit).
  - All rollups (`contracted_nights` etc., contract `total_*`) recomputed through aggregate in-unit (closes C-5 double-write).
- **Files affected:** aggregates ×3, VOs ×7, `lifecycle.service.ts`, handlers listed above, `group-block.repository.ts`, `allotment.repository.ts`, `group-allotment.module.ts` (register new commands), `permissions/group-allotment.permissions.ts` (existing `GROUP_BLOCK_WASH :14` used by WS-06).
- **DB/schema impact:** reads only, after WS-01. ⛔ type storage (BLK-1) blocks type-dependent behavior; ⛔ wash-log persistence (BLK-2) is WS-06.
- **Migration impact:** none beyond WS-01. · **API impact:** error semantics → deterministic codes + `409 CONFLICT` for TR-11.2 (see §8).
- **Event impact:** all 22 existing events kept; ensure every status/quota/stop-sale mutation publishes from within the transaction (outbox writer accepts `tx` — `event-bus.ts:36` `publish(event, tx)` ✓).
- **Transaction impact:** single aggregate-save per command (see WS-09 §12.2 table); quota+rollup+event one unit.
- **Concurrency impact:** `version` optimistic checks on allocations/quotas/contracts (mechanism `[DEFERRED]` per TR-11.7 — interface used, alternatives §13).
- **Idempotency:** create-commands idempotent on natural keys (hotel-scoped unique codes); set-quota re-run converges.
- **Tests required:** unit tests per transition (block §13.1, contract §13.4), guard tests (TR-2.2/2.3/2.4/3.2/3.3/3.5/5.1/7.4), stop-sale flag sync test, F-12 code test, S-2 create/set-quota rejection test + flagged-fact test, mapping-table tests (S-3).
- **Dependencies:** WS-01 (persistence declared). · **Ordering:** before WS-03 (pickup guards), WS-06 (release/wash use guards).
- **Rollback:** behavior changes behind handlers — revert code; no data migration. Statuses written pre-cutover remain readable (no status vocabulary change beyond adding `DRAFT`-first flow which already exists as schema default).
- **Verification criteria:** new unit suite green (`npx jest --testPathPattern group-allotment`), `pnpm typecheck`, A1 adapter spec (`availability-source.adapter.spec.ts`) still green, no raw counter SQL (grep gate in tests).

### WS-03 — Pickup / Voucher

- **Plan ID:** WS-03 · **Classification:** `modify` (rewrite) + `create` (canonical paths) + ⛔ BLK-3
- **Domain area:** canonical pickup record, voucher lifecycle, reservation linkage, consume/cancel, duplicate prevention, USED-voucher cancellation (S-5), anomalous linkage handling.
- **Current location:** commands `create-group-pickup.handler.ts` (raw SQL + `repo.save` non-atomic), `create-allotment-pickup.handler.ts` (quota save → port → raw `INSERT INTO allotment_pickups :129`; swallowed `rollbackQuota :165-185`; idempotency read of legacy table `:46-63`), `consume-allotment-voucher.handler.ts:26` (**fabricated `RES-${Date.now()}` — D-10 violation**), `cancel-allotment-voucher.handler.ts`, `cancel-allotment-pickup.handler.ts` (restore logic `:74-89`; **unscoped reservation UPDATE `:66`**; legacy table update `:90`), entities `group-pickup.entity.ts`, `allotment-voucher.entity.ts`, `prisma-reservation-association.adapter.ts`, `GET :id/pickups` route (`allotment.controller.ts:281`).
- **Target behavior:** contract §6/§7.1/§13.2/§13.3 —
  - **S-1:** ONE canonical record in `group_pickups` for both paths; vouchers = authorization artifacts; consumption without pickup record impossible (TR-4.9). Allotment pickups collapse into `group_pickups` (⛔ BLK-3 ALTER first).
  - **D-10-B:** consume creates/links a **real** reservation (real id, this hotel, or null) — fabricated ids prohibited (TR-3.4/4.4); second consume of `USED` voucher → deterministic rejection/idempotent return (INV-3).
  - **S-5:** cancelling `USED` voucher → quota restores **iff** linked reservation cancelled; command delegates to reservation-cancel semantics when reservation live (spec `:725`); `ISSUED` cancel restores directly (TR-12.5); `reservation already CANCELLED` → no second restore (TR-11.4).
  - **D-9:** reservation row + pickup record + counters (+ folio effects) commit atomically; swallowed compensation removed.
  - **Guard order:** eligibility → stop-sale → remaining (TR-7.4); containment `picked ≤ contracted − released` after commit (TR-4.6); checkout never changes counters (TR-4.8).
  - **D-11:** guest resolution order exact, hotel-scoped: explicit Guest ID → Passport ID → Email → create; `ILIKE`/fuzzy/`LIMIT 1` name binding prohibited; conflicting matches → explicit conflict error (TR-4.10/9.6).
  - **Anomalous linkage** (`USED` voucher, null `reservation_id`): surfaced as flagged anomaly in reads — **no guessed repair** (D-10 note 4, SOURCE-SILENT).
  - **Two-layer validation:** pickup passes A1/assertion consult (TR-4.1/10.6) — port shape `[DEFERRED]` (§13).
- **Files affected:** 4 pickup/voucher command handlers (+ commands), `create-allotment-voucher.handler.ts`, entities ×2, `allotment.aggregate.ts` (`consumeVoucher`, `pickupQuota`, `restoreQuota`), `group-block.aggregate.ts` pickup methods, `prisma-reservation-association.adapter.ts` (338 lines), `group-allotment.module.ts`, controllers (response shape), `use-group-allotment.ts` (`useCreateAllotmentPickup :401`, `useCancelAllotmentPickup :422`, `useConsumeAllotmentVoucher :305`…), `group-allotment.api.ts`.
- **DB/schema impact:** ⛔ BLK-3 ALTER; voucher `status` transitions need no schema change (`ISSUED/USED/CANCELLED` fit `allotment_vouchers.status`; `EXPIRED` derived at read or set by sweep — **mechanism noted §13, not a rule**); unique key `uq_allotment_voucher_hotel_allot_code` already exists (duplicate prevention at DB level ✓).
- **Migration impact:** BLK-3 only (+ optional backfill — RR).
- **API impact:** request/response unchanged except: consume no longer accepts/produces fabricated reservation ids (response `reservationId` real-or-null); error codes `VOUCHER_ALREADY_USED`, `QUOTA_INSUFFICIENT`, `GUEST_MATCH_CONFLICT`, `CONFLICT` (§8).
- **Event impact:** `group_pickup.created/cancelled`, `allotment.voucher_issued/consumed/cancelled` published **in-tx** with real `reservationId` in payload.
- **Transaction impact:** all §12.2 intake rows — one DB transaction via `common/database/unit-of-work` (WS-09).
- **Concurrency impact:** same-row counter races → exactly-one-wins (TR-11.1/11.2); mechanism `[DEFERRED]`.
- **Idempotency:** consume keyed by `(hotel_id, allotment_id, voucher_code)` + `EventIdempotencyService`-style dedup; cancel keyed by pickup id + status precondition; request-level dedup via existing `core/interceptors/idempotency.interceptor.ts` where wired (`[DEFERRED]` mechanics §13).
- **Tests required:** §9 mandatory scenarios — pickup consume, duplicate consume, real reservation linkage, cancel cascade preconditions, USED-voucher cancel (S-5 both branches), ISSUED cancel, no-checkout-restore, guest-order tests, hotel-isolation negative tests, atomicity (inject failure at each step → zero partial rows).
- **Dependencies:** WS-01 (⛔ BLK-3 for canonical writes — until resolved, allotment-pickup writes stay on the legacy table **marked as cutover-blocked**), WS-02 guards, WS-09 unit, WS-05 consult.
- **Ordering:** after WS-02 guards; before WS-04 cascade (cascade needs canonical record semantics).
- **Rollback:** handler-level revert; canonical-write switch is flag-controlled (§11 rollout); no destructive data change without the BLK-3 backfill decision.
- **Verification criteria:** unit + wiring + Postgres specs green; grep gate: no `RES-${` fabrication, no `UPDATE reservations ... WHERE id = $1` (unscoped), no `rollbackQuota` swallow; typecheck green.

### WS-04 — Reservation Integration

- **Plan ID:** WS-04 · **Classification:** `create` (events/consumers) + `modify` (FO handler) + `existing` (reservation module)
- **Domain area:** reservation creation/linking, cancellation cascade, pickup state sync, check-in, checkout, no-show, extension/modify non-behavior, idempotency — **respecting ownership** (§3.1: reservations own stay lifecycle; GBA owns pickup writes).
- **Current location:** `reservations/domain/events/reservation-domain.events.ts` (`ReservationCancelledDomainEvent → 'reservation.cancelled'` `:26-35`), `reservation.repository.ts:791` (publish in-tx), `shared/events.consumer.ts` (cache invalidation only — **no pickup cascade**), `front-office/.../check-out/check-out.handler.ts` (**raw pickup updates `:392` `allotment_pickups`, `:481` `group_pickups`**; status transition `:174-177`; events `ReservationUpdatedDomainEvent` `:254`, `FrontOfficeCheckedOut` `:263/:274`; folio auto-post `:322-383`), `cancel-allotment-pickup.handler.ts:66` (reverse cascade, unscoped).
- **Target behavior:** contract §7.2/§7.3/§13.3/§13.6 —
  1. **`reservation.cancelled` → GBA consumer (idempotent):** pickup → `CANCELLED`, unlink (TR-9.2), restore counters **exactly once** (TR-12.1/12.3), invalidate availability (TR-12.6). Precondition check (pickup already `CANCELLED` → no-op) prevents double-restore when the `cancel-allotment-pickup` command itself cancels the reservation (TR-11.4).
  2. **Checkout → pickup `CHECKED_OUT`, counters unchanged (TR-4.8/9.4):** add decided event contract `reservation.checked_out` (FIND-2) emitted from `check-out.handler.ts` at the status-transition point (inside the existing `client` transaction, like `:254`); GBA consumer writes pickup status. **Remove** raw `UPDATE` statements at `:392`/`:481` (L-13/FO-no-GBA-writes). Master-folio auto-post `:322-383` **retained unchanged** (financial-owner behavior).
  3. **Pickup-created reservations:** first-class rows with real ids (WS-03); confirmation semantics via reservation domain — **mechanism `[DEFERRED]`** (TR-9.7 §13); association on pickup record, bare columns secondary (TR-15.7).
  4. **Check-in:** voucher redemption = reservation check-in (§7.3); **no pickup counter changes** — verification task only.
  5. **No-show:** **no quota restore** (§7.3) — verification task only; pickup record status after no-show = SOURCE-SILENT → **no behavior added**.
  6. **Extension / overstay / modify / room-type-rate change:** SOURCE-SILENT in contract §7.3 → **plan implements NO new pickup/counter behavior**; adds a regression guard asserting pickup counters are untouched by these flows (preserves "no rule = no change"), plus an explicit non-goal note in code review checklist.
  7. **Cancel-pickup command ↔ cascade interaction:** command keeps its existing reservation-cancel side effect (behavior not re-decided) but must be hotel-scoped (`hotel_id` predicate — FIND-5) and its restore must be the single restore for that business event.
- **Files affected:** `check-out.handler.ts` (remove `:390-397`/`:479-485` raw SQL; add event emit), `reservation-domain.events.ts` (new `reservation.checked_out` IntegrationEvent) or shared events package (`packages/shared/src/events/index.ts` — verify export location at implementation), `shared/events.consumer.ts` (+ new handler methods), possibly a dedicated `group-allotment` consumer if routing preferred (registration in `group-allotment.module.ts` — decide at implementation as plumbing, not policy), `cancel-allotment-pickup.handler.ts`, `check-out.handler.spec.ts` (update).
- **DB/schema impact:** none.
- **Migration impact:** none. · **API impact:** none (events internal).
- **Event impact:** **producers:** `reservation.checked_out` (new); **consumers:** GBA pickup-cancel handler, GBA checkout handler — both registered in the `events` processor, idempotent, order-tolerant (no ordering guarantee — §7.2 contract).
- **Transaction impact:** emission in-tx (existing pattern `eventBus.publish(..., client)`); consumer writes are single-row updates + counter unit (WS-09).
- **Concurrency impact:** consumer vs command races on same pickup → status-precondition + version (INV-13/15).
- **Idempotency:** consumer idempotency via `EventIdempotencyService` (job.id dedup, `events.consumer.ts:40-43` ✓ pattern) + status-precondition no-ops (INV-6).
- **Tests required:** cascade restore exactly-once (incl. cancel-pickup-command-then-event sequence), checkout → `CHECKED_OUT` no counter change, no-show no-restore, check-in no-change, modify/extend counter-preservation guard, FO handler has zero GBA-table SQL (grep test), hotel-scoped cascade negatives, order-tolerance (deliver `reservation.cancelled` twice / out of order).
- **Dependencies:** WS-03 (canonical semantics), WS-09 (consumer unit), WS-10 (event contract registration).
- **Ordering:** after WS-03; before WS-11 reconciliation (needs cascade live).
- **Rollback:** feature-flag the new consumer handlers (§11); removing the FO raw SQL is safe only when consumer is active → **activation ordering: consumer first, then FO removal** (§13 sequence).
- **Verification criteria:** unit/wiring/Postgres specs green; `grep -r "UPDATE group_pickups" apps/api/src/modules/front-office` empty; `grep reservation.checked_out` hits producer+consumer; typecheck green.

### WS-05 — Availability Integration

- **Plan ID:** WS-05 · **Classification:** `modify` + `create` + `remove` (F-18) + `[DEFERRED]` (assertion port)
- **Domain area:** A1 integration, D-6a A3 rebuild, allotmentRemaining/physicalAvailable/sellableAvailable, explicit overbooking, no competing number, assertion integration where applicable.
- **Current location:** `availability/infrastructure/adapters/availability-source.adapter.ts` (A1 inputs — **already decided-conformant filters** `:49-52`, eligibility `:66`, integrity flags `:68-88`, overbooking `:116`; but `allotmentStopSaleActive: false` hard-coded `:113`), `reservation-consumption.adapter.ts`, `unresolved-restriction.adapter.ts`, `application/services/availability-snapshot.service.ts`, `availability-assertion.service.ts`, `domain/policies/snapshot-calculator.ts`, `infrastructure/reconciliation/availability-reconciliation.service.ts`, `api/controllers/availability.controller.ts`; A3 = `activities/availability-sales.controller.ts:237` matrix (`:430` local computation, `:549/:671` writes); F-18 = `group-booking.controller.ts:95` + `group-allotment.api.ts:29` + `useAvailableRooms :213`; legacy A4 = `rates-inventory/crs-engine.service.ts`; web `components/AvailabilitySales.tsx`.
- **Target behavior:** contract §11 —
  1. **Single authority:** no second sellable number — F-18 retired (`remove`), A3 rebuilt on A1 reads (D-6a = A; interim labeled-view (option B) **only** with explicit "derived view, not sellable availability" label).
  2. **A1 inputs unchanged in meaning** (already TR-1.3/10.2-conformant): GBA `gbaRemaining = contracted − picked − released` over eligible blocks; `allotmentRemaining` for HARD + ACTIVE + in-validity; stop-sale facts; transient `overbooking_limits` allowance (TR-10.5 — the sole overbooking facility; `Math.max` clamp at `:116` reads that input only).
  3. **Stop-sale display (TR-7.2/7.3):** wire `allotmentStopSaleActive`/restriction outcome so applied allotment stop-sale renders remaining 0 for selling display while quantity stays untouched — replace the `false` constant at `:113` with the computed fact (display semantics only, allotment-scoped; existing `UNRESOLVED` promotion guard `:98-102` kept).
  4. **Invalidation:** GBA events → availability cache/snapshot invalidation via WS-10 consumers (TR-10.3); stale-by-design prohibited.
  5. **Assertions:** pickup-created reservations participate in the assertion lifecycle (TR-4.1); port shape **`[DEFERRED]`** — alternatives in §13 (RR confirms; `inventory-commitment.port.ts` dead port is the replacement target).
  6. **Overbooking:** never promoted to contract overcommit; `quota > physical` rows stay flagged UNRESOLVED (TR-10.4/10.5).
- **Files affected:** `availability-source.adapter.ts` (+ its spec), A3 controller `availability-sales.controller.ts` (matrix section), `group-booking.controller.ts` (remove route), `group-allotment.api.ts:29` + `use-group-allotment.ts:213` (remove) + calling views (`features/group-allotment/views/`), `events.consumer.ts` (invalidation handlers — shared with WS-04/WS-10), `AvailabilitySales.tsx` consumer updates.
- **DB/schema impact:** none. · **Migration impact:** none.
- **API impact:** **remove** `GET /group-bookings/available-rooms` (F-18/D-6); A3 `availability/matrix` response **shape preserved** (frontend contract) but values sourced from authority — response-change review in §8.
- **Event impact:** consumes GBA event family (WS-10).
- **Transaction impact:** none (read-side) except invalidation side-effects.
- **Concurrency impact:** none (reads).
- **Idempotency:** invalidation is idempotent (cache delete).
- **Tests required:** `availability-source.adapter.spec.ts` extended (stop-sale display TR-7.2/7.3), A1 vs A3 parity test (same inputs → same sellable), F-18 route-absent test, no-second-number test (grep/HTTP 404), overbooking-flag tests, invalidation consumer tests, existing postgres specs stay green (`availability-isolation`, `fail-closed`, `lock-order`, `assertion*`).
- **Dependencies:** WS-02 (filters depend on statuses), WS-10 (invalidation), WS-01 (none).
- **Ordering:** A1/stop-sale edits early (independent); F-18 removal after A3 rebuild (frontend callers switched); assertion port = RR-gated.
- **Rollback:** A3 rebuild behind flag with interim labeled view (decision-sanctioned); F-18 removal last (route deletion reversible by redeploy).
- **Verification criteria:** adapter specs + postgres suite green (set `AVAILABILITY_TEST_DATABASE_URL`), A3 parity test, typecheck + `pnpm lint` (web) green.

### WS-06 — Cut-off / Wash / Release

- **Plan ID:** WS-06 · **Classification:** `create` (wash commands/scheduler interface) + `modify` (release) + ⛔ BLK-1/BLK-2 + `[DEFERRED]` scheduler (TR-6.6)
- **Domain area:** block cut-off, T-1 single wash event, T-2 rolling N-day release, GUARANTEED_BLOCK no-wash, manual release override, date-granular release, idempotent wash, event consumers, atomic wash transaction.
- **Current location:** `group-block.aggregate.ts` `washAllocation :332` (+ `GroupBlockWashedEvent`), `daily-room-allocation.value-object.ts` `canWash/wash :63/:97`, `release-block-allocation.handler.ts`, `release-allotment-allocation.handler.ts`, `allotment.controller.ts:239 @Post(':id/release')`, `group-booking.controller.ts:247 @Post(.../release)` + `:267` shoulder routes, `attrition-calculation.service.ts`, events `group_block.washed`/`group_block.released`/`allotment.released`, permission `GROUP_BLOCK_WASH` (`group-allotment.permissions.ts:14`); **no cut-off/rolling wash handler, no scheduler, no wash-log store**.
- **Target behavior:** contract §8/§13 —
  1. **Manual release (modify):** guard `release quantity ≤ held − picked` (TR-5.1 — current allowance beyond `held` prohibited); guard to `DEFINITE`/`OPEN_FOR_PICKUP` blocks + `ACTIVE` allotments (TR-5.3); emit `group_block.released`/`allotment.released` **in-unit** with invalidation effect (TR-5.2); canonical verb **POST**.
  2. **Cut-off wash (create):** command `ExecuteCutOffWash` (block): at `cutoff_date`, one atomic unit — unpicked remainder across **core + shoulder dates** (no exclusion filter, TR-8.6b) → `released += unpicked`, `picked` untouched (§8.2), wash record (⛔ BLK-2), attrition assessment + posting (WS-08), `group_block.washed` event with before/after (TR-6.5); **GUARANTEED_BLOCK excluded by type** ⛔ **BLOCKED by BLK-1** (type storage) — wash-by-date mechanics proceed, type-exclusion line ships only after RR; re-run = **no-op** (spec §32 idempotency, INV-10); date-granularity = applies from start of the cutoff day (TR-6.4).
  3. **Rolling release (create):** command `ExecuteRollingRelease` (allotment): for `ROLLING_RELEASE` contracts, release unpicked quota for dates within `today … today + release_days_before` window (spec `:600-609` semantics carried), `release_days_before` honored (existing default 14), `GUARANTEED_BLOCK` never washes (TR-6.3), `FREE_SALE` needs no release; new event `AllotmentWashExecuted` (spec §16.1 — no existing equivalent; `allotment.released` exists for manual path); idempotent per contract+window.
  4. **Scheduler mechanism `[DEFERRED]` (TR-6.6):** plan defines **the interface only** — `WashSchedulerPort { runDueWashes(now: LocalDate, hotelId): Promise<WashRunReport> }` invoked by whatever transport RR selects (alternatives §13: BullMQ repeatable jobs — infrastructure already present (`infrastructure/bullmq/`), `@nestjs/schedule`, Temporal (compose service exists)). **No transport chosen here; transport is not a business rule.**
  5. **Data honesty (TR-6.5):** until the scheduled path is enabled, `cutoff_date`-driven UI metadata rendered as non-authoritative; once enabled, authoritative — controlled by rollout flag (§11).
  6. **Manual override coexists** (TR-5.5) — existing release commands remain.
  7. **D-12 fix:** frontend `api.put → api.post` (FIND-3, `group-allotment.api.ts:146`).
- **Files affected:** `group-block.aggregate.ts`, `allotment.aggregate.ts`, `daily-room-allocation.value-object.ts`, new command dirs `execute-cut-off-wash/`, `execute-rolling-release/` (+ register in `group-allotment.module.ts`), `release-*-allocation.handler.ts`, `group-allotment.permissions.ts` (existing), `group-booking.controller.ts`/`allotment.controller.ts` (only if manual trigger endpoints are **required** — contract does not mandate HTTP wash triggers; scheduler invokes handlers directly → **no new endpoints**), `group-allotment.api.ts:146`, `use-group-allotment.ts` callers, `AllotmentDetailView.tsx:663-720` (Wash & Release button `:720`).
- **DB/schema impact:** ⛔ BLK-1 (block type for exclusion), ⛔ BLK-2 (wash/release log store); counters live in existing allocation/quota tables.
- **Migration impact:** conditional on blockers (RR-resolved).
- **API impact:** frontend `PUT→POST` release fix; no new endpoints invented.
- **Event impact:** `group_block.washed` (exists, enrich before/after), **new** `AllotmentWashExecuted`, wash idempotency markers via wash record (⛔ BLK-2) or event dedup — **store choice = RR**.
- **Transaction impact:** §12.2 wash rows: counters + wash log + attrition + fee posting **one commit** (D-9/TR-11.6/6.7/6.9); release: counter + event one unit.
- **Concurrency impact:** concurrent wash vs pickup vs release on same rows → exactly-one-wins (TR-11.1); wash-vs-wash same range → second no-op; mechanism `[DEFERRED]`.
- **Idempotency:** deterministic wash keys `(block_id, cutoff_date)` / `(allotment_id, windowStart, windowEnd)` — re-execution no-op (INV-10).
- **Tests required:** §9 — single wash event per block, T-2 window math, GUARANTEED_BLOCK exclusion (**⛔ gated**), release bounds, idempotent repeated wash, concurrent wash/pickup, wash includes shoulder dates, stop-sale untouched by wash, event emission, hotel-scoping.
- **Dependencies:** WS-01 (⛔ BLK-1/2), WS-02 guards, WS-08 (attrition inside unit), WS-09 (unit), WS-10 (events).
- **Ordering:** manual-release guard fix early; wash create after WS-08 contract of posting; type-exclusion + wash record after RR.
- **Rollback:** wash commands dormant until scheduler flag on (§11); release guard tightening is safe (rejects previously-invalid releases); no schema rollback needed if BLK items implemented as `IF NOT EXISTS`.
- **Verification criteria:** unit/Postgres wash specs green, `grep api.put.*release` empty, scheduler interface present without hardwired transport (review gate), typecheck green.

### WS-07 — Shoulder Days (D-13 = A, uniform)

- **Plan ID:** WS-07 · **Classification:** `modify` + `migrate` (declare columns)
- **Domain area:** shoulder allocations with identical semantics to core days (P1 attrition base, P2 wash coverage, P3 unrestricted pickup).
- **Current location:** `add-shoulder-days.handler.ts` (raw `INSERT ... contracted_qty 0` `:96-106`, raw `UPDATE group_blocks` writing **undeclared** `shoulder_days_before/after` `:113-125`, date-range extension `:39-47`), `remove-shoulder-days.handler.ts` (unguarded removal — F-11/C-9/C-10), `group-booking.controller.ts:267/:277` routes, `useAddShoulderDays :101` / `useRemoveShoulderDays :112`, live columns (FIND-1).
- **Target behavior:** contract §9 —
  1. Ordinary allocations in extended range with same entities/guards/events/atomicity/hotel scope (TR-8.1).
  2. **Declared quantities only — silent qty-0 placeholders prohibited (TR-8.2):** handler must require explicit per-room-type quantities (request contract change in WS-08 API section) instead of inserting zeros.
  3. **Removal obeys no-removal-with-active-pickups guard + rollup recompute (TR-8.3)** — kill raw deletes (F-11).
  4. **Mutations emit domain events + invalidate Availability (TR-8.4)** — reuse/map `group_block.allocation_changed` (mapping §10.1).
  5. **Shoulder bookkeeping columns declared** (TR-8.5) — WS-01 task.
  6. P1/P2/P3 uniformity: no attrition filter (TR-8.6a), wash covers extended range (TR-8.6b), pickups unrestricted (TR-8.6c).
- **Files affected:** `add-shoulder-days.handler.ts`, `remove-shoulder-days.handler.ts`, `group-block.aggregate.ts` (shoulder methods + rollup), `group-booking.controller.ts` (request validation), `group-allotment.api.ts`/`use-group-allotment.ts` (request shape), `schema.prisma` (FIND-1 declaration).
- **DB/schema impact:** declare existing columns (WS-01 T-02). **Migration impact:** WS-01 baseline/conditional.
- **API impact:** shoulder add request must carry declared quantities (TR-8.2) — request-shape change traced to contract (§8).
- **Event impact:** shoulder changes publish `group_block.allocation_changed` (or new mapped event) in-unit.
- **Transaction impact:** allocations + block date/counters + rollup + event one unit (TR-8.1, §12.2).
- **Concurrency impact:** same counter unit rules.
- **Idempotency:** add with `ON CONFLICT DO NOTHING` currently `:100` — becomes idempotent-convergent via aggregate precondition (re-add = no-op or explicit error per existing behavior — behavior pinned by test, no new rule).
- **Tests required:** §9 — shoulder pickup guard parity, wash includes shoulder (with WS-06), attrition base includes shoulder (with WS-08), removal guard, qty-0 rejection, rollup recompute, event emission, hotel-scoping.
- **Dependencies:** WS-01 (T-02), WS-02 (aggregate guards), WS-06/WS-08 semantics for P1/P2 tests.
- **Ordering:** T-02 early; handler rewrite after WS-02 aggregate work.
- **Rollback:** code revert; columns already live (no data risk).
- **Verification criteria:** shoulder suite green; `grep shoulder_days` hits `schema.prisma`; no raw counter/deletion SQL in shoulder handlers.

### WS-08 — Attrition / Financial Integration (D-14 = A)

- **Plan ID:** WS-08 · **Classification:** `modify` + `create` + ⛔ BLK-2 + `[DEFERRED]` ledger ownership
- **Domain area:** 80% default threshold, uniform per-block threshold, wash-time calculation, shoulder inclusion, shortfall, penalty, `ATTRITION_FEE`, master-folio posting, atomicity with wash, idempotency, financial ownership boundary.
- **Current location:** `domain/services/attrition-calculation.service.ts` + `value-objects/attrition-policy.value-object.ts` (calculation exists — verify against D-14 math), `group_blocks.attrition_threshold` (default 80 ✓ `:17095`), `create-group-booking.command.ts` (`attritionThreshold?`), `prisma-folio.port.ts` (`INSERT INTO folio_postings :36`), `prisma-billing-instruction.adapter.ts` (`:194`), master-folio commands (`post-master-charge`, `record-payment`), `penalty_due` table (`schema.prisma:9718` — reservation-scoped, **not** the block attrition store), **no wash-time assessment call site, no attrition record** (BLK-2).
- **Target behavior:** contract §10 —
  1. Assessment **exactly once, at the wash, inside the wash transaction** (TR-6.7); read-only w.r.t. inventory; **no** evaluation at pickup/booking/availability/expiry; pre-wash reports carry no authority.
  2. Policy: single percentage per block, default 80%, uniform across room types (TR-6.8); `50–100%` bounds `[SPEC-CARRIED]`; rejected variants (VO 85/per-day/cumulative/room-type) **must not** exist in code paths (audit VO variants removed or normalized).
  3. Math: `minimumRequired = ceil(contracted × threshold / 100)`; `shortfall = max(0, minimumRequired − picked)`; base = wash range **including shoulder** (TR-8.6a).
  4. If `shortfall > 0`: `liabilityDue = shortfall × negotiatedRate` posted to block master folio as **`ATTRITION_FEE`** **within the wash unit** — never async (TR-6.9; posting via existing `prisma-folio.port` pattern; verify/create the `ATTRITION_FEE` trx-code reference during implementation — verification task, not a rule).
  5. Idempotency: wash idempotency (INV-11) ⇒ single assessment; no double penalty.
  6. **Ledger ownership of the posting `[DEFERRED]`** (spec open question 2): posting lands in existing financial/folio structures; **which module owns the ledger entry is an RR implementation dependency** — this plan does not invent a financial architecture. ⛔ BLK-2 blocks persistence of an attrition *record* (assessment is still computed + posted; the durable audit row awaits RR storage choice).
- **Files affected:** `attrition-calculation.service.ts`, `attrition-policy.value-object.ts`, new/modified wash command (WS-06) invoking it, `group-allotment.module.ts` (wiring), `prisma-folio.port.ts` (call — likely no change), `create-group-booking.handler.ts` (threshold validation at creation).
- **DB/schema impact:** threshold column exists ✓; ⛔ BLK-2 attrition record store.
- **Migration impact:** conditional on BLK-2. · **API impact:** none new (assessment surfaced via existing master-folio reads `GET :id/master-folio` + events).
- **Event impact:** `AttritionAssessed` (spec §16.1) — **create** (no existing equivalent), published in wash unit.
- **Transaction impact:** inside §12.2 wash row (single commit with counters + log + posting).
- **Concurrency impact:** wash-level (WS-06). · **Idempotency:** wash key (INV-11).
- **Tests required:** §9 — 80% default, custom threshold, ceil math, shortfall/liability math, shoulder-in-base, meetsThreshold boundary (picked == minimumRequired → no penalty), single-assessment on repeated wash, `ATTRITION_FEE` posting inside wash tx (assert same-tx via failure injection), no inventory mutation (read-only proof), allotment exclusion (TR-6.7 scope).
- **Dependencies:** WS-06 (wash unit), WS-01 (⛔ BLK-2 for record), WS-02 (aggregate rates), financial port.
- **Ordering:** calculation-service conformance test early; wiring with WS-06 wash; RR-gated persistence last.
- **Rollback:** dormant until wash enabled (flag); posting failure aborts wash unit (fail-closed — no partial wash).
- **Verification criteria:** attrition suite green; wash+attrition integration spec green; typecheck green.

### WS-09 — Concurrency / Transactions

- **Plan ID:** WS-09 · **Classification:** `modify` (cross-cutting) + `[DEFERRED]` lock mechanism (TR-11.7)
- **Domain area:** concrete transaction boundaries for every §12.2 operation; locks/versioning/isolation/retry/idempotency/partial-failure.
- **Current location:** `common/database/unit-of-work/` (`unit-of-work.ts`, `transaction-manager.ts`, `ambient-transaction.ts`, `index.ts` — `runWithTransaction` pattern used by reservations tests e.g. `reservation-cancel-wiring.spec.ts:120`), `common/events/event-idempotency.service.ts`, `core/interceptors/idempotency.interceptor.ts`, `common/outbox/outbox-writer.ts` (accepts `tx`), current GBA handlers **without** unit-of-work (evidence: multi-step + swallowed rollback `create-allotment-pickup.handler.ts:71-123`, `rollbackQuota :165-185`; raw sequential SQL in shoulder handlers; C-1…C-17).
- **Target behavior:** contract §12 —
  1. Implement each §12.2 row as **one unit-of-work transaction** (adopt `common/database/unit-of-work` — `existing` vehicle, don't build new infra): pickup-block, pickup-allotment, voucher issue/consume, cancel cascades, wash, manual release, attrition (nested in wash), status transitions, quota batch, shoulder.
  2. **No lost updates / conflict = `CONFLICT` / version monotonicity / inequalities-after-commit** (TR-11.1/11.2/11.3/11.5) — implement via the **version columns already on allocations/quotas/contracts** in a conditional-update pattern; **exact mechanism (optimistic retry vs row lock vs advisory lock), lock ordering, isolation level = `[DEFERRED]` (TR-11.7) — alternatives in §13, RR confirms.** The plan pins only the *interface*: all counter writes go through a single `CounterUnitOfWork.commit(expectedVersion, mutations, events)` seam so the mechanism is swappable.
  3. **Exactly-once per business event (TR-11.4):** status-precondition checks + outbox event ids + `EventIdempotencyService` on consumers; duplicate requests converge (INV-3/6/10/11).
  4. **Partial failure:** any step failure rolls the unit back and surfaces the error; **delete `rollbackQuota` swallow pattern**; no compensating-as-guarantee (D-9).
  5. **Hotel isolation sweep (TR-14.1 / FIND-5):** audit **every** `$queryRaw`/`$executeRawUnsafe` under `group-allotment/` (known: `cancel-allotment-pickup.handler.ts:66` unscoped; `create-allotment-pickup` rate-plan lookup `:86`; shoulder `:50/:65/:96/:113`; stop-sale/pickup idempotency reads) — add `hotel_id = $n` predicates; add a lint/test gate.
  6. **Lock ordering (deadlock avoidance):** document the acquisition order used by the seam (reservation → pickup → allocation/quota, matching availability's existing lock-order spec pattern `availability-lock-order.postgres.spec.ts`) — implementation concern validated by test, stated as mechanism not policy.
  7. **Isolation level:** current harness/tx pattern uses `ReadCommitted` (`availability-postgres.harness.ts:13-16`) — reuse unless RR opts otherwise (§13).
- **Files affected:** all WS-02/03/04/06/07/08 handlers; `group-allotment.module.ts`; new `infrastructure/transaction` helper colocated in group-allotment (or reuse shared unit-of-work — prefer shared); tests listed in §9.
- **DB/schema impact:** none required (version columns exist). · **Migration impact:** none.
- **API impact:** error mapping → `409 CONFLICT`, deterministic 4xx codes (§8). · **Event impact:** outbox-in-tx for every unit (`event-bus.publish(evt, tx)` ✓ exists).
- **Transaction/concurrency/idempotency:** as above — this workstream *is* those impacts for all WS.
- **Tests required:** concurrency Postgres suite: concurrent consume, concurrent cancellation, concurrent wash, double-delivery consumer, partial-failure atomicity (fault injection at each step asserts zero partial rows), version-stale rejection, hotel-isolation matrix (pattern: `availability-isolation.postgres.spec.ts`), unscoped-SQL grep gate.
- **Dependencies:** none for the seam; each handler adopts it individually (see §6 graph).
- **Ordering:** seam created **before** WS-03 rewrite; handler adoptions follow their WS.
- **Rollback:** seam adoption is code-level; behavior flags where risk is high (§11).
- **Verification criteria:** full jest suite green incl. new postgres specs (`AVAILABILITY_TEST_DATABASE_URL` set against compose Postgres `xylo-postgres` — infrastructure verified up), typecheck green, grep gates pass.

### WS-10 — Events / Integration

- **Plan ID:** WS-10 · **Classification:** `modify` + `create` (+ `existing` outbox plumbing)
- **Domain area:** producers, contracts, consumers, ordering, idempotency, failure handling, outbox, replay, observability.
- **Current location:** `common/events/event-bus.ts` (IntegrationEvent → `outboxWriter.save(msg, tx)` ✓; internal → dispatcher), `common/outbox/` (`outbox-writer.ts`, `outbox-publisher.ts` PENDING→`events` queue, `outbox-processor.ts`), `shared/events.consumer.ts` (`@Processor('events')`, job.id dedup, **no GBA cases** — `default: warn :57`), `event-idempotency.service.ts`, `group-allotment.events.ts` (22 IntegrationEvents ✓ in-tx via `publishFromAggregate`), `reservation-domain.events.ts`, `platform/audit/audit-subscriber.service.ts:123` (subscribes `reservation.cancelled`), `packages/shared/src/events/index.ts`.
- **Target behavior:** contract §16 —
  1. **Contract mapping table (§10.1 of this plan):** spec §16.1 names ↔ existing event types (e.g. `BlockCreated`↔`group_block.created`, `ManualReleaseExecuted`↔`group_block.released`/`allotment.released`, `CutOffWashExecuted`↔`group_block.washed`, `VoucherIssued`↔`allotment.voucher_issued`, …). **Missing contracts to create:** `AllotmentWashExecuted`, `AttritionAssessed`, `ShoulderAllocationChanged` (or explicit alias to `group_block.allocation_changed` — mapping chosen at implementation as naming mechanics, documented here), `reservation.checked_out` (FIND-2), stop-sale events exist ✓.
  2. **Consumers (mandatory INV-19 — every published event ≥1 consumer):** register GBA-event cases in `events.consumer.ts` (or a registered sibling processor): availability invalidation for **all** `group_*`/`allotment_*` events (TR-10.3), pickup cascade on `reservation.cancelled` (WS-04), pickup status on `reservation.checked_out` (WS-04). Zero `default: warn` for Phase-4 event types (add explicit no-op-with-log only where a consumer is intentionally not needed — review gate).
  3. **Ordering:** none guaranteed — consumers order-tolerant (§7.2); state-preconditions make late/duplicate deliveries safe.
  4. **Idempotency:** job-id + event-id dedup (`EventIdempotencyService` pattern `:40-43` ✓); handler idempotency (status preconditions).
  5. **Failure handling:** consumer throw → queue retry (existing behavior `:61-63` rethrows); **failures never swallowed** (D-4 rule); outbox `maxRetries: 5` (`event-bus.ts:34`) unchanged.
  6. **Replay:** outbox PENDING rows re-drivable; consumer idempotency makes replay safe — document runbook only (no new transport).
  7. **Transport design: none** (existing BullMQ `events` queue + outbox retained — "do not design transport infrastructure beyond necessity").
- **Files affected:** `shared/events.consumer.ts`, `group-allotment.events.ts` (new event classes), `reservation-domain.events.ts` (+ shared events export), `group-allotment.module.ts`, `audit-subscriber.service.ts` (optional GBA audit lines), tests `modules/shared/__tests__/events.consumer.spec.ts` (exists ✓ — extend).
- **DB/schema impact:** none. · **Migration impact:** none. · **API impact:** none.
- **Event impact:** the workstream above. · **Transaction impact:** producers publish in-tx (existing seam). · **Concurrency impact:** consumers serialized per job id by queue semantics; cross-handler races guarded by WS-09 preconditions.
- **Idempotency:** as above. · **Tests required:** §9 — consumer routing matrix, duplicate delivery, out-of-order delivery, retry-on-failure, GBA event → invalidation called, all-22-events-have-consumer static test.
- **Dependencies:** WS-04 (cascade logic), WS-06 (wash events), WS-08 (attrition event).
- **Ordering:** contract mapping table early (it pins names); consumers before FO raw-SQL removal (activation order §13).
- **Rollback:** consumers behind flags; new event classes additive (old consumers ignore unknown types via `default` — verify warn-only, not throw).
- **Verification criteria:** `events.consumer.spec.ts` extended green; producer-in-tx tests; static coverage test green; typecheck.

### WS-11 — Legacy Containment / Reconciliation

- **Plan ID:** WS-11 · **Classification:** `read-only-legacy` + `modify` (guards/reads) + `create` (reconciliation) + `remove` (only where contract requires: F-18 already in WS-05)
- **Domain area:** legacy counters, legacy availability projections, old GBA paths, duplicate pickup paths, read/write boundaries, reconciliation, cutover prerequisites.
- **Current location:** legacy tables `allotment`(`schema.prisma:105`)/`allotment_pickup :137`/`allotment_room_types :152`/`allotment_ledger :16068`/`reservation_groups :9100`/`reservation_block`/`reservation_groups` migration `20260920`; legacy `availability :433` counters written by `rates-inventory/crs-engine.service.ts` (A4, audit §3); **live undeclared `allotment_pickups`** written by current pickup path (BLK-3); `reservations` GBA columns (secondary — TR-15.7); `VouchersModal.tsx` (legacy UI, exists) vs `features/group-allotment/components/shared/VoucherPickupModal.tsx`; A3 old computation (WS-05); `events.consumer` legacy cache keys.
- **Target behavior:** contract §18/§19 —
  1. **Read-only-legacy:** GBA performs **zero** new writes to legacy tables; A4 counters **never read/written** by GBA (TR-15.1) — add regression tests + code-comment boundaries; legacy `allotment_pickups` becomes read-only **after** BLK-3 canonical cutover (write path switch = flag, §11).
  2. **Legacy reads that remain temporarily active:** legacy allotment list endpoints in `rates-inventory` (if any) and legacy UI routes stay until Phase 11 retirement — **not deleted** (contract does not require deletion).
  3. **Secondary columns:** `reservations.block_code/group_block_id/pickup_type` retained; tests assert they are never authoritative for counters/association (TR-15.7) — association reads come from `group_pickups`.
  4. **Duplicate pickup paths:** direct/manual intake remains supported (S-1 confirmed) but **both paths converge on the canonical record** (WS-03); legacy-table path retired under flag.
  5. **Reconciliation (read-side, no authority — §19):** extend `availability-reconciliation.service.ts` + add GBA reconciliation checks: (a) `picked` ⇔ count(ACTIVE+CHECKED_OUT pickups + USED vouchers) per hotel/date/room-type, (b) `picked + released ≤ quota` violations, (c) `quota > physical` rows (flag, never rewrite — TR-10.4/S-2), (d) `USED` voucher with null/missing reservation (anomaly), (e) legacy-vs-new drift report for cutover readiness — **alerts + UNRESOLVED flags only, no auto-repair**.
  6. **Audit:** subscribe GBA events in `platform/audit/audit-subscriber.service.ts` (pattern `:123`) for wash/release/status trail (partial mitigation while ⛔ BLK-2 unresolved).
- **Files affected:** pickup handlers (write-path switch), `availability-reconciliation.service.ts`, `audit-subscriber.service.ts`, tests, `group-allotment.api.ts` (canonical reads for `GET :id/pickups`), `useAllotmentPickups :381`.
- **DB/schema impact:** reads only (+ BLK-3 switch). · **Migration impact:** BLK-3 backfill (RR).
- **API impact:** `GET .../pickups` response sourced from canonical table (shape preserved — same fields exist in both tables; verify field parity in §8).
- **Event impact:** reconciliation triggered by consumer cadence (schedule = existing infra concern; no new transport).
- **Transaction impact:** none (read-side).
- **Concurrency impact:** none. · **Idempotency:** reconciliation idempotent (pure read + report).
- **Tests required:** §9 — legacy-write absence tests (GBA handlers never target legacy tables), A4 non-read test, secondary-column non-authority test, reconciliation detectors (seed each drift condition → flagged), canonical-read parity.
- **Dependencies:** WS-03/WS-04 (canonical + cascade live before reconciliation is meaningful), WS-01 (BLK-3).
- **Ordering:** boundaries early (tests first), reconciliation after WS-04, write-path switch last (cutover §11).
- **Rollback:** read-path flag reverts to legacy reads; legacy tables never dropped (Phase 11).
- **Verification criteria:** legacy-boundary suite green; reconciliation report runs clean on seeded data; typecheck + web lint.

---

## 5. Task-Level Plan

Every task: **ID · objective · files · prerequisites · action · expected result · tests · verification command · rollback.**
Gates: `[RR]` = needs Readiness Review confirmation first (mechanism, not business rule) · ⛔ = blocked by BLK-1/2/3.

### 5.1 WS-01 tasks

**T-01 — Baseline migration for new-world GBA tables** `[migrate]`
Files: `packages/db/migrations/<new>_gba_baseline/migration.sql`, `packages/db/schema.prisma` (verify only).
Prereq: none. Action: write `CREATE TABLE IF NOT EXISTS` for `group_bookings`, `group_blocks`, `group_block_daily_allocations`, `group_pickups`, `allotment_contracts`, `allotment_daily_quotas`, `allotment_vouchers`, `allotment_stop_sales` (+ analytics tables if absent from history) matching live DDL + declared uniques/indexes; `ALTER TABLE ... ADD COLUMN IF NOT EXISTS shoulder_days_before/after` (T-02 folded). Expected: fresh DB + `pnpm db:migrate` reproduces target schema (FIND-4 closed).
Tests: migration-from-scratch harness test (reuse `splitSqlStatements` from `availability-postgres.harness.ts`); drift test `prisma migrate diff` = empty. Verify: `pnpm db:generate && npx prisma validate`, run harness test. Rollback: down-script `DROP TABLE IF NOT EXISTS` (only where tables didn't pre-exist — confirm per env before writing down-script).

**T-02 — Declare shoulder columns in `schema.prisma`** `[migrate]`
Files: `packages/db/schema.prisma` (`group_blocks`). Prereq: none. Action: add `shoulder_days_before Int @default(0)`, `shoulder_days_after Int @default(0)` matching live DDL (FIND-1/TR-8.5). Expected: `grep shoulder_days schema.prisma` hits; drift check clean.
Tests: schema declaration test. Verify: `npx prisma validate`. Rollback: revert declaration (columns remain live).

**T-03 — ⛔ BLK-3 pickup canonicalization ALTER** `[migrate] [RR]`
Files: `schema.prisma` `group_pickups`, new migration, WS-03 write paths. Prereq: **RR approves ALTER + backfill stance**. Action: make `group_block_id` nullable; add nullable `allotment_id` (+ FK to `allotment_contracts`, hotel-scoped index `(hotel_id, allotment_id)`); decide backfill of live `allotment_pickups` (migrate rows vs start-fresh) — **RR data decision**. Expected: canonical table can hold both pickup flavors; legacy `allotment_pickups` becomes retireable.
Tests: both-flavor insert tests; parity read test (T-28). Verify: `npx prisma validate` + harness. Rollback: drop column/restored NOT NULL only after verifying no orphan allotment refs.

**T-04 — ⛔ BLK-1 block contract-type storage** `[migrate] [RR]`
Files: TBD by RR (`group_blocks` column vs alternative). Prereq: **RR resolves storage location**. Action: implement RR choice so `GUARANTEED_BLOCK` exclusion (TR-6.3) is queryable per block. Expected: wash-by-type can read block type. Tests: type persisted + read-back. Verify: harness. Rollback: per RR.

**T-05 — ⛔ BLK-2 wash/release/attrition persistence** `[migrate/create] [RR]`
Files: TBD by RR (new tables vs event-only vs audit log). Prereq: **RR resolves store within D-2 constraints**. Action: implement RR choice to satisfy TR-6.1/6.5/6.9 audit requirements. Expected: wash before/after + attrition record durably stored. Tests: record written in wash unit. Verify: harness. Rollback: per RR.

**T-06 — Schema gate test (declared = executed)** `[create]`
Files: new `packages/db/__tests__` or api harness test. Prereq: T-01/T-02. Action: assert target schema reachable from migrations alone; assert GBA columns (incl. shoulder) declared. Expected: TR-15.3/15.4 enforced by CI. Tests: this is the test. Verify: `pnpm test` in `packages/db` scope as configured (or api harness). Rollback: n/a (test only).

**T-07 — (Phase-2b scope, optional) INV-14 DB CHECK constraints** `[migrate, PHASE-2B]`
Files: migration. Prereq: RR confirms 2b scope. Action: `CHECK (picked >= 0 AND released >= 0 AND picked + released <= quota)` on quotas/allocations. Expected: defense-in-depth only (code remains the guard). Tests: constraint rejects violation insert. Verify: harness. Rollback: drop constraint.

### 5.2 WS-02 tasks

**T-08 — Block lifecycle transition enforcement** `[modify]`
Files: `domain/services/lifecycle.service.ts`, `group-block.aggregate.ts`, `group-allotment.module.ts`. Prereq: none. Action: implement §13.1 legal chain incl. first-class `DRAFT→TENTATIVE` (D-15/TR-2.1), invalid-transition rejection (INV-13), inventory-holding only `DEFINITE|OPEN_FOR_PICKUP` (TR-2.2), explicit-transition-only (no implicit widen). Expected: state machine matches §13.1 exactly; each transition emits mapped event in-unit (TR-2.5).
Tests: unit matrix (every legal/illegal pair), holding-status tests. Verify: `npx jest --testPathPattern "group-allotment|lifecycle"`. Rollback: code revert.

**T-09 — Block code generation (F-12)** `[modify]`
Files: `create-group-block.handler.ts`, `infrastructure/repositories/group-block.repository.ts`. Prereq: T-08. Action: derive `block_code` from actual per-hotel count/sequence (unique `(hotel_id, block_code)` `:17107`), not UUID-vs-count mismatch. Expected: no collision failures; unique holds. Tests: N-concurrent creates → all codes unique per hotel. Verify: unit suite. Rollback: code revert.

**T-10 — Allotment consumption gate + expiry zero-contribution** `[modify]`
Files: `allotment.aggregate.ts`, `create-allotment.handler.ts`, handlers consuming quota. Prereq: none. Action: consume rejected unless `ACTIVE` and not past `validity_end` (TR-3.3/13.1); confirm A1 validity filter keeps expired contributing zero (already `availability-source.adapter.ts:50` ✓ — test pins it, TR-13.2); expiry does **not** trigger wash (TR-13.4 — wash code absence assertion). Tests: unit + adapter. Verify: jest. Rollback: code revert.

**T-11 — Contract-type canonical mapping (S-3)** `[modify]`
Files: new VO helper in `domain/value-objects/`, `allotment.aggregate.ts`, `create-allotment`/`create-allotment` read paths, `availability-source.adapter.ts` (filter must follow mapping). Prereq: none. Action: implement decided mapping table (triad `ROLLING_RELEASE|GUARANTEED_BLOCK|FREE_SALE` + HARD/SOFT commitment; legacy folds: `HARD_COMMITMENT`→HARD, `GUARANTEED`→GUARANTEED_BLOCK, `is_rolling_release`→ROLLING_RELEASE, `FREE_SALE`→FREE_SALE, `SOFT_QUOTA`→SOFT); **⚠ storage of canonical value vs legacy `contract_type` values: verify read/write round-trip at implementation; if canonical persistence needs a column, raise as schema note to RR (do not invent)**. Expected: one published mapping, no scattered string literals.
Tests: mapping table unit tests (every legacy value → canonical). Verify: jest + adapter spec. Rollback: revert (values in DB unchanged).

**T-12 — Quota guard + guard ordering** `[modify]`
Files: `allotment.aggregate.ts` (`pickupQuota`, `restoreQuota`), intake handlers. Prereq: none. Action: enforce `picked + released ≤ quota` after every commit (TR-3.2/11.3); enforce order eligibility → stop-sale → remaining (TR-7.4/4.5) in every allotment intake path. Expected: insufficient-quota deterministic `QUOTA_INSUFFICIENT`; stop-sale blocks before quantity check.
Tests: guard unit tests, ordering tests (stop-sale active + quota available → blocked; error type proves stop-sale ran first). Verify: jest. Rollback: code revert.

**T-13 — quota ≤ physical guard + flagged violations** `[modify]`
Files: `create-allotment.handler.ts`, `set-allotment-quota.handler.ts`; reconciliation detector (T-66). Prereq: none. Action: at contract create and set-quota, compare daily quota vs physical inventory (rooms count via A1 `physicalCount` source or direct `rooms` read — same definition as A1 line 55-56 exclusions); reject `quota > physical` (TR-1.5/3.5); existing violating rows **surfaced only** (TR-10.4) through T-66, never rewritten. Expected: new violations impossible; legacy violations visible.
Tests: create/set-quota rejection; historical-violation flagged-not-changed test. Verify: jest + harness. Rollback: code revert.

**T-14 — Stop-sale flag synchronization (F-10)** `[modify]`
Files: `apply-allotment-stop-sale.handler.ts`, `lift-allotment-stop-sale.handler.ts`, `allotment.aggregate.ts`. Prereq: none. Action: apply/lift updates `allotment_daily_quotas.stop_sale_active` for the date×category set **in the same transaction** as the `allotment_stop_sales` row; lift restores the flag from other applied stop-sales (overlap-aware). Expected: A1 conflict detection (`availability-source.adapter.ts:84-87`) stays silent.
Tests: apply/lift/overlap tests; adapter conflict test. Verify: jest + `availability-source.adapter.spec.ts`. Rollback: code revert.

**T-15 — Voucher legal-transition enforcement (INV-13)** `[modify]`
Files: `allotment-voucher.entity.ts`, `cancel-allotment-voucher.handler.ts`, `consume-allotment-voucher.handler.ts`, `create-allotment-voucher.handler.ts`. Prereq: none. Action: enforce §6.3 matrix: `ISSUED→USED|CANCELLED|EXPIRED`, `USED→CANCELLED` only via decided S-5 path, all other moves deterministic rejection; no re-issue from terminal states. (`EXPIRED` transition mechanism — read-derived vs sweep — **[RR] note §13; enforcement semantics unchanged**.) Expected: matrix-enforced states.
Tests: full transition matrix unit tests. Verify: jest. Rollback: code revert.

**T-16 — Aggregate-only mutation sweep (C-5/C-9/C-10/F-11)** `[modify]`
Files: `set-daily-allocation.handler.ts`, `set-allotment-quota.handler.ts`, `add/remove-room-category.handler.ts`, `add/remove-shoulder-days.handler.ts`, repositories. Prereq: T-08/T-12. Action: move raw counter SQL + multi-row double-writes into aggregate saves so quota rows + contract/block rollups commit as one unit with recomputation (TR-2.4/11.3); no raw counter mutation outside repositories loading aggregates (TR-2.5). Expected: single write path; events always emitted.
Tests: rollup-consistency tests; grep gate `INSERT INTO group_block_daily_allocations`/`UPDATE group_blocks SET.*qty` outside allowed files. Verify: jest + grep gate script in test. Rollback: code revert.

**T-17 — Delete-path verification (soft-delete consistency)** `[modify-verify]`
Files: `delete-group-booking.handler.ts`, `delete-group-block.handler.ts`, allotment delete (`allotment.controller.ts:321 @Delete`). Prereq: none. Action: verify deletes use soft-delete (`deleted_at`) consistent with A1 filters (`deleted_at: null` `availability-source.adapter.ts:49-50`); if any hard-deletes, align to soft-delete (declared columns exist) — **behavior pinning, not new policy** (A1 already assumes soft-deleted rows remain). Expected: no orphan counter rows.
Tests: delete → A1 contribution stops; row retained. Verify: jest + adapter spec. Rollback: code revert.

**T-18 — Pickup/allocation eligibility preconditions (TR-2.3)** `[modify]`
Files: `create-group-pickup.handler.ts`, `set-daily-allocation.handler.ts`, `release-*` handlers. Prereq: T-08. Action: explicit state preconditions on block (`DEFINITE|OPEN_FOR_PICKUP` for pickup; allocation edits blocked per §4.3 removal guard) with deterministic errors. Expected: ineligible-state operations rejected before any write. Tests: precondition matrix. Verify: jest. Rollback: code revert.

### 5.3 WS-03 tasks

**T-20 — create-group-pickup atomic rewrite** `[modify]`
Files: `create-group-pickup.handler.ts`, `prisma-reservation-association.adapter.ts`, `group-allotment.module.ts`. Prereq: T-56 (unit seam), T-08. Action: wrap reservation creation + `group_pickups` insert + allocation counter update + events in ONE unit-of-work transaction; hotel-scoped everywhere; remove partial-commit ordering. Expected: §12.2 row 1 holds; failure → zero partial rows.
Tests: atomicity fault-injection (fail at each step), linkage test (real `reservation_id`), duplicate-submit idempotency, hotel-isolation negative. Verify: jest + new postgres spec (`AVAILABILITY_TEST_DATABASE_URL` env — compose `xylo-postgres` verified up). Rollback: handler revert (flag-protected deploy).

**T-21 — create-allotment-pickup atomic rewrite + canonical write** `[modify] ⛔(T-03)`
Files: `create-allotment-pickup.handler.ts` (currently quota→port→raw legacy insert `:129`, swallowed `rollbackQuota :165-185`). Prereq: T-56, T-12, T-03 (canonical target), T-11. Action: single transaction (quota hold + real reservation + **canonical** pickup row + events); delete `rollbackQuota` (D-9); idempotency read moves to canonical table (`:46-63` legacy read replaced). Expected: no fabricated/legacy writes; no swallowed compensation.
Tests: atomicity, duplicate voucher-code pickup idempotency, real-linkage, stop-sale ordering (T-12), isolation. Verify: jest + postgres. Rollback: flag back to legacy-write path (cutover §11) until BLK-3 resolved.

**T-22 — Voucher consume: real reservation, no fabrication (D-10)** `[modify]`
Files: `consume-allotment-voucher.handler.ts` (`:26` `RES-${Date.now()}`), `allotment.aggregate.ts` `consumeVoucher`, `create-allotment-voucher.handler.ts`. Prereq: T-15, T-56. Action: consume creates (or links existing) **real** reservation for this hotel via the association port inside the same unit; `reservationId` real-or-null only (TR-3.4/4.4/9.3); second consume of `USED` → deterministic rejection/idempotent return of the same reservation (INV-3, spec §32); voucher state moves `ISSUED→USED` in-unit.
Tests: consume creates real reservation (assert FK row exists in `reservations` with hotel match); duplicate consume; null-safe linkage; atomicity. Verify: jest + postgres. Rollback: code revert.

**T-23 — Voucher cancellation per S-5/TR-12.5** `[modify]`
Files: `cancel-allotment-voucher.handler.ts`, `allotment-voucher.entity.ts`. Prereq: T-15, T-22. Action: `ISSUED` → restore `picked--` (floor-bounded) in-unit + event; `USED` → if linked reservation live → **delegate to reservation-cancel semantics** (spec `:725`) rather than writing quota; if reservation already cancelled → no second restore (TR-11.4); never restore for live reservation (INV-5). Expected: S-5 both branches.
Tests: §9 mandatory — USED-cancel w/ live reservation (no restore), USED-cancel w/ cancelled reservation (restore once), ISSUED-cancel restore, duplicate-cancel no-op. Verify: jest + postgres. Rollback: code revert.

**T-24 — Cancel-pickup restore exactly-once + hotel scoping** `[modify]`
Files: `cancel-allotment-pickup.handler.ts` (`:66` unscoped reservation UPDATE; `:90` legacy table), `create-group-pickup`-family cancel paths, `group-pickup.entity.ts`. Prereq: T-03, T-32 (cascade exists to interlock with). Action: pickup-cancel unit: status precondition → restore counters once → unlink → event; reservation-cancel side effect **hotel-scoped** (`WHERE id=$1 AND hotel_id=$2`) (FIND-5/TR-14.1); write canonical row not legacy `allotment_pickups`. Expected: TR-12.3 + no cross-tenant write + no double restore with cascade (T-36).
Tests: restore-once, isolation negative (foreign hotel id → no-op), CHECKED_IN/CHECKED_OUT no-restore precondition (existing `:59-60` behavior pinned), canonical row updated. Verify: jest + postgres. Rollback: code revert.

**T-25 — Guest identity resolution order (D-11 = C amended)** `[modify]`
Files: `prisma-reservation-association.adapter.ts` (guest lookups `:82`, `:91`, `:245`, `:254`). Prereq: none. Action: implement exact hotel-scoped order: explicit Guest ID → Passport ID → Email → create new; prohibit name `ILIKE`/fuzzy/`LIMIT 1` binding; conflicting match (passport vs email → different guests) → `GUEST_MATCH_CONFLICT` error requiring explicit caller handling (TR-4.10/9.6). Caller-contract fallout (adapters/controllers) fixed in same task.
Tests: order tests (each resolution branch), conflict test, isolation tests (foreign-hotel guest not matched), no-fuzzy-match negative. Verify: jest. Rollback: code revert.

**T-26 — Two-layer pickup validation consult (TR-4.1/10.6)** `[create] [RR: port shape]`
Files: intake handlers (T-20/21/22), new port replacing dead `inventory-commitment.port.ts`, `availability-snapshot.service.ts` (consult target). Prereq: T-56; WS-05 authority read available. Action: layer (a) pool check across every stay date×room-type + layer (b) route the created reservation through the availability assertion consult per agreed seam; **port shape/assert-vs-read mechanism = [DEFERRED] per D-7 — RR confirms alternatives (§13)**; wire against existing assertion service (`availability-assertion.service.ts`) for interim read-time consult if RR confirms. Expected: pickup cannot overcommit availability (TR-1.4).
Tests: pickup rejected when authority denies; accepted case passes; mechanism swap test (seam mockable). Verify: jest + assertion specs. Rollback: seam flag off (behavior reverts to quota-only guards — documented risk in §12).

**T-27 — Anomalous linkage surfacing (D-10 note 4)** `[create]`
Files: voucher/pickup query handlers (`get-allotments.handler.ts`, `GET :id/pickups`). Prereq: T-22. Action: reads flag `USED` voucher with null/missing reservation as anomaly (provenance field) — **no repair logic**. Expected: visible, not guessed. Tests: seeded anomaly → flagged. Verify: jest. Rollback: revert.

**T-28 — Canonical pickup reads** `[modify] ⛔(T-03)`
Files: `allotment.controller.ts:281`, `group-booking` pickup routes, `useAllotmentPickups` (`use-group-allotment.ts:381`), views consuming pickups. Prereq: T-03, T-21. Action: serve pickup lists from `group_pickups` (field parity with legacy verified — both carry guest/room/dates/status/reservation_id); flag-switched (§11). Expected: single ledger visible end-to-end.
Tests: read parity test (legacy fixture vs canonical row → same API shape). Verify: jest + web typecheck. Rollback: read flag to legacy.

**T-29 — Intake idempotency keys** `[create]`
Files: intake handlers (T-20/21/22/23/24), optionally `core/interceptors/idempotency.interceptor.ts` wiring. Prereq: T-56. Action: business idempotency: voucher code unique (DB ✓ `uq_allotment_voucher_hotel_allot_code`), pickup status-preconditions, optional `Idempotency-Key` request header via existing interceptor where already wired — **request-level dedup mechanics [RR] §13**. Expected: duplicate requests converge (INV-3/6).
Tests: double-submit tests per path. Verify: jest. Rollback: revert.

### 5.4 WS-04 tasks

**T-30 — `reservation.checked_out` event producer** `[create]` (FIND-2, spec §16.2)
Files: `reservations/domain/events/reservation-domain.events.ts` (or `packages/shared/src/events/index.ts` — verify export home at implementation), `front-office/.../check-out/check-out.handler.ts`. Prereq: none. Action: new IntegrationEvent `reservation.checked_out` carrying `{reservationId, hotelId}` emitted at the status-transition point inside the existing `client` transaction (pattern at `:254`). Expected: event exists, outbox-written in-tx.
Tests: emission-once-per-checkout, in-tx atomicity (checkout rolls back → no event). Verify: jest + outbox test. Rollback: remove emission (consumer tolerates absence).

**T-31 — Remove FO raw GBA writes (L-13)** `[modify] ⛔(after T-32/T-33 active)`
Files: `check-out.handler.ts` — delete `:390-397` (`UPDATE allotment_pickups`) and `:479-485` (`UPDATE group_pickups`); **retain** folio auto-post `:322-383` (financial owner) and events. Prereq: T-30, T-32, T-33 consumers live (activation order §13). Expected: FO performs zero GBA writes (§3.1); pickup status driven by events only.
Tests: grep gate `UPDATE group_pickups|UPDATE allotment_pickups` empty under `modules/front-office`; checkout end-to-end → pickup `CHECKED_OUT` via consumer. Verify: jest + grep. Rollback: re-add raw SQL only under emergency flag (documented; not default).

**T-32 — Reservation-cancelled → GBA cascade consumer** `[create]`
Files: `shared/events.consumer.ts` (new handler), `group-allotment` repository for pickup lookup. Prereq: T-56. Action: on `reservation.cancelled`: find pickup(s) by `(hotel_id, reservation_id)`; if `ACTIVE` → set `CANCELLED`, unlink (`reservation_id` cleared per TR-9.2 unlink semantics), restore counters **once** (precondition: already-`CANCELLED` → no-op), invalidate availability cache (TR-12.6) — all inside consumer unit; idempotent via `EventIdempotencyService` (`:40-43` pattern). Expected: TR-12.1/12.3/12.6 satisfied event-driven (FO already emits `reservation.cancelled` in-tx — `reservation.repository.ts:791` ✓).
Tests: cascade restore-once; duplicate event → no-op; pickup-already-cancelled → no-op; foreign hotel event → no-op; out-of-order tolerance. Verify: extended `modules/shared/__tests__/events.consumer.spec.ts` + postgres. Rollback: consumer flag off (⚠ during off-state, cancellation does not restore — flagged as monitored window §11).

**T-33 — Checkout → pickup `CHECKED_OUT` consumer** `[create]`
Files: `shared/events.consumer.ts`. Prereq: T-30. Action: on `reservation.checked_out`: pickup `ACTIVE→CHECKED_OUT`, **no counter change** (TR-4.8), no availability math change; idempotent (already `CHECKED_OUT` → no-op). Expected: §7.2 row 2.
Tests: status flip + counter-invariance + duplicate delivery. Verify: consumer spec. Rollback: flag off (paired with T-31 ordering).

**T-34 — Check-in / no-show invariance verification** `[create-verify]`
Files: `front-office` check-in command (existing), `reservations` no-show path. Prereq: T-32/33. Action: **no code change expected** — add tests proving check-in (incl. voucher redemption) and no-show do not alter pickup counters or voucher quota (§7.3: no-restore for no-show). If a counter write is discovered → **stop, document as blocker** (would contradict contract silence only if it invents behavior; existing counter write on no-show would contradict §7.3 → BLK).
Tests: invariance tests. Verify: jest. Rollback: n/a.

**T-35 — Modify/extend counter-preservation guard** `[create-verify]`
Files: reservations modify/extend handlers (existing paths). Prereq: T-32. Action: **no new behavior** (SOURCE-SILENT §7.3) — regression tests asserting pickup counters/records are untouched by modify/room-type-change/extend flows; non-goal note in PR checklist. Expected: contract silence preserved (no rule invented).
Tests: counter-preservation across modify/extend. Verify: jest. Rollback: n/a.

**T-36 — Cancel-pickup ↔ cascade double-restore interlock test** `[create-verify]` (TR-11.4)
Files: tests over `cancel-allotment-pickup.handler.ts` + consumer. Prereq: T-24, T-32. Action: sequence test: cancel-pickup command (which cancels linked reservation per existing behavior) → emitted `reservation.cancelled` → consumer no-op (pickup already `CANCELLED`) → counters restored exactly once. Expected: INV-6 proven.
Tests: this test + variant (event arrives before command completes → command precondition path). Verify: jest + postgres. Rollback: n/a.

### 5.5 WS-05 tasks

**T-37 — A1 stop-sale display wiring (TR-7.2/7.3)** `[modify]`
Files: `availability/infrastructure/adapters/availability-source.adapter.ts` (`:113` constant, `:97-102` status logic), `snapshot-calculator.ts` (restriction outcome), `availability-source.adapter.spec.ts`. Prereq: T-14 (flag sync makes facts consistent). Action: compute per-quota stop-sale display fact; selling display remaining reads 0 when applied (TR-7.2), lift restores display to actual (TR-7.3); allotment scope only (TR-7.1/S-4); keep `UNRESOLVED` promotion guard. Expected: display semantics without quantity change.
Tests: applied→display 0 (quota counters unchanged), lifted→display actual, block pickups unaffected, conflict rows stay UNRESOLVED. Verify: `npx jest --testPathPattern availability-source`. Rollback: revert (constant returns).

**T-38 — A3 rebuild on Availability (D-6a = A)** `[modify]`
Files: `activities/availability-sales.controller.ts` (`@Get('availability/matrix') :237`, computation `:430`, `bulk-update :549`, `interval-update :671` — writes stay A3's own restrictions; **availability numbers** come from authority), `apps/web/components/AvailabilitySales.tsx`. Prereq: WS-05 authority read stable; T-37. Action: matrix values sourced from availability snapshot service (same inputs as A1); if phased, interim response labeled `"derived view, not sellable availability"` (decision-sanctioned option B labeling); response **shape preserved** for frontend. Expected: no second availability computation.
Tests: parity test A3 value == A1 value for same inputs; label assertion when interim. Verify: jest + web typecheck/lint. Rollback: flag to legacy computation (labeled) — §11.

**T-39 — Retire F-18 `available-rooms` (D-6/F-18)** `[remove]`
Files: `group-booking.controller.ts:95-116`, `group-allotment.api.ts:29`, `use-group-allotment.ts:213`, calling views under `features/group-allotment/views/` (switch to availability API read). Prereq: T-38 (frontend has authoritative read), T-42. Action: delete route + hook + api fn; update callers; 404 acceptable (contract requires retirement, not compat alias). Expected: no competing available-number source.
Tests: route-absence test; grep `available-rooms` empty; caller still renders (component test). Verify: jest + `pnpm lint`/typecheck (web). Rollback: re-deploy previous artifact (route restore is code revert).

**T-40 — GBA event → availability invalidation consumers** `[create]`
Files: `shared/events.consumer.ts`. Prereq: none (can parallel WS-04). Action: register cases for all 22 GBA event types → `cache.delPattern('availability:${hotelId}:*')` (existing `invalidateAvailability :93-100`) + snapshot/assertion invalidation hooks as used by authority; consume `group_block.released|washed|cancelled|closed|allocation_changed`, `allotment.quota_changed|released|stop_sale_*|voucher_*`, `group_pickup.*`. Expected: TR-10.3 stale-by-design eliminated for GBA mutations.
Tests: each event → invalidation invoked; idempotent duplicate; unknown-event warn gate removed for Phase-4 types (T-63). Verify: consumer spec. Rollback: flag off (risk: stale cache — monitored §11).

**T-41 — Pickup → assertion wiring seam (TR-4.1) [RR]** `[create] [DEFERRED shape]`
Files: port (replace `inventory-commitment.port.ts`), pickup handlers (T-26 overlap — **this task records the RR decision record + final wiring**). Prereq: RR confirms assert-at-write vs read-time+assertion-consumption alternatives (§13). Action: implement confirmed option. Expected: pickup-created reservations pass the assertion lifecycle (A5 gap closed).
Tests: asserted-flow integration tests per confirmed option. Verify: jest + assertion postgres specs. Rollback: seam flag.

**T-42 — Explicit overbooking guard tests (TR-10.5/S-2)** `[create-verify]`
Files: tests over A1 adapter + quota guards. Prereq: T-13. Action: assert the only overbooking input is `overbooking_limits` (`availability-source.adapter.ts:52/:116`); assert no code path creates contract overcommit; assert `quota > physical` surfaces UNRESOLVED flag not clamp. Tests as above. Verify: jest. Rollback: n/a.

### 5.6 WS-06 tasks

**T-43 — Manual release guard + in-unit events (TR-5.1–5.5)** `[modify]`
Files: `release-block-allocation.handler.ts`, `release-allotment-allocation.handler.ts`, `group-block.aggregate.ts` release methods, `allotment.aggregate.ts`, `daily-room-allocation.value-object.ts`. Prereq: T-08/T-12. Action: clamp release to `held − picked` (floor 0); reject against `DRAFT/CLOSED/CANCELLED` blocks and non-`ACTIVE` allotments (TR-5.3); counter + `group_block.released`/`allotment.released` event one unit (TR-5.2/11.6); hotel-scoped. Expected: releasing picked rooms impossible without reservation cancel (TR-5.1).
Tests: bounds matrix, ineligible-state rejection, invalidation event emitted, isolation. Verify: jest + postgres. Rollback: code revert.

**T-44 — Release verb frontend fix (D-12, FIND-3)** `[modify]`
Files: `apps/web/features/group-allotment/api/group-allotment.api.ts:146` (`api.put` → `api.post`), callers (`useReleaseBlockAllocation :200`, `GroupBookingDetailView.tsx`, `AllotmentDetailView.tsx:663-720`). Prereq: none. Action: align caller to decided `POST` (controller `group-booking.controller.ts:247` already `@Post`). Expected: block release works from UI; **no PUT alias added** (decision).
Tests: api fn test asserting POST; web typecheck. Verify: `cd apps/web && pnpm typecheck && pnpm lint && pnpm test`. Rollback: revert one line.

**T-45 — ExecuteCutOffWash command (block)** `[create] ⛔(partial: T-04 type exclusion, T-05 record)`
Files: new `application/commands/execute-cut-off-wash/` (+ `.command/.handler`), `group-block.aggregate.ts` (wash-all-unpicked across extended range), `group-allotment.module.ts` registration, `permissions/group-allotment.permissions.ts` (existing `GROUP_BLOCK_WASH :14` for any manual invocation surface). Prereq: T-56, T-52 (attrition call), T-04/T-05 for full conformance (**date mechanics + counters + event ship without them; exclusion/log lines gated**). Action: load blocks with `cutoff_date <= today` not yet washed → per block one unit: compute unpicked per date (core+shoulder, no exclusion filter TR-8.6b) → `released += unpicked` (picked untouched) → wash record ⛔ → attrition (T-53) → `group_block.washed` before/after event → idempotent key `(block_id, cutoff_date)` re-run no-op → date-granular from start of day (TR-6.4). Expected: §8.2 semantics; A1 `gbaRemaining` reads 0 for washed range.
Tests: single-wash-event, repeated-wash no-op, shoulder-included, picked-preserved, GUARANTEED_BLOCK exclusion (**gated T-04**), atomicity (attrition failure → no counter change), event payload before/after. Verify: jest + postgres wash suite. Rollback: command dormant until scheduler flag (§11).

**T-46 — ExecuteRollingRelease (allotment) + `AllotmentWashExecuted`** `[create]`
Files: new `application/commands/execute-rolling-release/`, `allotment.aggregate.ts`, `group-allotment.events.ts` (new event class), `group-allotment.module.ts`. Prereq: T-56, T-11 (type mapping), T-05 for record ⛔. Action: contracts with canonical type `ROLLING_RELEASE` → release unpicked quota for dates in `today..today + release_days_before` window (spec `:600-609` carried); `GUARANTEED_BLOCK` excluded (TR-6.3 — contract-level, uses mapping not BLK-1); `FREE_SALE` skipped (TR-6.3 family); idempotent per `(allotment_id, windowStart, windowEnd)`; emit `AllotmentWashExecuted` + invalidation. Expected: T-2 semantics.
Tests: window math, type exclusions, idempotency, counter bounds, event, isolation. Verify: jest + postgres. Rollback: dormant until scheduler flag.

**T-47 — `WashSchedulerPort` interface (TR-6.6 [DEFERRED] transport)** `[create] [RR]`
Files: new `domain/ports/wash-scheduler.port.ts` + invocation entrypoint `runDueWashes(now, hotelId)` calling T-45/T-46; **no transport implementation selected here**. Prereq: T-45/T-46 exist. Action: define port + report shape; document RR alternatives in §13 (BullMQ repeatable jobs — `infrastructure/bullmq/` exists; `@nestjs/schedule`; Temporal — compose service exists); RR selects transport. Expected: wash runnable in tests deterministically (`runDueWashes(fixedDate)`) without a chosen production transport.
Tests: `runDueWashes` integration test with fixed clock (drives T-45/46). Verify: jest. Rollback: n/a (interface only).

**T-48 — TR-6.5 data-honesty gating** `[modify]`
Files: read handlers exposing `cutoff_date`/`release_days_before` metadata (booking/block detail queries), frontend labels (`GroupBookingDetailView.tsx`, `AllotmentDetailView.tsx`), rollout flag (§11). Prereq: none. Action: while scheduler inactive → label wash metadata "scheduled — not yet authoritative"; when active → authoritative. Expected: never stored-but-ignored without disclosure (TR-6.5).
Tests: flag-state label tests. Verify: jest + web tests. Rollback: flag toggle.

### 5.7 WS-07 tasks

**T-49 — add-shoulder rewrite: declared quantities + aggregate path (TR-8.1/8.2/8.4)** `[modify]`
Files: `add-shoulder-days.handler.ts` (raw zero-insert `:96-106`, raw block update `:113-125`), `group-block.aggregate.ts`, `group-booking.controller.ts:257` request validation, `group-allotment.api.ts`/`useAddShoulderDays :101`. Prereq: T-01/T-02 (columns declared), T-16 (aggregate path). Action: request must carry explicit per-room-type shoulder quantities (reject silent 0 placeholders — TR-8.2); allocations created through aggregate with core-day guards (TR-8.1); date-range extension + shoulder counters updated in-unit; emit mapped `group_block.allocation_changed`/`ShoulderAllocationChanged` (TR-8.4) with invalidation. Expected: no raw SQL; declared quantities only.
Tests: qty-0 rejection, guard parity with core days, event emission, rollup recompute, isolation, idempotent re-add. Verify: jest. Rollback: code revert.

**T-50 — remove-shoulder guard + rollup (TR-8.3, F-11)** `[modify]`
Files: `remove-shoulder-days.handler.ts`, `group-block.aggregate.ts`. Prereq: T-49. Action: removal rejected while active pickups exist on those dates (TR-8.3); allowed removal recomputes block rollups; no unguarded deletes (kill raw path). Expected: §9 rule 3.
Tests: pickup-present → rejected; pickup-free → removed + rollups recomputed; event emitted. Verify: jest. Rollback: code revert.

**T-51 — Shoulder uniformity (P1/P2/P3) cross-suite** `[create-verify]` (D-13=A)
Files: tests spanning `attrition-calculation.service.ts` (P1 base includes shoulder), T-45 wash (P2 covers extended range), pickup handlers (P3 unrestricted). Prereq: T-45, T-52, T-49/50. Action: **no new code** — conformance tests proving the three differentiators + "same guards as core". Expected: D-13 fully pinned.
Tests: as listed. Verify: jest. Rollback: n/a.

### 5.8 WS-08 tasks

**T-52 — Attrition calculation conformance (D-14 = A)** `[modify]`
Files: `domain/services/attrition-calculation.service.ts`, `domain/value-objects/attrition-policy.value-object.ts`, `create-group-booking.handler.ts` (threshold validation), `create-group-booking.command.ts:25`. Prereq: none. Action: single-percentage policy default 80 uniform (TR-6.8); math `ceil(contracted×threshold/100)`, `shortfall = max(0, minimumRequired − picked)`, `meetsThreshold`; bounds 50–100 `[SPEC-CARRIED]`; **remove/normalize rejected VO variants** (85 default, per_day_minimum, cumulative, room_type_specific); base excludes nothing (shoulder included — T-51). Expected: read-only calculation, no inventory mutation (TR-6.7).
Tests: default-80, custom threshold, ceil boundary, shortfall 0 at exact threshold, shoulder-in-base, rejected-variant absence, no-inventory-write proof. Verify: jest. Rollback: code revert.

**T-53 — Wash-time assessment + `ATTRITION_FEE` posting in wash unit (TR-6.7/6.9)** `[create] ⛔(record: T-05)`
Files: T-45/T-46 handlers (call site), `prisma-folio.port.ts` (existing posting path `:36`), `group-allotment.module.ts`, trx-code registry (verify `ATTRITION_FEE` exists — `rates-inventory`/codes table read; **verification step**, create only if the code table legitimately lacks the decided code — flag in PR). Prereq: T-52, T-45, T-56. Action: inside wash transaction when `shortfall > 0`: `liabilityDue = shortfall × negotiatedRate` posted to block master folio as `ATTRITION_FEE`; single commit with counters/log/assessment (never async); wash idempotency ⇒ assessed once (INV-11); **ledger module ownership = [RR] dependency §13** (posting uses existing folio structures as-is). Expected: §10 contract rows.
Tests: posting inside wash tx (failure injection: posting fails → no wash), single-assessment on repeat, no-post when shortfall 0, amount math, master folio balance integration. Verify: jest + postgres. Rollback: dormant with wash flag.

**T-54 — `AttritionAssessed` event + audit trail** `[create] ⛔(persistence: T-05)`
Files: `group-allotment.events.ts` (new class), wash handlers, `platform/audit/audit-subscriber.service.ts` (subscribe — pattern `:123`). Prereq: T-53. Action: emit in wash unit `{hotelId, blockId, contracted, picked, threshold, shortfall, liabilityDue}`; subscribe to audit log (interim durable trail while BLK-2 open). Expected: observable + auditable assessment.
Tests: event emitted in-tx; audit subscriber invoked. Verify: jest. Rollback: revert emission.

**T-55 — ⛔ BLK-2 attrition/wash record persistence** `[create] [RR]`
Prereq: T-05 (RR store decision). Action: implement chosen store writes inside wash unit. Tests: record row exists post-wash with before/after. Verify: harness. Rollback: per RR.

### 5.9 WS-09 tasks

**T-56 — `CounterUnitOfWork` seam + mechanism alternatives (TR-11.7 [DEFERRED])** `[create] [RR]`
Files: shared `common/database/unit-of-work` (reuse) + thin GBA seam (e.g. `group-allotment/infrastructure/transaction/counter-unit.ts`) exposing `commit({expectedVersions, rowMutations, aggregateSaves, events}, tx?)`. Prereq: none. Action: build seam with **pluggable concurrency strategy**; document RR alternatives (§13): (a) optimistic conditional UPDATE on `version` + bounded retry, (b) `SELECT ... FOR UPDATE` row locks in fixed order, (c) advisory locks per hotel+key — **RR picks**; isolation level decision also RR (current pattern ReadCommitted — harness `:13-16`). Events always via `eventBus.publish(evt, tx)`/outbox-in-tx. Expected: one vehicle for every §12.2 unit; mechanism swappable without touching handlers.
Tests: seam unit tests (version mismatch → `CONFLICT`), retry convergence, in-tx outbox write. Verify: jest. Rollback: seam unused → nothing changes.

**T-57 — Handler adoption batch A (pickup/voucher/cancel)** `[modify]`
Files: T-20–T-24 handlers (adoption embedded in their rewrites). Prereq: T-56. Action: all intake/cancel paths run through the seam; partial-failure tests attached. Expected: §12.2 rows 1–7 satisfied. Tests: fault-injection atomicity per path. Verify: jest + postgres. Rollback: per-handler.

**T-58 — Handler adoption batch B (wash/release/shoulder/quota/status)** `[modify]`
Files: T-43/T-45/T-46/T-49/T-16/T-08 handlers. Prereq: T-56. Action: same adoption. Expected: §12.2 rows 8–13 satisfied. Tests: atomicity per path; concurrent wash vs pickup vs release (§9). Verify: jest + postgres. Rollback: per-handler.

**T-59 — Hotel-isolation raw-SQL sweep (TR-14.1, FIND-5)** `[modify]`
Files: **all** `$queryRaw`/`$executeRawUnsafe` under `apps/api/src/modules/group-allotment/` — known hits: `cancel-allotment-pickup.handler.ts:29/:53/:66/:90`, `create-allotment-pickup.handler.ts:46/:86`, `add-shoulder-days.handler.ts:50/:65/:96/:113`, plus repository/adapters (`prisma-reservation-association.adapter.ts`, `prisma-folio.port.ts`, `prisma-billing-instruction.adapter.ts`). Prereq: none. Action: add `hotel_id = $n` predicate to every statement touching tenant data (read **and** write); exception: statements already scoped by FK-of-scoped-parent must still carry explicit predicate; add repo test gate enumerating raw statements (whitelist + predicate assertion). Expected: zero unscoped statements (closes C-7 family).
Tests: isolation matrix tests (foreign-hotel ids → no rows/no writes); static gate test. Verify: jest + grep. Rollback: code revert.

**T-60 — Partial-failure purge + fault-injection suite (D-9)** `[modify]`
Files: `create-allotment-pickup.handler.ts` (`rollbackQuota :165-185` delete), any remaining swallowed catches in intake paths. Prereq: T-57. Action: failures propagate → unit rollback → deterministic error; no compensating-swallow left (audit F-6/C-1…C-6). Expected: TR-11.6 honored.
Tests: fault injection at each step of each §12.2 unit → assert zero partial rows + error surfaced. Verify: jest + postgres. Rollback: code revert.

**T-61 — Error semantics mapping (`CONFLICT` etc.)** `[modify]`
Files: GBA handlers' `Result.failure` codes, `apps/api/src/common/exceptions/exception.filter.ts` (existing mapping), controllers. Prereq: T-56. Action: stale-version/lost-update → `409 CONFLICT` (TR-11.2); deterministic codes for `QUOTA_INSUFFICIENT`, `ALLOTMENT_NOT_ACTIVE`, `VOUCHER_ALREADY_USED`, `GUEST_MATCH_CONFLICT`, `BLOCK_INELIGIBLE_STATE`, `SHOULDER_PICKUP_PRESENT`, `QUOTA_EXCEEDS_PHYSICAL`. Expected: no 200-wrapped failures for conflict class; INV-13 visible to clients.
Tests: contract tests asserting status codes + error codes. Verify: jest. Rollback: code revert.

### 5.10 WS-10 tasks

**T-62 — Event contract mapping + missing event classes** `[create/modify]`
Files: `group-allotment.events.ts`, reservation events, this plan §10.1 mapping table (published in-repo as module-level doc comment). Prereq: none. Action: add `AllotmentWashExecuted`, `AttritionAssessed` (T-54), `ShoulderAllocationChanged` **or** formally alias to `group_block.allocation_changed` (alias = naming mechanics chosen at implementation, documented), `reservation.checked_out` (T-30); map every spec §16.1/16.2 name to its implemented type; ensure all carry `hotelId`, aggregate ids, before/after where §16.1 requires. Expected: contract §16 satisfiable end-to-end.
Tests: static test asserting every spec name has a mapped implemented type; payload-shape tests (hotelId present). Verify: jest. Rollback: additive classes removable.

**T-63 — Consumer routing matrix for all GBA events (INV-19)** `[modify]`
Files: `shared/events.consumer.ts`, `modules/shared/__tests__/events.consumer.spec.ts`. Prereq: T-40, T-32, T-33. Action: explicit case for every Phase-4 event type (no `default: warn` for them); unknown non-Phase-4 events keep warn. Expected: every published event has ≥1 consumer (contract §16, INV-19).
Tests: static coverage test (event-type list from `group-allotment.events.ts` ⊆ consumer cases); routing matrix test. Verify: jest. Rollback: code revert.

**T-64 — Duplicate/out-of-order/retry delivery suite** `[create-verify]`
Files: consumer tests. Prereq: T-63. Action: tests for at-least-once duplicates, out-of-order pairs (cancel+checkout, wash+pickup), consumer failure → queue retry (existing rethrow `:61-63`), replay after outbox redelivery. Expected: order-tolerance + idempotency proven (§7.2, §16.2).
Tests: as listed. Verify: jest. Rollback: n/a.

### 5.11 WS-11 tasks

**T-65 — Legacy-write absence regression tests** `[create-verify]` (TR-15.1)
Files: new tests over GBA handlers; static gate enumerating raw SQL targets. Prereq: T-59. Action: assert GBA code never `INSERT/UPDATE/DELETE`s `allotment`, `allotment_pickup`, `allotment_room_types`, `reservation_groups`, `reservation_block`, `availability` (A4); exception until cutover: `allotment_pickups` writes allowed only behind canonical-switch flag (BLK-3). Expected: legacy freeze enforced by CI.
Tests: static gate + spot runtime tests. Verify: jest. Rollback: n/a.

**T-66 — GBA reconciliation detectors (read-side)** `[create]`
Files: `availability/infrastructure/reconciliation/availability-reconciliation.service.ts` (extend) or new `group-allotment/infrastructure/reconciliation/gba-reconciliation.service.ts`; wire to existing reconciliation cadence. Prereq: T-32 (data flows). Action: detectors (a) `picked` ⇔ ACTIVE+CHECKED_OUT pickups + USED vouchers, (b) `picked+released ≤ quota` violations, (c) `quota > physical` rows (flag only — TR-10.4/S-2), (d) `USED` voucher missing reservation (anomaly), (e) legacy-vs-canonical drift (cutover readiness). Output: structured report + alerts — **no auto-repair ever** (§19). Expected: drift visible (F-1/F-2/F-3 monitored).
Tests: seed each drift condition → detector flags; healthy state → clean report; repair-assertion (report never mutates rows). Verify: jest + harness. Rollback: disable service.

**T-67 — GBA audit subscription** `[modify]`
Files: `platform/audit/audit-subscriber.service.ts` (pattern `:123`). Prereq: T-62. Action: subscribe wash/release/status/voucher lifecycle events to audit trail (interim durability while ⛔ BLK-2 open). Expected: before/after actor trail per §8.2 requirements as far as event payloads allow.
Tests: audit row written for each subscribed type. Verify: jest. Rollback: revert subscription.

**T-68 — Canonical write-path switch (cutover prep)** `[modify] ⛔(T-03)`
Files: flag wiring in `create-allotment-pickup`/`cancel-allotment-pickup` (T-21/T-24 embed), rollout config (§11). Prereq: T-03, T-28, T-21, T-24, T-66 clean. Action: single flag selects canonical vs legacy write; legacy `allotment_pickups` marked read-only when on; switch sequence per §11. Expected: reversible cutover, no dual-write (justified: single-writer flag, §11).
Tests: flag-on tests (canonical written), flag-off tests (legacy written), no simultaneous writes. Verify: jest + harness. Rollback: flag off (§11).

**T-69 — Secondary-column non-authority tests (TR-15.7/9.2)** `[create-verify]`
Files: tests over reservation reads + pickup association. Prereq: T-32. Action: assert `reservations.block_code/group_block_id/pickup_type` never drive counter/association logic (pickup record does); legacy A4 `availability` never read by GBA paths. Expected: no second source of truth (§18).
Tests: as listed (mutate secondary column → behavior unchanged). Verify: jest. Rollback: n/a.

**T-70 — Global quality gates** `[verify]`
Files: repo-wide. Prereq: all. Action: run full gate battery — `cd apps/api && pnpm typecheck`, `pnpm test` (with `AVAILABILITY_TEST_DATABASE_URL` set for postgres specs), `cd apps/web && pnpm typecheck && pnpm lint && pnpm test`, schema validate/drift (T-06), grep gates (T-31/T-39/T-59/T-65). Expected: green baseline for Readiness Review. Tests: n/a (the run). Verify: commands above. Rollback: fix-forward (no releases until green).

---

## 6. Dependency Graph

### 6.1 Task-level dependencies (edges)

```
T-01 ─┬─► T-02 ─► T-49/T-50            (shoulder declaration → shoulder rewrite)
      ├─► T-06 (schema gate)
T-03 ⛔(RR) ─► T-21 ─► T-28 ─► T-68     (canonical pickup chain)
T-04 ⛔(RR) ─► T-45 (GUARANTEED_BLOCK exclusion line)
T-05 ⛔(RR) ─► T-45/T-46 (wash record) ─► T-55 (attrition record)

T-08 ─► T-09, T-18, T-43
T-12 ─► T-13, T-21
T-11 ─► T-21, T-46
T-14 ─► T-37
T-15 ─► T-22, T-23
T-16 ─► T-49

T-56 ─► T-20, T-21, T-22, T-23, T-24, T-26, T-29, T-32, T-43,
        T-45, T-46, T-53, T-57, T-58, T-61        ← seam is the widest gate
T-57 ─► T-60 ─► T-70

T-30 ─► T-31 (removal gated on consumers live)
T-32 ─┬─► T-24 interlock test T-36
      ├─► T-34/T-35/T-69 (invariance suite)
      └─► T-66 (reconciliation meaningful)
T-33 ─► T-31
T-63 ─► T-64

T-37, T-38 ─► T-39 (F-18 removal last among availability tasks)
T-45 ─► T-47 ─► T-48
T-52 ─► T-53 ─► T-54 ─► T-55
T-45 + T-52 + T-49 ─► T-51
T-59 ─► T-65
T-03 + T-21 + T-24 + T-28 + T-66 ─► T-68
ALL ─► T-70
```

### 6.2 Classification of task kinds

| Kind | Tasks |
|---|---|
| **Blocking (first)** | T-01, T-56 (plus RR sessions for T-03/04/05/T-41/T-47-transport/T-56-mechanism) |
| **Migrations first** | T-01, T-02, then ⛔ T-03/T-04/T-05 (RR-gated) |
| **Domain services first** | T-08–T-18 before intake rewrites (T-20–T-24) |
| **Event contracts first** | T-30, T-62 before consumers T-32/T-33/T-63; consumers before FO removal T-31 |
| **Reservation integration required** | T-32–T-36, T-66, T-69 |
| **Availability integration required** | T-26/T-41 (pickups consult), T-37–T-40 (authority), T-42 |
| **Frontend changes** | T-28, T-39, T-44, T-48 (+ views for pickup/wash surfacing under T-21/T-45) |
| **Reconciliation required** | T-66 (pre-cutover), T-68 gate |
| **Parallelizable (independent tracks)** | {T-08…T-18 domain guards} ∥ {T-01/T-02/T-06 schema} ∥ {T-37/T-38 availability reads} ∥ {T-30/T-62 event contracts} ∥ {T-59 sweep} — all before the intake-rewrite convergence |
| **Strictly sequential** | T-30 → T-32/T-33 → T-31; T-03 → T-21 → T-28 → T-68; T-56 → any §12.2 rewrite |

### 6.3 Recommended parallel tracks (after T-01/T-56)

- **Track A (schema/RR):** T-06, RR sessions (T-03/04/05), ⛔ tasks.
- **Track B (domain guards):** T-08…T-18, T-43, T-44.
- **Track C (authority):** T-37, T-38, T-39, T-42.
- **Track D (events):** T-30, T-62, T-63, T-64, T-40.
- **Track E (isolation):** T-59, T-65.
Convergence: intake rewrites (T-20–T-27) need B + D + seam; wash (T-45–T-48) needs B + A(T-04/05) + T-52/53; cutover (T-68) needs everything + T-66 clean.

---

## 7. Database / Migration Plan (D-2 reconciled — planning only, no migrations run)

### 7.1 Reconciliation matrix

| Item | In `schema.prisma` (declared) | In `migrations/` history | In live DB (verified 2026-09-30) | Gap action |
|---|---|---|---|---|
| `group_bookings` | ✅ `:17046` | ❌ none | ✅ | **T-01 create-from-history** |
| `group_blocks` | ✅ `:17079` | ❌ | ✅ | T-01 |
| `group_block_daily_allocations` | ✅ `:17113` | ❌ | ✅ | T-01 |
| `group_pickups` | ✅ `:17136` | ❌ | ✅ (extra col `room_number` — declare if missing) | T-01 + **T-03 ALTER** (⛔ BLK-3) |
| `allotment_contracts` | ✅ `:17161` | ❌ | ✅ | T-01 |
| `allotment_daily_quotas` | ✅ `:17198` | ❌ | ✅ | T-01 |
| `allotment_vouchers` | ✅ `:17221` | ❌ | ✅ | T-01 |
| `allotment_stop_sales` | ✅ `:17252` | ❌ | ✅ | T-01 |
| `group_block_analytics` / `allotment_analytics` | ✅ `:17275/:17293` | ❌ | ✅ | T-01 (include for completeness) |
| `overbooking_limits` | ✅ `:7434` | ✅ (pre-existing) | ✅ | none |
| `shoulder_days_before/after` on `group_blocks` | ❌ (**FIND-1**) | ❌ | ✅ | **T-02 declare** (+ include in T-01) |
| `group_blocks.block_type` | ❌ | ❌ | ❌ | **⛔ BLK-1 → T-04 (RR)** |
| wash/release/attrition stores | ❌ | ❌ | ❌ | **⛔ BLK-2 → T-05 (RR)** |
| `allotment_pickups` (live ledger) | ❌ undeclared | ❌ | ✅ (written by code today) | **T-03 collapse (S-1) + T-68 retire** |
| legacy `allotment`/`allotment_pickup`/`allotment_room_types`/`reservation_groups`/`reservation_block`/`allotment_ledger`/`availability` | ✅ (legacy models) | ✅/partial (`20260920` alters) | ✅ | read-only (WS-11) — **no change** |
| `reservations` GBA columns | ✅ | ✅ | ✅ | none (secondary, TR-15.7) |
| `penalty_due` | ✅ `:9718` | ✅ `20260902_s3_a3_penalty_due` | ✅ | none (reservation-scoped, unrelated to block attrition store — not a substitute for BLK-2) |

### 7.2 Migration ordering (when implementation starts — **not created now**)

1. `…_gba_baseline` — T-01 (+T-02 declarations). Non-destructive (`IF NOT EXISTS`). Applies cleanly over the live DB **and** reconstructs everything on a fresh DB.
2. `…_group_pickups_allotment_refs` — T-03 (after RR): nullable `group_block_id`, `allotment_id` + FK/index; optional backfill per RR.
3. `…_block_type` / `…_wash_attrition_store` — T-04/T-05 (after RR).
4. Optional Phase-2b CHECK constraints — T-07 (out of Phase-4 exit scope).

### 7.3 Backfill stance

- **Required:** none by default. T-01/T-02 are declarations of existing state (no data rewrite).
- **Conditional (RR review each):** (i) T-03 legacy `allotment_pickups` → `group_pickups` rows (migrate vs start-fresh); (ii) any BLK-1 legacy value normalization; (iii) none for BLK-2 (new store starts empty).
- **Prohibited:** any rewrite of counters, quota>physical rows, or legacy drift rows as part of migration (S-2 §2 — flagged, never repaired; Phase 11 owns drift cleanup).

### 7.4 Rules while planning/running migrations later

- Read-only verification only during planning (already done: `information_schema` checks — no writes).
- Every migration re-run-safe (`IF NOT EXISTS`/idempotent guards).
- No destructive statements in Phase 4 (drop happens only in Phase 11 with rollback criteria).

---

## 8. API Contract Plan

All traced to `13_FINAL_DOMAIN_SPECIFICATION.md`; **no endpoint invented for convenience**.

| Endpoint (verified route) | Current | Target change | Contract trace |
|---|---|---|---|
| `POST /group-bookings/:bookingId/blocks/:blockId/release` (`group-booking.controller.ts:247`) | `@Post` ✓ but frontend calls `PUT` (`group-allotment.api.ts:146`) | **Frontend caller → POST** (T-44); no PUT alias | D-12 |
| `POST /allotments/:id/release` (`allotment.controller.ts:239`) | `@Post` ✓, frontend `POST` ✓ | behavior guard change only (TR-5.1) — **T-43** | §8.3 |
| `GET /group-bookings/available-rooms` (`group-booking.controller.ts:95`) | counts `AVAILABLE` rooms (competing number) | **RETIRE (route removal)** + frontend hook removal (T-39) | D-6/F-18/TR-1.2 |
| `POST /allotments/:id/vouchers` (`:184`) | issues voucher | unchanged shape; guard-order + in-tx events (T-12/T-15) | §6.3 |
| `POST /allotments/:id/vouchers/:vid/consume` (`:206`) | fabricates reservation id server-side (handler `:26`) | **response `reservationId` = real id or null**; errors: `VOUCHER_ALREADY_USED`, `GUEST_MATCH_CONFLICT`, `ALLOTMENT_NOT_ACTIVE`; idempotent re-POST returns same reservation | D-10, S-1, §6.3 |
| `POST /allotments/:id/vouchers/:vid/cancel` (`:217`) | current restore logic | S-5 semantics (T-23); `409` when reservation live and not routed through cancel flow | S-5, TR-12.4/12.5 |
| `POST /allotments/:id/pickups` (`:251`) / `POST /group-bookings/:bookingId/blocks/:blockId/pickups` (`group-booking.controller.ts:236`) | multi-step, partial-commit | atomic unit (T-20/21); two-layer validation rejection (T-26) → `409 CONFLICT` / `422` per authority denial; deterministic codes | §7.1, §12 |
| `POST /allotments/:id/pickups/:pid/cancel` (`:271`) | unscoped reservation write (FIND-5) | hotel-scoped + restore-once (T-24); `409` for CHECKED_IN/CHECKED_OUT pickup (existing precondition pinned) | TR-12.3, TR-14.1 |
| `GET /allotments/:id/pickups` (`:281`) | reads legacy `allotment_pickups` | reads canonical `group_pickups` behind flag (T-28), **response shape preserved** (field parity verified in T-28 test) | S-1, TR-9.2 |
| `PUT /allotments/:id/quotas` (`:174`) | quota set | + quota≤physical rejection (`400 QUOTA_EXCEEDS_PHYSICAL`) (T-13); rollups in-unit (T-16) | S-2, TR-3.5 |
| `POST /allotments/:id/stop-sales` + `.../lift` (`:195/:228`) | flag desync risk (F-10) | same-tx flag sync (T-14); no shape change | TR-7.x, S-4 |
| `POST /group-bookings/:bookingId/blocks/:blockId/shoulder` + `/remove` (`:267/:277`) | accepts zero-quantity adds (TR-8.2 violation) | **request requires explicit quantities** (reject silent 0) (T-49); removal → `409` when active pickups (T-50) | TR-8.2/8.3 |
| `DELETE /group-bookings/:id`, `.../blocks/:blockId`, `DELETE /allotments/:id` (`:297/:304/:321`) | verify soft-delete (T-17) | align to soft-delete if hard (A1 assumes `deleted_at`) | TR-15.7, A1 filters |
| Wash execution (T-45/46) | **no HTTP surface exists** | **none created** — scheduler invokes handlers via `WashSchedulerPort` (T-47). Manual trigger endpoint **not mandated by contract → not invented** | §8, TR-6.6 |
| A3 `GET availability/matrix` (`availability-sales.controller.ts:237`) | self-computed numbers | values from authority; **shape preserved**; interim label per D-6a option-B if phased | D-6a, TR-10.1 |
| Master folio `GET/POST` routes (`group-booking.controller.ts:143-177`) | existing | unchanged; receive `ATTRITION_FEE` postings from wash unit (T-53) | TR-6.9 |

**Cross-cutting API rules (T-61):** hotel scoping via existing property-scope guard (all routes already hotel-scoped — raw-SQL layer is the defect, T-59); idempotency via existing `core/interceptors/idempotency.interceptor.ts` where already wired (`[RR]` request-dedup mechanics §13); error semantics: deterministic codes + `409 CONFLICT` for TR-11.2 conflicts; authorization unchanged (`group-allotment.permissions.ts` incl. existing `GROUP_BLOCK_WASH`).

---

## 9. Test Strategy

**Infrastructure (existing, verified):** jest (`apps/api/package.json` — `testRegex .spec\.ts$`, rootDir `src`); Postgres harness `apps/api/src/modules/reservations/infrastructure/__tests__/availability-postgres.harness.ts` (`describePostgres` = `describe` only when `AVAILABILITY_TEST_DATABASE_URL` set, else skip; `splitSqlStatements`; ReadCommitted `txOptions`); compose `xylo-postgres` verified running. Web: `apps/web/jest.config.js` + `pnpm lint`. **GBA has zero tests today — everything below is create.**

### 9.1 Levels

| Level | Location pattern | Scope |
|---|---|---|
| Unit (domain) | `group-allotment/domain/**/__tests__/*.spec.ts` | aggregates/VOs/services: transitions, guards, math, mapping (T-08…T-18, T-52) |
| Domain/service | `group-allotment/application/commands/*/__tests__/*.spec.ts` | handler behavior with mocked repos/ports (T-20…T-36, T-43…T-50) |
| Integration (Prisma) | `group-allotment/infrastructure/__tests__/*.postgres.spec.ts` (harness pattern) | real DB units: atomicity, idempotency, canonical rows, wash (T-20/21/24/45/46/53) |
| PostgreSQL-level | same harness + `splitSqlStatements` | migration T-01/T-02 from-scratch, constraints T-07, uniques (voucher code), isolation matrix |
| Transaction/concurrency | `*.postgres.spec.ts` (pattern: `availability-assertion-concurrency.spec.ts`) | concurrent consume/cancel/wash, version conflicts, double delivery (T-56…T-61, T-64) |
| API | controller tests (existing route-test style) + contract tests | status codes/error codes (T-61), route absence (T-39), verb (T-44) |
| End-to-end workflow | scripted scenarios over services (issue→consume→checkout→cancel; wash day; shoulder add→pickup→wash→attrition) | §9.2 mandatory scenarios chained |
| Reconciliation | `gbA-reconciliation` tests (T-66) | detectors + no-mutation proof |
| Frontend | `apps/web` jest + `pnpm typecheck`/`pnpm lint` | T-28/T-39/T-44/T-48 hooks/views |

### 9.2 Mandatory scenarios (contract §-level coverage map)

| # | Scenario | Contract anchor | Tasks |
|---|---|---|---|
| 1 | quota ≤ physical enforced at create/set-quota; historical violations flagged not rewritten | S-2, TR-1.5/3.5, TR-10.4 | T-13, T-42, T-66 |
| 2 | pickup consume (block + allotment paths) atomic, counters correct | S-1, TR-4.2/3.2 | T-20, T-21, T-57 |
| 3 | duplicate consume (same voucher) → one reservation, no double counters | TR-11.4, spec §32 | T-22, T-29, T-64 |
| 4 | reservation linkage: real `reservation_id`, hotel-scoped, fabricated ids impossible | D-10, TR-3.4/9.3 | T-22, T-25 |
| 5 | reservation cancellation → pickup CANCELLED + restore exactly once + invalidate | D-4, TR-12.1/12.3/12.6 | T-32, T-36, T-64 |
| 6 | USED voucher cancellation (both S-5 branches) | S-5, TR-12.4 | T-23 |
| 7 | ISSUED voucher cancellation restores quota | TR-12.5 | T-23 |
| 8 | concurrent consume (N parallel) → exactly one wins, rest `CONFLICT` | TR-11.1/11.2/11.3 | T-56…T-58 |
| 9 | concurrent cancellation vs consume; no double restore | TR-11.4/11.5 | T-36, T-58 |
| 10 | concurrent wash + wash same range → second no-op | INV-10/11, spec §32 | T-45, T-46 |
| 11 | repeated wash (sequential) → no-op, logged once | TR-6.1, spec §32 | T-45, T-53 |
| 12 | rolling release T-2 window math + N-day honoring | TR-6.2 | T-46 |
| 13 | GUARANTEED_BLOCK never washes (allotment now; block ⛔ after T-04) | TR-6.3 | T-46, T-45-gated |
| 14 | shoulder days: same guards, wash coverage, attrition base, unrestricted pickup (P1/P2/P3) | D-13, TR-8.1–8.6 | T-49…T-51 |
| 15 | attrition: 80% default, ceil math, shortfall, liability, single assessment | D-14, TR-6.7/6.8 | T-52, T-53 |
| 16 | master folio `ATTRITION_FEE` posting inside wash transaction | TR-6.9 | T-53 |
| 17 | A1/A3 parity; F-18 absent; single availability number | D-6/D-6a, TR-10.1/1.2 | T-38, T-39, T-42 |
| 18 | hotel isolation across every mutation/read (foreign hotel → no effect) | TR-14.1–14.5 (S-6) | T-59 + per-suite negatives |
| 19 | legacy containment: no GBA writes to legacy tables/A4; secondary columns non-authoritative | TR-15.1/15.2/15.5/15.7 | T-65, T-69 |
| 20 | checkout → pickup `CHECKED_OUT`, counters unchanged; FO has no GBA SQL | TR-4.8/9.4, L-13 | T-31, T-33 |
| 21 | check-in / no-show invariance | §7.3 | T-34 |
| 22 | modify/extend counter preservation (SOURCE-SILENT guard) | §7.3 | T-35 |
| 23 | stop-sale: blocks unaffected, quantity untouched, ordering before remaining | S-4, TR-7.1/7.2/7.4 | T-14, T-12, T-37 |
| 24 | guest identity order + conflict, no fuzzy matching | D-11, TR-4.10/9.6 | T-25 |
| 25 | block lifecycle full matrix incl. `DRAFT→TENTATIVE` | D-15, TR-2.1–2.3 | T-08 |

**Gates:** every task's `Verify:` command must pass before the next dependent task starts; T-70 is the Readiness-Review entry gate.

---

## 10. Observability (requirements — not implemented now)

### 10.1 Event contract mapping table (spec §16 ↔ implemented types)

| Spec §16.1/16.2 name | Implemented type (verified) | Status |
|---|---|---|
| `BlockCreated` / `BlockStatusChanged` | `group_block.created` / `group_block.confirmed`·`closed`·`cancelled` (`group-allotment.events.ts:66/:83/:198/:215`) | existing — map + enrich status payload (T-62) |
| `PickupCreated` / `PickupCancelled` | `group_pickup.created :127` / `group_pickup.cancelled :145` | existing |
| `VoucherIssued/Consumed/Cancelled` | `allotment.voucher_issued :291` / `voucher_consumed :310` / `voucher_cancelled :329` | existing |
| `ManualReleaseExecuted` | `group_block.released :163` / `allotment.released :386` | existing |
| `CutOffWashExecuted` | `group_block.washed :181` | existing (enrich before/after, T-45) |
| `AllotmentWashExecuted` | — | **create (T-46/T-62)** |
| `ShoulderAllocationChanged` | alias of `group_block.allocation_changed :104` (or new class — T-62 choice, documented) | map/create |
| `StopSaleApplied/Lifted` | `allotment.stop_sale_applied :350` / `stop_sale_lifted :368` | existing |
| `AttritionAssessed` | — | **create (T-54/T-62)** |
| `reservation.cancelled` | `ReservationCancelledDomainEvent` (`reservation-domain.events.ts:26`) | existing |
| `reservation.checked_out` | — (checkout emits `ReservationUpdatedDomainEvent` + `FrontOfficeCheckedOut` `:254/:263`) | **create (T-30/T-62)** |
| quota/batch-set events (spec `[SPEC-CARRIED]`) | `allotment.quota_changed :272`, `group_block.allocation_changed :104` | existing |

### 10.2 Requirements

- **Structured logs:** every handler logs operation, `hotelId`, aggregate id, counter before/after on mutation, outcome; errors include stack (existing `Logger` pattern — GBA handlers already log; standardize fields in T-20…T-58).
- **Audit events:** subscribe wash/release/status/voucher/attrition to `platform/audit/audit-subscriber.service.ts` (T-67); interim durable trail while ⛔ BLK-2 open.
- **Correlation IDs:** outbox already carries `correlationId`/`causationId`/`eventId` (`event-bus.ts:23-31`) — preserve through consumers; log with job id (dedup key).
- **Identifiers on every log/audit line:** `reservationId`, pickup id, voucher code, block id, `block_code`, allotment id, `allotment_code`, wash key, idempotency key.
- **Transaction failures:** unit failures logged with step name + rollback confirmation; surfaced to caller (no swallow — D-9); consumer failures rethrow for retry with attempt count (`events.consumer.ts:61-63`).
- **Reconciliation alerts:** T-66 report emitted on cadence + on-demand; any detector hit → alert with hotel + entity ids (read-only; never auto-repair).
- **Metrics:** no metrics platform verified in-repo → **requirements only**: counters for wash runs/no-ops, consumer retries, `CONFLICT` rate, reconciliation detector hits, invalidation counts — **implementation vehicle chosen at Readiness Review if/when a metrics stack exists (not invented here)**.

---

## 11. Rollout / Cutover Plan

**Principle:** non-destructive first; every switch reversible; **no dual-write** (single-writer + flag, justified: one writer per table at any time).

1. **Migration sequence (implementation phase):** T-01/T-02 baseline (`IF NOT EXISTS`, safe over live) → RR-gated T-03/T-04/T-05 → (Phase 2b) T-07. Verify with T-06 drift gate before any code deploy.
2. **Feature flags (in order of use):**
   - `gba.consumers.cascade` — enable T-32/T-33 consumers **before** T-31 (FO raw SQL removal) — hard ordering rule.
   - `gba.wash.schedulerEnabled` — off until T-47 transport confirmed at RR + wash suite green; controls TR-6.5 authority labeling (T-48).
   - `gba.pickup.canonicalWrite` / `gba.pickup.canonicalRead` — BLK-3 cutover (T-68): enable read first (parity-tested T-28), then write, then legacy becomes read-only.
   - `gba.a3.authoritative` — A3 rebuild switch (T-38); interim labeled view allowed while off (D-6a option B).
3. **Dual-read/dual-write:** dual-**read** only for the pickup parity window (T-28 compares legacy vs canonical responses); dual-write **not used** (avoids divergence — §19 single-truth rule).
4. **Reconciliation period:** after canonical write-on and after consumers-on, run T-66 detectors for a soak window; **exit gate: zero detector hits attributable to the new path** (legacy drift hits are expected and reported, not blocking — Phase 11 owns them).
5. **Activation sequence:** schema → domain guards → seam → consumers → intake rewrites → authority reads → A3 → F-18 removal → wash (flag) → attrition (with wash) → canonical cutover → FO raw-SQL removal → sweep gates green (T-70).
6. **Legacy retirement criteria (Phase 11 — not in this phase):** canonical ledger authoritative + reconciliation clean + no legacy reads by GBA + cutover rehearsal done (per AGENTS.md Phase 11). **No legacy code/table deletion in Phase 4** (contract §18 requires retention).
7. **Rollback strategy:** every flag off → previous behavior; migrations non-destructive (rollback = flag off + optional down-script for ⛔ additions only); consumer off during incident = cancellation/checkout events queue (outbox retains) and drain on re-enable — **documented monitored window**: while consumers off, cancellations do not restore quota (risk R-04).
8. **Monitoring gates:** (a) reconciliation report, (b) consumer retry/error logs, (c) `CONFLICT` rate, (d) A1 `UNRESOLVED` counts before/after (T-37 must not increase), (e) grep gates in CI.

---

## 12. Risk Register

| ID | Risk | Impact | Likelihood (evidence) | Mitigation | Detection | Rollback |
|---|---|---|---|---|---|---|
| R-01 | Partial commits during pickup rewrites | Wrong quota + orphan reservation (data integrity) | **High** today — multi-step + swallowed `rollbackQuota` (`create-allotment-pickup.handler.ts:71-185`; C-1…C-6) | T-56 seam + T-57/T-60 fault-injection suite; D-9 rule | fault-injection tests; reconciliation detector (a) | deploy revert (flag off) |
| R-02 | Concurrent counter lost-update during rewrite window | Over-sell / invariant breach | **High** exposure (no mechanism yet — TR-11.7 deferred) | RR selects mechanism at readiness; seam isolates it; version columns already present | concurrency suite (T-56…58); detector (b) | revert to serialized path if RR picks locks |
| R-03 | Double restore (cancel command + cascade event) | Under-counting → over-sell later | **Medium** — interaction exists (`cancel-allotment-pickup.handler.ts:66` cancels reservation → event) | T-36 interlock + status preconditions (INV-6) | interlock test; detector (a) | flag cascade off (restores become manual) |
| R-04 | Consumers off while cancellations occur | Counters not restored until replay | **Medium** (flag windows) | enable consumers before T-31; outbox retains events; soak monitoring | consumer retry logs; detector (a) delta | re-enable drains queue (at-least-once) |
| R-05 | BLK-3 unresolved → dual ledgers persist | S-1 violation continues; drift grows | **High until RR** (live `allotment_pickups` written today) | RR session early (Track A); read parity (T-28) prepares cutover | detector (e) drift count | legacy path remains (flag off) |
| R-06 | Wash ships without type exclusion/log (BLK-1/2 partial) | GUARANTEED_BLOCK washed (rule breach) or unauditable wash | **Medium** — blocked on RR | gate the exclusion/log lines behind BLK resolution (T-45 marked ⛔); wash flag off until conformance | wash suite (gated tests) | flag off (no scheduled wash) |
| R-07 | Availability divergence (A3 interim vs A1) | Two numbers visible (D-6 violation) | **Medium** during T-38 phasing | option-B labeling mandatory; parity test; short interim window | parity test; label assertion | flag to labeled legacy view |
| R-08 | Hotel-isolation defect survives sweep | Cross-tenant data exposure/write | **Medium** — known unscoped write exists (FIND-5, `cancel-allotment-pickup.handler.ts:66`) | T-59 static gate + isolation matrix | static gate in CI; ISO postgres suite pattern | hotfix revert |
| R-09 | Migration drift (tables exist only live) | Fresh env / CI cannot run | **High** today (FIND-4 — zero migrations create GBA tables) | T-01/T-02 + T-06 drift gate | `prisma migrate diff` gate | N/A (additive) |
| R-10 | Event replay/duplicate causes double effects | Wrong counters/status | **Medium** (at-least-once by design) | idempotent consumers (T-64), event-id dedup | duplicate-delivery suite; detector (a) | disable consumer case |
| R-11 | Attrition posted outside wash tx (regression) | Double/missed penalty, ledger mismatch | **Low** if T-53 tests hold; **High** impact | single-unit test with failure injection | T-53 tests; folio reconciliation | wash flag off |
| R-12 | F-18/A3 frontend callers break on retirement/rebuild | UI errors | **Medium** (callers verified: `group-allotment.api.ts:29`, `useAvailableRooms`) | T-39 updates callers; T-38 shape-preserved | web typecheck/lint/tests; route-absence test | redeploy previous artifact |
| R-13 | Backfill of legacy `allotment_pickups` miscounts | Wrong canonical history | **Medium** if RR picks migrate | parity report (T-66 e) before switch; migrate-vs-fresh decided at RR with data inspection | drift report | flag to legacy read |
| R-14 | Guest-identity strictness breaks existing callers (no fuzzy match) | Pickups fail to find existing guest → duplicates | **Medium** — D-11 deliberately tightens | conflict error surfaces explicitly; caller audit in T-25; duplicate-guest detection via reconciliation | `GUEST_MATCH_CONFLICT` rate log | caller-level fallback (create-new branch) |

---

## 13. Deferred Items (mechanism-shaped — NOT converted into business decisions)

| # | Item | Already decided | Deferred | Why | Where resolved | Blocks implementation? | RR confirmation required? |
|---|---|---|---|---|---|---|---|
| DEF-1 | **Wash scheduler transport (TR-6.6)** | Wash semantics: when (cutoff date / N-day window), date-granularity, single event, idempotency, atomicity, events (§8.2) | Which transport runs `WashSchedulerPort.runDueWashes` | mechanism, not policy | **Readiness Review** — alternatives: (a) BullMQ repeatable jobs (infra exists `infrastructure/bullmq/`), (b) `@nestjs/schedule` cron, (c) Temporal (compose service exists) | **No** — port + tests ship; production scheduling waits for RR pick | **Yes** |
| DEF-2 | **Concurrency lock mechanism (TR-11.7)** | Invariants: no lost updates, `CONFLICT` on stale, inequalities after commit, exactly-once, version monotonicity (§12.1) | Optimistic-version-retry vs row locks vs advisory locks; lock ordering; isolation level | mechanism | **Readiness Review** — alternatives in T-56; harness baseline ReadCommitted | **No** — seam ships with pluggable strategy; invariant tests pass against chosen strategy post-RR | **Yes** |
| DEF-3 | **Pickup → assertion port shape (D-7 note)** | Two-layer validation required; pickup-created reservations pass availability lifecycle (TR-4.1/10.6); single authority | Assert-at-write vs read-time consult + assertion consumption wiring; replacement of dead `inventory-commitment.port.ts` | mechanism | **Readiness Review** — alternatives in T-41/T-26 | Partially — intake rewrites ship behind seam; final consult wiring awaits RR (interim: read-time consult against `availability-snapshot.service.ts` if RR confirms) | **Yes** |
| DEF-4 | **`ATTRITION_FEE` ledger ownership (spec open Q2)** | Assessment + posting inside wash unit as `ATTRITION_FEE` to master folio (TR-6.7/6.9); posting uses existing folio structures | Which module ultimately owns/queries the ledger entry | architecture boundary | **Readiness Review** (implementation dependency, listed §10.5 of spec) | No — posting implemented via existing `prisma-folio.port` | **Yes** (dependency note) |
| DEF-5 | **Guest-identity caller contract (D-11 note)** | Lookup order, hotel-scoped exact matching, conflict handling (TR-4.10/9.6) | Adapter/controller changes needed to surface conflicts explicitly | caller mechanics | Implementation task T-25 with RR review of conflict UX | No | No (tracked as T-25 review) |
| DEF-6 | **Reservation confirmation semantics for pickup-created reservations (TR-9.7 note)** | Not hardcoded to a fixed status; honors reservation domain semantics | Exact status chosen by existing reservation state machine at creation | mechanism | Implementation (reservation state machine already exists — `reservation-status.service.ts`) | No | No |
| DEF-7 | **Request-level idempotency dedup mechanics (TR-11.4 note)** | Exactly-once counter mutation per business event; business-key dedup (voucher code, status preconditions) | Whether/where `Idempotency-Key` header interceptor wraps GBA routes | mechanism | Readiness Review (interceptor already exists `core/interceptors/idempotency.interceptor.ts`) | No — business-key idempotency ships regardless | **Yes** (optional scope) |
| DEF-8 | **Forward migration mechanism family detail (D-2 note)** | Declared schema is source of truth; reproducibility required (TR-15.3/15.4); repo standard = Prisma migrate (`pnpm db:migrate`) | Exact CI wiring of the drift gate | mechanics | Implementation (T-06) | No | No |
| DEF-9 | **`EXPIRED` voucher transition mechanism (TR-13.3 note)** | States + rejection semantics (§6.3); expiry does not wash (TR-13.4) | Read-derived expiry vs background sweep | mechanism | Implementation choice at T-15 (both satisfy the rule) | No | No |
| DEF-10 | **Metrics vehicle (observability)** | Metric requirements listed §10.2 | Which stack (none verified in-repo) | tooling | Readiness Review | No | **Yes** (if metrics in scope) |

**Rule preserved:** none of the above may appear in code comments/docs as a decided business rule; each ships as an interface + alternatives + RR decision record.

---

## 14. Traceability — Domain Spec Rule → Workstream → Tasks → Tests → Verification

Every TR family of `11_TARGET_BUSINESS_RULES.md` (97 = 95 decided + 2 deferred) and all 22 decisions / 6 confirmations map to implementation work. **No orphaned rule; no unanchored task.**

| Contract § / TR family (rules) | Decision(s) / Confirmation | WS | Task IDs | Tests (§9 rows) | Verification |
|---|---|---|---|---|---|
| §11 facts/authority — TR-1.1, 1.2, 1.3, 10.1, 10.2, 10.6, 15.5 | D-6, D-6a | WS-05, WS-10 | T-37, T-38, T-39, T-40, T-42, T-63 | 17, 18 | adapter specs; A3 parity; route-absence |
| TR-1.4 (pickup vs availability) | D-7 | WS-05, WS-03 | T-26, T-41 ⛔[RR] | 2 | assertion suites |
| TR-1.5, 3.5, 10.4, 10.5 (quota≤physical, explicit overbooking) | **S-2 = A** | WS-02, WS-05, WS-11 | T-13, T-42, T-66(c) | 1, 17 | create/quota rejection tests; flagged-not-rewritten |
| TR-1.6, 3.6 (contract-type mapping) | **S-3 = A** | WS-02 | T-11 | 25 (mapping unit) | mapping table tests |
| §4 blocks — TR-2.1…2.7 | D-15, D-1 | WS-02 | T-08, T-09, T-16, T-18 | 25 | lifecycle matrix; code uniqueness |
| TR-3.1, 3.2, 3.3, 13.1, 13.2, 13.4, 13.5 | D-1, D-15 | WS-02 | T-10, T-12 | 23 (guards) | guard unit tests |
| TR-3.4 (real reservation id) | **D-10 = B** | WS-03 | T-22 | 3, 4 | real-linkage tests |
| §5/§7 pickup — TR-4.1, 4.2, 4.4, 4.5, 4.6, 9.1, 9.3, 9.5, 9.7 | D-7, D-9, D-10, S-1 | WS-03, WS-09 | T-20, T-21, T-22, T-26, T-56…T-61 | 2, 4, 8, 18 | atomicity + linkage + conflict suites |
| TR-4.7, 8.6c (shoulder pickup) | **D-13 = A** | WS-07 | T-49, T-51 | 14 | shoulder suite |
| TR-4.8, 9.4 (checkout no counter) | D-4 | WS-04 | T-31, T-33 | 20, 21 | consumer + grep gate |
| TR-4.9, 3.4, 9.2, 15.6, 15.7 (one pickup record / association) | **S-1 = A**, D-1 | WS-03, WS-11 | T-21, T-28, T-68, T-69 | 4, 19 | canonical reads; non-authority tests |
| TR-4.10, 9.6 (guest identity) | **D-11 = C amended** | WS-03 | T-25 | 24 | order/conflict/isolation tests |
| §8 release — TR-5.1…5.5 | D-12, D-3 | WS-06 | T-43, T-44 | 12, 18 | bounds + verb tests |
| §8 wash — TR-6.1, 6.2, 6.4, 6.5, 6.7, 6.8, 6.9; 6.3; 6.6 | D-3, D-14, **D-6**; TR-6.6 **[DEFERRED]** | WS-06, WS-08 | T-45 ⛔partial, T-46, T-47 [RR], T-48, T-52, T-53, T-55 ⛔ | 10, 11, 12, 13, 15, 16 | wash/attrition suites; scheduler port review |
| §9 shoulder — TR-8.1, 8.2, 8.3, 8.4, 8.5, 8.6a, 8.6b | **D-13 = A**, D-2 | WS-07, WS-01 | T-01/T-02, T-49, T-50, T-51 | 14 | shoulder + declaration tests |
| §12 cancellation — TR-12.1, 12.2, 12.3, 12.6 | D-4, **S-5 = A** | WS-04, WS-03 | T-23, T-24, T-32, T-36 | 5, 6, 7, 9 | cascade + S-5 suites |
| TR-12.4 (restore iff cancelled) | **S-5 = A** | WS-03 | T-23, T-32 | 6 | both-branch tests |
| TR-12.5 (issued cancel restore) | D-4 family | WS-03 | T-23 | 7 | restore test |
| §7 stop sale — TR-7.1…7.5 | **S-4 = A** | WS-02, WS-05 | T-12, T-14, T-37 | 23 | ordering + display + scope tests |
| §10 concurrency — TR-11.1…11.6; 11.7 | D-8, D-9; TR-11.7 **[DEFERRED]** | WS-09 | T-56 [RR], T-57…T-61 | 8, 9, 10 | concurrency + atomicity suites |
| §13 expiry — TR-13.3 | D-15 family | WS-02 | T-15, T-62 | 25 (matrix) | transition tests |
| §14 isolation — TR-14.1…14.5 | **S-6 = A** | WS-09, WS-11 | T-59, T-65 | 18 | ISO matrix + static gate |
| §15 legacy — TR-15.1, 15.2, 15.3, 15.4, 15.7 | D-1, D-2 | WS-01, WS-11 | T-01, T-02, T-06, T-65, T-69 | 19 | drift gate + legacy gates |
| §16 events — D-5 contract (every event consumed, event-less prohibited) | D-5 | WS-10 | T-40, T-62, T-63, T-64 | 17, 25 (event emission), duplicates | routing matrix + static coverage |
| Confirmation **D-2 = A** (+ live-DB PASS) | D-2 | WS-01 | T-01, T-02, T-06 | 4 (migration) | `prisma validate` + drift |
| Confirmation **D-4 = B/C** | D-4 | WS-04 | T-30…T-33, T-36 | 20 | consumer + grep gates |
| Confirmation **D-6a = A** | D-6a | WS-05 | T-38 | 17 | A3 parity + label |
| Confirmation **D-10 = B** | D-10 | WS-03 | T-22, T-27 | 3, 4 | real-id + anomaly tests |
| Confirmation **S-1 = A** | S-1 | WS-03 | T-21, T-28, T-68 | 2, 4 | canonical single ledger |
| Confirmation **S-3 = A** | S-3 | WS-02 | T-11 | mapping | mapping round-trip |
| Spec `SOURCE-SILENT` spots (§7.3 modify/extend/no-show status; D-10 anomaly) | (none — silence preserved) | WS-04, WS-03 | T-27, T-34, T-35 | 21, 22 | invariance tests (no invented behavior) |
| Spec §17 declared-schema gaps (FIND-1…4, BLK-1…3) | D-2 mechanics | WS-01 | T-01…T-05 [RR] | migration harness | drift gate |

**Coverage count:** 22/22 decisions · 6/6 confirmations · 97/97 TRs (95 decided mapped to tasks/tests; TR-6.6 & TR-11.7 mapped as `[DEFERRED]` rows → §13 DEF-1/DEF-2) · contract §1–§20 all represented.

---

## 15. Implementation Order (derived from §6 graph)

```
Phase A — Foundation (parallel)
  A1: T-01, T-02, T-06 (schema declared)          A2: T-56 seam + RR session #1 (DEF-1/2/3)
  A3: T-08…T-18 domain guards                      A4: T-30, T-62 event contracts
  A5: T-59 hotel-isolation sweep
Phase B — Core linkage
  B1: T-20, T-21, T-22, T-23, T-24, T-25, T-27, T-29 (intake/cancel rewrite)
  B2: T-32, T-33 consumers (+T-63 routing, T-40 invalidation)  [BEFORE B3]
  B3: T-31 FO raw-SQL removal
  B4: T-34, T-35, T-36 invariance/interlock, T-60 fault injection
Phase C — Authority
  C1: T-37, T-42, T-61  →  C2: T-38 (A3 rebuild)  →  C3: T-39 (F-18 retire)
  C4: T-26/T-41 consult wiring (per RR #2)
Phase D — Wash/Attrition/Shoulder
  D1: T-43, T-44 (manual release)   D2: T-49, T-50 (shoulder)
  D3: T-52 → T-53 → T-54 (attrition)  D4: T-45 ⛔, T-46, T-47 [RR #3], T-48, T-51, T-55 ⛔
Phase E — Hardening
  E1: T-57, T-58 adoption batches   E2: T-64, T-67  E3: T-65, T-69 legacy gates
Phase F — Reconciliation & Cutover
  F1: T-66 detectors → soak  F2: T-03 ⛔/T-68 canonical switch → parity window
  F3: T-70 full gates → READINESS REVIEW ENTRY
```

---

## 16. Implementation Readiness Checklist (for Readiness Review)

- [ ] **All required files identified** — every touched path verified against the repo (§3, §4, §5; e.g. handlers/controllers/adapters/hooks listed above exist on disk; legacy components cited by audit `AllotmentDetail.tsx`/`AllocationStatus.tsx` verified **absent → already removed**, no task targets them).
- [ ] **All schema changes identified** — T-01, T-02, ⛔T-03/04/05, optional T-07 (§7).
- [ ] **All migrations identified** — ordering + `IF NOT EXISTS` discipline + none created now (§7.2).
- [ ] **All API changes identified** — retire 1 route, frontend verb fix, request-shape changes (shoulder quantities), error semantics; **no invented endpoints** (§8).
- [ ] **All event contracts identified** — mapping table (§10.1), 4 new types, consumer matrix (T-63), INV-19 coverage.
- [ ] **All transaction boundaries identified** — §12.2 rows mapped to T-56…T-58 (per-op table in WS-09 + contract §12.2).
- [ ] **All concurrency mechanisms identified or explicitly deferred** — seam shipped, mechanism = DEF-2 [RR]; isolation level flagged.
- [ ] **All tests identified** — §9 levels + 25 mandatory scenarios mapped to task IDs.
- [ ] **All rollback paths identified** — per-task rollback fields + §11 flags + risk register R-rolls.
- [ ] **All legacy boundaries identified** — WS-11 read-only list, A4 freeze, secondary columns, `allotment_pickups` retirement path.
- [ ] **All target rules traceable** — §14 matrix: 97/97 TRs, 22/22 decisions, 6/6 confirmations.
- [ ] **No unresolved business decisions** — 0; all open items are mechanisms → §13 (10 items, 6 RR-confirmed).
- [ ] **No hidden assumptions** — blockers BLK-1/2/3 documented §1.6; SOURCE-SILENT preserved; backfill stance explicit (§7.3); metrics/transport/ledger choices exposed as RR items.
- [ ] **Deferred items remain deferred** — DEF-1…DEF-10 unchanged in kind (mechanism, not rule).
- [ ] **Code/schema/API/test changes made during planning: NONE.**

---

## Final Quality Gate

| Gate | Result |
|---|---|
| **A. Domain coverage** | PASS — contract §1–§20 mapped (§14; WS per §4; §16 events, §17 schema, §18 legacy, §19 recon, §20 traceability all have tasks) |
| **B. Decision coverage** | PASS — 22/22 in §14 with task+test anchors |
| **C. Confirmation coverage** | PASS — 6/6 rows in §14 |
| **D. Deferred coverage** | PASS — DEF-1…DEF-10 preserve TR-6.6/TR-11.7 + 8 other mechanism deferrals as RR items; none converted to rules |
| **E. Repository coverage** | PASS — every path in §3/§4/§5 verified this session (controllers, handlers, adapters, consumers, hooks, views, schema lines, migrations list, live-DB `information_schema` checks); non-existent audit-era files (`AllotmentDetail.tsx`, `AllocationStatus.tsx`, `GroupBookingPanel.tsx`) confirmed removed and not targeted |
| **F. Dependency correctness** | PASS — §6 graph; §15 order never places a task before prereq (consumers before FO removal; seam before rewrites; A3 before F-18 removal; RR gates before blocked tasks) |
| **G. No hidden business decisions** | PASS — 0 new rules; contract silence preserved (T-27/34/35); blockers documented not resolved (§1.6) |
| **H. No premature implementation** | PASS — only `14_IMPLEMENTATION_PLAN.md` created; no code/schema/migration/API/test/data changes |

---

## Completion Report

- **Document created:** YES — `docs/availability/phase-4/14_IMPLEMENTATION_PLAN.md`
- **Status:** COMPLETE / READY FOR READINESS REVIEW
- **Domain rules traced:** 97/97 (95 decided → tasks/tests; 2 deferred → §13 DEF-1/DEF-2)
- **Decisions traced:** 22/22
- **Confirmations traced:** 6/6
- **Implementation tasks:** 70 (T-01…T-70; 6 RR-gated, 4 blocked on BLK-1/2/3)
- **Schema/migration impacts identified:** YES (T-01, T-02, ⛔T-03/04/05, optional T-07; §7 full reconciliation)
- **API impacts identified:** YES (§8 — 1 retirement, 1 verb fix, 2 request/response changes, error semantics; 0 invented endpoints)
- **Event contracts identified:** YES (§10.1 mapping + 4 new types + consumer matrix)
- **Transaction/concurrency plan complete:** YES (§12.2 per-op boundaries; seam + `[DEFERRED]` mechanism alternatives §13 DEF-2)
- **Tests mapped:** YES (§9 — 8 levels, 25 mandatory scenarios → task IDs)
- **Deferred mechanisms preserved:** YES (DEF-1…DEF-10; TR-6.6 & TR-11.7 remain deferred)
- **Unresolved business decisions:** 0 (blockers BLK-1/2/3 are mechanism/storage questions routed to Readiness Review, not reopened decisions)
- **Code/schema/API/test changes made:** NONE
- **Next phase:** **Readiness Review**

**STOP after creating the Implementation Plan.**










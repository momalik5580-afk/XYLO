# Phase 4 — Audit Summary & Gate Status

**Phase:** 4 — GBA / Allotment Integration (Forensic Audit only)
**Date:** 2026-09-30
**Deliverables:** 9/9 written to `docs/availability/phase-4/`
**Scope honored:** no code, schema, migration, backfill, API, or behavior changes; no Phase 1–3 reopening; no implementation plan.

---

## 1. Phase 4 Audit Gate

```
╔══════════════════════════════════════════════════════════════════════╗
║  PHASE 4 FORENSIC AUDIT STATUS:  COMPLETE — EVIDENCE-BASED PASS     ║
║  PHASE 4 IMPLEMENTATION READINESS: NOT READY — BLOCKED BY DECISIONS ║
╚══════════════════════════════════════════════════════════════════════╝
```

| Gate criterion | Status | Evidence |
|---|---|---|
| All 9 deliverables present in `docs/availability/phase-4/` | **PASS** | `01`…`09` |
| Every conclusion cited with `file:line` | **PASS** | all deliverables |
| CURRENT vs TARGET separated | **PASS** | `05_…` §1 vs §2 vs §3 |
| Existing Phase 4-adjacent docs identified, not duplicated | **PASS** | `01_…` §1 (`docs/design/gba-domain-spec.md`, `docs/enterprise/availability-phase3-*`) |
| No implementation performed | **PASS** | read-only commands only |
| Audit trail sufficient to start implementation | **BLOCKED** | 15 open decisions in `08_OPEN_DECISIONS.md` (D-1…D-15), incl. D-2 (schema reproducibility), D-15 (block lifecycle) and D-8 (concurrency guarantee) |

**Interpretation:** the *audit* phase may be closed. The *next* phase may not start until D-2 (what exists in the live DB / how it is migrated) and D-8/D-9 (the concurrency + transaction guarantees for pickup) are decided.

---

## 2. Findings Register (ranked)

| # | Finding | Severity | Evidence | Decision |
|---|---|---|---|---|
| F-1 | New GBA tables have **no migration**; `allotment_pickups` and `group_blocks.shoulder_days_*` exist in no migration/model yet are executed at runtime | **P0** | `01_…` §3.2, `03_…` §2 | D-2, D-13 |
| F-2 | Pickup commits **reservation first, counters later**, in separate transactions → reservation without `picked_qty` increment (Availability double-count window) | **P0** | `create-group-pickup.handler.ts:44,80`; `06_…` §3.1 | D-9 |
| F-3 | **No write path uses locking or version predicates** despite `version` columns → concurrent pickups lose increments | **P0** | `group-block.repository.ts:134,168`; `allotment.repository.ts:153`; `06_…` §2 | D-8 |
| F-4 | Two divergent "available" computations (Availability A1 vs Activities matrix A3) disagree on **every** eligibility filter | **P0** | `availability-source.adapter.ts:49` vs `availability-sales.controller.ts:306-335,459`; `04_…` §3 | D-6 |
| F-5 | GBA never notifies Availability: `IInventoryCommitmentPort` dead; 22 events dropped by consumer | **P1** | `inventory-commitment.port.ts`; `events.consumer.ts:46-57`; `04_…` §1 | D-5, D-7 |
| F-6 | **Cutoff date and rolling release are stored but never enforced**; no scheduler; `ReleaseWindow.isWithinReleaseWindow` uncalled | **P1** | `release-window.value-object.ts:26` (0 call sites); `05_…` B-1/B-2 | D-3 |
| F-6b | **Block lifecycle is unreachable from commands**: `GroupBlock.confirm()/openForPickup()/close()` and `washAllocation()` have zero call sites, so blocks stay `DRAFT` and — because A1 counts only `DEFINITE`/`OPEN_FOR_PICKUP` — contribute **0** to Availability; allotments by contrast are `activate()`d at creation | **P1** | `group-block.aggregate.ts:131,138,144,332`; `create-allotment.handler.ts:49`; `availability-source.adapter.ts:66`; `05_…` §1.1, `04_…` §4 | new decision (lifecycle command set) |
| F-7 | `ConsumeAllotmentVoucherHandler` **fabricates** `reservationId` when absent | **P1** | `consume-allotment-voucher.handler.ts:18` | D-10 |
| F-8 | **4 statements omit `hotel_id` scoping** (cancel-allotment-pickup `UPDATE`s, adapter `linkReservation`) while `x-property-id` override is accepted and dev property check is disabled | **P1** | `cancel-allotment-pickup.handler.ts:66,89-94`; `tenant.interceptor.ts:19-20`; `resource-access.guard.ts:29` | (security review) |
| F-9 | Stop-sale + quota + shoulder/category paths are **double-writes without transactions**; stop-sale flag update uses string-interpolated SQL | **P1** | `set-allotment-quota.handler.ts:32,35`; `allotment.repository.ts:246-261`; `remove-shoulder-days.handler.ts:79-112` | D-8, D-9 |
| F-10 | **0 tests** for 25 commands / 5 queries / 2 controllers / 2 repositories | **P1** | `01_…` §3.6 | test plan |
| F-11 | `remove-shoulder-days` deletes allocations with **no pickup check** and does not recompute block rollups; `remove-room-category` check-then-delete TOCTOU | **P2** | `remove-shoulder-days.handler.ts:79-94`; `remove-room-category.handler.ts:41-55` | D-13 |
| F-12 | `nextBlockCode` counts `WHERE group_booking_id = <bookingCode>` — UUID column vs human code → suffix derived from a non-matching query; second block in one booking targets duplicate `block_code` | **P2** | `group-block.repository.ts:212-216`; `create-group-block.handler.ts:30` | code fix |
| F-13 | Frontend uses `PUT` for block release; controller declares `POST` → 405 | **P2** | `group-allotment.api.ts:146` vs `group-booking.controller.ts:247` | D-12 |
| F-14 | Attrition service registered but never used; threshold stored, never evaluated | **P2** | `group-allotment.module.ts:72,121`; `05_…` B-7 | D-14 |
| F-15 | Legacy GBA tables are schema-only (no TS readers/writers) while `reservations` carries both legacy and new linkage columns with no exclusivity rule | **P2** | `07_…` §1, §4 | D-1 |
| F-16 | FO checkout writes pickup status best-effort (Phase 3 **L-13**, line numbers now `check-out.handler.ts:390-392,479-481`) with no counter changes | **P2** | `07_…` §2 | D-4 |
| F-17 | Guest resolution by `full_name ILIKE` merges same-name guests | **P3** | `prisma-reservation-association.adapter.ts:82-97` | D-11 |
| F-18 | `GET /group-bookings/available-rooms` returns rooms with `room_status='AVAILABLE'` ignoring stay dates — a third, undocumented "availability" | **P3** | `group-booking.controller.ts:95-116` | D-6 (adjacent) |
| F-19 | 22 `IntegrationEvent`s published then discarded (audit-trail churn) | **P3** | `group-allotment.events.ts:3-396`; `events.consumer.ts:57` | D-5 |

---

## 3. What Is Working (stated explicitly)

- Intra-block capacity enforcement (`canPickup`) and intra-allotment quota + stop-sale enforcement on voucher **issue** (`allotment.aggregate.ts:234-252`).
- Repository read scoping by `hotel_id` on the standard paths (`group-block.repository.ts:23,30,36`; `allotment.repository.ts:24,30,36`).
- `group_blocks`/`group_block_daily_allocations`/`allotment_daily_quotas` natural keys exist (`schema.prisma:17130,17215`) — a sound foundation for concurrency work.
- Availability's read path filters GBA state strictly (DEDUCT + booking status + `deleted_at` + HARD_COMMITMENT + validity window) and reports provenance/unresolved facts (`availability-source.adapter.ts:49,93`; `availability-snapshot.service.ts:83`).
- Permission decorators cover every GBA route; block soft-delete preserves history.
- Block save path is transactional (`group-block.repository.ts:103-202`).

---

## 4. Coverage Against the Audit Brief

| Brief requirement | Where delivered |
|---|---|
| Forensic audit only | `01_FORENSIC_AUDIT.md` §1–§2, §5 |
| Current architecture, evidence-based | `02_CURRENT_ARCHITECTURE.md` |
| Data model (new / legacy / drift / mismatch) | `03_DATA_MODEL_AUDIT.md` |
| Availability integration matrix + divergence | `04_AVAILABILITY_INTEGRATION_MATRIX.md` |
| Business rules CURRENT vs TARGET | `05_BUSINESS_RULES_CURRENT_STATE.md` |
| Concurrency & transactions | `06_CONCURRENCY_AND_TRANSACTION_AUDIT.md` |
| Legacy dependency map + Phase 3 L-items (L-13) | `07_LEGACY_DEPENDENCY_MAP.md` |
| Open decisions (no resolution) | `08_OPEN_DECISIONS.md` |
| Gate status | this file §1 |

---

## 5. Recommended Next Actions (decisions, not implementation)

1. **Verify live schema** against `03_DATA_MODEL_AUDIT.md` §2 (read-only `information_schema` query) → resolves D-2/D-13 scope.
2. **Decide D-8 (concurrency) + D-9 (pickup transaction)** — these two define the correctness contract for every subsequent GBA change.
3. **Decide D-6 (authoritative availability number)** — it determines whether Phase 6/9 consumers integrate against A1 or A3.
4. Then sequence D-7 (pickup ↔ assertion engine), D-3 (wash automation), D-5 (event consumers), D-4 (close L-13).
5. Define a test plan for the 25 untested commands before any behavior change (F-10).

---

## 6. Exit Statement

> **Phase 4 (GBA / Allotment Integration) forensic audit is complete.** Nine evidence-based deliverables exist in `docs/availability/phase-4/`; no repository state was modified. The audit gate passes for the *audit* phase. **Implementation of Phase 4 is not authorized by this document** — 15 open decisions (`08_OPEN_DECISIONS.md`, D-1…D-15) and the P0 findings F-1…F-4 (plus P1 F-6b) must be resolved first.

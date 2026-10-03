# XYLO Availability Phase 5 — Forensic Audit

| Field | Value |
|---|---|
| Document | `docs/availability/phase-5/01_FORENSIC_AUDIT.md` (Phase 5 artifact **01** — next number verified against the directory before creation; `phase-5/` did not exist prior to this run) |
| Phase | **Availability Phase 5 — Other Consumers + Frontend Migration** |
| Stage | **Stage 1 of 6 — Forensic Audit (read-only)** |
| Date | 2026-10-03 |
| Workspace | `C:\Users\Pro\Desktop\XYLO` (local workspace authoritative; GitHub out of scope) |
| Mode | **READ-ONLY.** No source file, schema, migration, config, or test was modified. The only artifact created is this document. |
| Authority basis | Phase 1–4 finalized artifacts + current implementation (§15.1) |
| Prohibited | Reopening Phase 1–4 decisions, resolving business-rule ambiguities, implementing fixes, running migrations, dependency upgrades, git mutation |

**Stage-1 output obligation:** audit only. Every ambiguity found is *recorded*, not resolved — resolution belongs to **Stage 2 — Business Rules / Decisions** (§12).

---

## 1. Executive Summary

### 1.1 Headline

The Phase 1–4 Availability authority exists, is fully wired on the backend **write** side, and has **zero consumers on its HTTP read surface**. The frontend, the Activities matrix, the rates/CRS engine, and several dashboard KPIs each still produce an independent "available" number. Worse, under the production dependency binding the authority can currently resolve **neither** a sellable number **nor** a capacity it is willing to assert — so any consumer migrated onto it today would render `0 available / UNKNOWN`.

Four structural facts dominate Phase 5:

1. **The authority's read surface is orphaned.** `GET /api/v1/properties/:propertyId/availability/snapshot` and `.../reconciliation` (`apps/api/src/modules/availability/api/controllers/availability.controller.ts:8,12,20`) have **no caller anywhere** in `apps/web`, `apps/admin`, `apps/mobile`, `gateway/`, or any other backend service (verified by repo-wide grep, §15.3-E1).
2. **The production restriction evaluator is a permanent `UNRESOLVED` stub.** `availability.module.ts:26` binds `RESTRICTION_EVALUATOR → UnresolvedRestrictionAdapter`, which returns `status: 'UNRESOLVED'` with a non-empty `unresolvedSources` on **every** call (`unresolved-restriction.adapter.ts:47-59`). Fail-closed propagation then sets `bookingEligibility: 'UNKNOWN'` and `sellableAvailable: 0` (`availability-snapshot.service.ts` snapshot assembly), and the assertion engine rejects with `UNRESOLVED_CAPACITY` (`availability-assertion.service.ts:881-893`) → `assertReservationCreate` throws `AVAILABILITY_ASSERTION_REJECTED` (`reservation-availability-wiring.ts` result branch). **No test exercises this binding end-to-end** — every suite injects a `RESOLVED` restriction stub (§10 F-01, §12 DS-01).
3. **Five to seven independent "available" numbers are live at once:** the authority snapshot, the Activities matrix (A3, raw SQL), the legacy `availability` counters via `/rates/engine/*`, `GET /rates/availability` (unscoped ad-hoc counts), Front Office / dashboard occupancy KPIs, `GET /analytics/occupancy`, and multiple client-side formulas (§9).
4. **The frontend cannot tell authoritative from derived.** `GET /availability/matrix` emits `label: 'derived view, not sellable availability'` only while `gba.a3.authoritative` is OFF (`availability-sales.controller.ts:612`), and `AvailabilityPage` never surfaces the label; there is no `NEXT_PUBLIC_*` availability flag at all (§5.6).

### 1.2 Scale of the surface

| Measure | Value | Evidence |
|---|---|---|
| Source files mentioning `availability`/`sellable`/`occupancy` (apps + packages + gateway) | **286** | §15.3-E2 |
| `apps/web` files mentioning `availab` | **149** | §15.3-E2 |
| `apps/api/src` files mentioning `availab` | **275** (178 are `*.spec.ts`) | §15.3-E2, E4 |
| `apps/admin` + `apps/mobile` + `packages/ui-*` files mentioning `availab` | **7** (all incidental) | §15.3-E2 |
| Availability-domain HTTP endpoints enumerated | **53** (2 authority, 14 activities/`availability/*`, 7 rates, 21 group/allotment, 9 reservations-availability mutations, 7 front-office, 4 channels) | §7 |
| Frontend consumers of the authority endpoints | **0** | §15.3-E1 |
| Feature flags gating availability behavior | **7 distinct** (`gba.a3.authoritative`, `gba.consumers.cascade` ×3, `gba.wash.schedulerEnabled` ×3, `gba.reconciliation.enabled`, `gba.pickup.twoLayerConsult`, `gba.pickup.canonicalRead`, `gba.pickup.canonicalWrite`), all default **OFF** | §5.6 / §7.4 |
| API spec suites / postgres-gated suites | **178 / 48** (all 48 silently skip without `AVAILABILITY_TEST_DATABASE_URL`) | §15.3-E4, F-17 |
| Frontend test files in `apps/web` | **5** (availability-math coverage: **0**) | §5.8 |
| Findings raised | **27** (4 critical, 9 high, 10 medium, 4 low) | §10 |
| Items requiring a Stage-2 business decision | **12** | §12 |
| Phase 5 blockers identified at Stage 1 | **1 hard** (DS-01), **1 conditional** (DS-02) | §16 |

### 1.3 What this audit does **not** do

No business rule is created, changed, or silently resolved here. Items that look like bugs but encode ratified Phase 1–4 behaviour (e.g. `CHECKED_OUT` not releasing an assertion) are recorded as **compliant** in §13, not reopened.

---

## 2. Current Architecture / Consumer Map

### 2.1 The five worlds that still coexist

```
                       ┌──────────────────────────────────────────────┐
   WRITE PATH (Phase 3) │  RESERVATIONS / FRONT OFFICE                 │
                        │  reservation.repository.ts                   │
                        │    ├─ assertCreateAvailability ──────────┐   │
                        │    ├─ replaceReservationAssertion ───────┤   │
                        │    ├─ applyStatusAvailability ───────────┤   │
                        │    └─ crs.modifyReservation  (LEGACY ◄───┼─── dual write, :625
                        │  front-office: check-out ✓ authority     │   │
                        │                upgrade ✗ legacy gate :69 │   │
                        └───────────────────────────┬──────────────┘   │
                                                    ▼                  │
   ┌────────────────────────────────────────────────────────────────┐  │
   │  AUTHORITY (Phases 1-2 + 3 + 4)                               │  │
   │  availability-snapshot.service.ts ── snapshot-calculator.ts    │  │
   │  availability-source.adapter.ts    ── reservation-consumption  │  │
   │  unresolved-restriction.adapter.ts ◄── ALWAYS UNRESOLVED (F-01)│◄─┘
   │  availability-assertion.service.ts (RESERVATION_AVAILABILITY_PORT)
   │  availability-reconciliation.service.ts                        │
   │  HTTP: GET properties/:pid/availability/snapshot|reconciliation│
   │        ── 0 consumers (F-02)                                   │
   └────────────┬──────────────────────────────┬────────────────────┘
                │ read (flag-gated)            │ read (flag-gated)
                ▼                              ▼
   ┌─────────────────────────┐   ┌──────────────────────────────────────┐
   │ A3 ACTIVITIES MATRIX    │   │ GBA pickup two-layer consult         │
   │ availability-sales      │   │ pickup-availability.snapshot-consult │
   │ controller.ts (root @   │   │ (gba.pickup.twoLayerConsult OFF)     │
   │ Controller, Property-   │   └──────────────────────────────────────┘
   │ Scope(false))           │
   │  flag gba.a3.authoritative
   │  OFF → raw SQL engine   │──── RESTRICTION WRITES (raw SQL, 8 tables)
   │  ON  → snapshot numbers │     bulk-update :676/686/697
   │       + raw restrictions│     interval-update :825-885
   └──────────┬──────────────┘
              │ GET /availability/matrix, /room-types,
              │ /restriction-rows, /bulk-update
              ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ LEGACY / SHADOW READS                                          │
   │  rates-inventory: inventory.domain-service.ts (availability    │
   │     table counters) ← crs-engine :345,:391,:531-542,:589       │
   │  GET /rates/availability → rates-inventory.service.ts:34-40    │
   │     (rooms.count() with NO hotel_id — F-06)                    │
   │  GET /analytics/occupancy, /front-office/dashboard, widget data│
   └────────────────────────────────────────────────────────────────┘
              │
              ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ FRONTEND (apps/web) — 149 files                                 │
   │  /reservations/availability → AvailabilityPage (A3 payload +   │
   │       2 client-side occupancy formulas)                         │
   │  /rates-inventory "Availability Snapshot" tab (legacy counts)   │
   │  quick-book AvailableRatesMatrix (always renders 0 — F-10)      │
   │  use-crs-book quote gate (write-blocking on legacy — F-07)      │
   │  dashboard/FO KPI occupancy (4 independent formulas)            │
   │  group/allotment views (recompute quota−picked−released)        │
   │  admin + mobile: zero availability surface (greenfield)         │
   └────────────────────────────────────────────────────────────────┘
```

### 2.2 Direction-of-flow summary

| Flow | Exists? | Evidence |
|---|---|---|
| Reservations/FO → authority (assert/release/replace) | **YES** | `reservation.repository.ts:469,621,732,800,826,857,1000`; `check-out.handler.ts:194-230` |
| Authority → legacy `availability` counters | **NO** | no authority file references `inventory.domain-service` |
| Legacy counters ← CRS engine (independent) | **YES** | `crs-engine.service.ts:345,391,531-542,589` |
| Authority → A3 matrix | **YES, flag-gated** | `availability-sales.controller.ts:275,461-514` |
| Authority → frontend | **NO** | §15.3-E1 (0 hits) |
| GBA → authority (write/notify) | **NO** (read-time only) | `IInventoryCommitmentPort` zero implementations; A1 reads GBA at query time |
| Authority → channels (outbound push) | **NO** | `channels.service.ts:162-183` pushes a caller-supplied number |
| Restriction writers → authority | **NO** | authority adapter declares those stores `UNRESOLVED`; writers are raw SQL in A3 |

---

## 3. Complete Consumer Inventory

Classification codes: **AUTH** = consumes Phase 1–4 authority; **LEG** = legacy/shadow computation; **MIX** = both; **NONE** = no availability dependency; **GREEN** = no surface yet.

| # | Consumer | Location | Kind | Evidence |
|---|---|---|---|---|
| C-01 | Availability snapshot HTTP | `apps/api/src/modules/availability/api/controllers/availability.controller.ts:8,12` | AUTH | read of `AvailabilitySnapshotService` |
| C-02 | Availability reconciliation HTTP | same file `:20` | AUTH | compares authority vs legacy projection |
| C-03 | Assertion engine (write authority) | `availability/application/services/availability-assertion.service.ts` | AUTH | port bound `availability.module.ts:25` |
| C-04 | Reservations repository (create/update/cancel/no-show/reinstate/batch/waitlist) | `reservations/infrastructure/repositories/reservation.repository.ts:469,621,646,732,800,826,851,941,1208,1237` | MIX | authority + legacy `crs.modifyReservation` `:625` |
| C-05 | Front Office check-out | `front-office/application/commands/check-out/check-out.handler.ts:194-230` | AUTH | overstay/early-departure `replaceInTransaction` |
| C-06 | Front Office room upgrade | `front-office/application/commands/upgrade-room/upgrade-room.handler.ts:69,85` | MIX | **legacy read `:69` + authority write `:85`** |
| C-07 | Front Office check-in (room options) | `front-office/check-in/application/queries/get-check-in-room-options.handler.ts:50-113` | NONE (room-level, documented) | `:15-24` states client never filters |
| C-08 | Front Office room assignment / transfer | `front-office/domain/services/room-assignment.service.ts:119-148`; `transfer-room.handler.ts` | NONE (room-level) | no rate-type inventory consult |
| C-09 | CRS engine (quote/book/modify/release) | `rates-inventory/crs-engine.service.ts:97,345,391,504-542,586-595` | LEG | legacy `availability` counters |
| C-10 | Rates HTTP (`/rates/availability`, `/rates/engine/*`) | `rates-inventory/rates-inventory.controller.ts:34-186` | LEG | `inventory.domain-service.ts` + ad-hoc service |
| C-11 | Activities availability matrix (A3) | `activities/availability-sales.controller.ts:265-614` | MIX | flag `gba.a3.authoritative` `:275` |
| C-12 | Activities restriction writes | same file `:651,773` | LEG (bypasses authority) | raw SQL into 8 restriction tables |
| C-13 | Activities reference reads (`/availability/room-types`, `/restriction-rows`, logs) | same file `:41,719,752,900` | LEG | raw SQL / `room_inventory` |
| C-14 | GBA pickup two-layer consult | `group-allotment/infrastructure/adapters/pickup-availability.snapshot-consult.ts:25,66,79,139-146` | AUTH (flag-gated) | reads `availability_assertion_balances` |
| C-15 | GBA pickup guards / grid / release | `group-allotment/domain/*`, handlers | AUTH (GBA's own ledger) | `canPickup`, `quota − picked − released` |
| C-16 | GBA pickup canonical read/write switches | `allotment.controller.ts:313`; `reservation-pickup-cascade.service.ts:102` | MIX (staged) | flags default OFF |
| C-17 | Event cascade consumers | `shared/events.consumer.ts:150,171,188` | AUTH-adjacent | `gba.consumers.cascade` |
| C-18 | Reconciliation detectors | `group-allotment/infrastructure/reconciliation/gba-reconciliation.service.ts:133` | AUTH-adjacent | `gba.reconciliation.enabled` |
| C-19 | Wash scheduler | `group-allotment/infrastructure/schedulers/wash-scheduler.service.ts:33` | BLOCKED | Deviations A+B |
| C-20 | Channels push/pull | `channels/channels.service.ts:162-195`; `channels.controller.ts:33-48` | LEG | caller-supplied `available` |
| C-21 | OTA webhook ingress | `channels/webhook-ingress/webhook.service.ts:71,95-104` | AUTH (indirect, via reservations.create) | overbooking branch likely dead (F-27) |
| C-22 | Analytics occupancy | `reporting-analytics/reporting-analytics.service.ts:162-205` | LEG | raw occupancy SQL |
| C-23 | Corporate board occupancy | `reporting-analytics/corporate-board.service.ts:21,104,120` | LEG | derived KPI |
| C-24 | Front Office dashboard KPIs | `front-office/front-office.service.ts:87,141` | LEG | vacant/available counts |
| C-25 | Command-center widgets | `command-center/**` (`occupancy-gauge` widget) | LEG | opaque widget contract |
| C-26 | Web availability page | `apps/web/features/reservations/workspace/pages/AvailabilityPage.tsx` | LEG (A3) | `:309,545,788,927` |
| C-27 | Web quick-book rate matrix | `apps/web/features/reservations/workspace/components/quick-book/AvailableRatesMatrix.tsx` | LEG + broken | cells hardcoded `available: 0` |
| C-28 | Web CRS book hook | `apps/web/features/reservations/hooks/use-crs-book.ts:87,129` | LEG (write gate) | quote availability blocks booking |
| C-29 | Web rates-inventory tab | `apps/web/app/(dashboard)/rates-inventory/page.tsx:143` | LEG | labelled "Availability Snapshot" |
| C-30 | Web dashboard/FO occupancy KPIs | `apps/web/app/(dashboard)/layout.tsx:108`; `features/front-office/components/dashboard/KpiHeader.tsx:30-37` | LEG | independent formulas |
| C-31 | Web command-center/reporting charts | `apps/web/components/command-center/ChartsRow.tsx:61-101`; `app/(dashboard)/reporting-analytics/page.tsx:14` | LEG / mock | `78.4%` hardcoded |
| C-32 | Web group/allotment views | `apps/web/features/group-allotment/views/*.tsx` | LEG (client recompute) | `quota − picked − released` etc. |
| C-33 | Web room grid / room maps | `packages/ui-web/src/components/domain/RoomGrid.tsx:118-131`; `apps/web/components/AnimatedRoomRack.tsx` | LEG | status counting |
| C-34 | Web stores | `apps/web/store/reservationStore.ts:187`, `analyticsStore.ts:34,51-56`, `hotelStore.ts:123,148`, `settingsStore.ts:20` | LEG / dead | `totalRooms \|\| 200` |
| C-35 | Admin properties page | `apps/admin/app/(corporate)/properties/page.tsx:7-10,43-50` | GREEN/mock | hardcoded rooms/occupancy |
| C-36 | Mobile app | `apps/mobile/**` | GREEN | zero availability references |
| C-37 | Gateway (Kong/nginx/WAF) | `gateway/**` | NONE | catch-all, no availability routing |
| C-38 | Stock/warehouse inventory (`inventory/availability`, `warehouses/:id/capacity`) | `apps/api/src/modules/inventory/**` | OUT OF SCOPE | goods, not rooms |
| C-39 | Banquet/venue capacity (`activities`, `alternate-spaces`, `function-diary`) | `apps/api/src/modules/activities/**` | OUT OF SCOPE | venue capacity |
| C-40 | Legacy GBA tables (`allotment`, `block_pickup`, `orms_pickup`, `reservation_block`, …) | `packages/db/schema.prisma` | orphaned | schema-only, no TS readers/writers (Phase 4 `07_LEGACY_DEPENDENCY_MAP.md` §1 re-verified) |

---

## 4. Legacy Availability Logic Inventory

### 4.1 The single legacy counter writer

`apps/api/src/modules/rates-inventory/domain/services/inventory.domain-service.ts` (283 lines). Its header (`:20-33`) claims it is the "single writer for the `availability` counter"; the file is in fact the **only** writer, but it is not the only availability computation in the system.

| Member | Lines | SQL on `availability` | Status | Verified callers |
|---|---|---|---|---|
| `checkAvailability` | 56-118 | `FROM availability` `:74` + `rooms.count()` `:65-67` + reservation counts `:79-91` + fallback math `:109-116` | LIVE | `crs-engine.service.ts:97,251,504` |
| `isAvailable` | 121-127 | `SELECT available` `:123`; **absent row ⇒ `true`** `:126` | LIVE | `front-office/.../upgrade-room.handler.ts:69` |
| `assertAvailability` | 134-156 | `SELECT … FOR UPDATE` `:145` | LIVE | `crs-engine.service.ts:345` |
| `reserve` | 159-173 | `INSERT … DO UPDATE` `:164` | LIVE | `crs-engine.service.ts:391,531,542` |
| `release` | 176-186 | `UPDATE … reserved > 0` `:181` | LIVE | `crs-engine.service.ts:533,535,539,589` |
| `dateList`, `reserveRooms`, `releaseRooms` | 51-53, 189-202 | — | dead | zero callers |
| `blockAvailability`, `consumePickup`, `releaseUnsold` | 208-282 | read/update `availability` `:220,:227,:247,:256,:275` | dead | zero callers (legacy equivalents of Phase 4 wash/pickup) |

**Related:** `reservation.repository.ts:15` imports and `:89` injects `InventoryDomainService` — **never referenced again** in the 1590-line file (dead DI).

### 4.2 Legacy `availability` table access (schema `packages/db/schema.prisma:433-453`)

| Accessor | file:line | Op |
|---|---|---|
| `inventory.domain-service.ts` | `:74,:123,:145,:164,:181,:220,:227,:247,:256,:275` | read ×3, write ×7 |
| Authority reconciliation (permitted) | `availability/infrastructure/reconciliation/availability-reconciliation.service.ts:17` | read-only comparison |
| Grep gates proving no other access | `group-allotment/__tests__/t65-legacy-write-absence.spec.ts:40,46,118,141`; `t69-secondary-column-non-authority.postgres.spec.ts` | static guards |

### 4.3 Raw SQL inventory outside the authority

| Class | file:line | Statement purpose |
|---|---|---|
| Physical/room inventory | `activities/availability-sales.controller.ts:62,249-252,284-297,369-381` | `rooms`, `room_inventory` (**`room_inventory` has no writer anywhere — F-11**) |
| Restriction reads | `crs-engine.service.ts:152,372`; `availability-sales.controller.ts:384-405,912-948` | `rate_restrictions`, `close_to_arrival/departure`, `zero_sell_limits`, sell-limit/LOS tables |
| Restriction writes | `availability-sales.controller.ts:676-707,825-885` | 8 tables incl. a **second** restrictions table (`restrictions`, `rate_code='CUTOFF'` `:878`) |
| GBA quotas | `availability-sales.controller.ts:342-366` | `group_block_daily_allocations`, `allotment_daily_quotas` (flag-gated numbers) |
| Channel mirrors | `channels/channels.service.ts:88-99,126-179`; `webhook-ingress/webhook.service.ts:95-99` | `channel_availability_log` writes; `channel_availability` read `:187-195`; **no writer for `channel_availability`** |
| Room status moves | `front-office/.../execute-scheduled-room-move.handler.ts:57,79,85` | room-level, not inventory |
| Orphan adapters | `reservations/infrastructure/adapters/prisma-inventory-reservation.adapter.ts:29-35,84-95`; `prisma-rates.adapter.ts:32-39` | never DI-registered (dead) |

### 4.4 Ad-hoc availability computations still reachable

| # | Computation | file:line | Formula | Endpoint |
|---|---|---|---|---|
| 1 | Rates availability KPI | `rates-inventory/rates-inventory.service.ts:34-40` | `rooms.count() − reservations.count(...)` → `available`, `occupancyPct` (**no `hotel_id` filter; div-by-zero if 0 rooms**) | `GET /rates/availability` |
| 2 | Legacy mixed check | `inventory.domain-service.ts:96-117` | `max(0, available − oversell)` or `roomCount − activeReservations` | `GET /rates/engine/availability` |
| 3 | A3 matrix (flag OFF) | `availability-sales.controller.ts:548-581` | `max(0, physical − ooo − reserved − groupCommit − allotCommit)` + proportional distribution | `GET /availability/matrix` |
| 4 | A3 matrix (flag ON) | `availability-sales.controller.ts:532-546` | `available = hasZeroSell ? 0 : fact.sellableAvailable` | same |
| 5 | Front Office room grid | `front-office/front-office.query.handlers.ts:174-188` | ad-hoc `rooms.findMany` | `GET /front-office/room-grid` |
| 6 | Analytics occupancy | `reporting-analytics/reporting-analytics.service.ts:162-209` | occupied/total | `GET /analytics/occupancy` |
| 7 | Client-side formulas | `apps/web/**` (§5.3) | several | n/a |

---

## 5. Frontend Migration Inventory

### 5.1 Display surfaces

| # | Surface | File:line | Data source | Compliance |
|---|---|---|---|---|
| F-01 | Availability grid page | `apps/web/features/reservations/workspace/pages/AvailabilityPage.tsx` (route `apps/web/app/(dashboard)/reservations/availability/page.tsx:4`) | `GET /availability/matrix` via `reservation.api.ts:232` | LEG/A3 |
| F-02 | Same page — occupancy row | `AvailabilityPage.tsx:870-871` | client: `round((physical − available)/physical)` | **independent** |
| F-03 | Same page — header row labelled "avail" | `AvailabilityPage.tsx:1006-1015` | **sums `physicalInventory`**, variable `totalAvail` | **independent + mislabelled** |
| F-04 | Rates-inventory "Availability Snapshot" tab | `apps/web/app/(dashboard)/rates-inventory/page.tsx:312-351` (heading `:314`, call `:143`) | `GET /rates/availability` | **LEG, name collides with authority** |
| F-05 | Global dashboard occupancy chip | `apps/web/app/(dashboard)/layout.tsx:108-115` | `/front-office/dashboard` + `analyticsStore.dashboard.occupancy.today` fallback | LEG |
| F-06 | FO dashboard KPI | `apps/web/features/front-office/components/dashboard/KpiHeader.tsx:30-37` | `/front-office/dashboard` | LEG |
| F-07 | Room grid occupancy | `packages/ui-web/src/components/domain/RoomGrid.tsx:118-131,173-174` | client status counting | independent |
| F-08 | Command-center occupancy trend/gauge | `apps/web/components/command-center/ChartsRow.tsx:61-101`; `KpiRow.tsx:13-20` | `/analytics/occupancy`, widget data | LEG |
| F-09 | Reporting page | `apps/web/app/(dashboard)/reporting-analytics/page.tsx:14` | **hardcoded `78.4%`** | mock |
| F-10 | Admin properties | `apps/admin/app/(corporate)/properties/page.tsx:7-10,43-50` | hardcoded mocks | mock |
| F-11 | Group bookings list/detail | `apps/web/features/group-allotment/views/GroupBookingsListView.tsx:99-101,154`; `GroupBookingDetailView.tsx:96-114,418,815-819` | client pickup/wash math | independent |
| F-12 | Allotment detail | `.../AllotmentDetailView.tsx:687,757,764` | `quota − picked − released`; hardcoded `$420` | independent |
| F-13 | Quick-book rate matrix | `.../quick-book/AvailableRatesMatrix.tsx:61-89` (render `:270,294,300,306`) | `useAvailabilityMatrix` **but cells never mapped → always 0** | **broken** |
| F-14 | Check-in room options | `.../front-office/components/dialogs/CheckInDialog.tsx:611,641-642` | `GET /check-in/.../room-options` | compliant-by-design (room-level) |
| F-15 | Day-use available rooms | `.../front-office/hooks/useDayUseAvailableRooms.ts:11`; `utils/day-use.helpers.ts:5-23` | room status mirror | LEG (room-level) |
| F-16 | Transfer/upgrade/room helpers | `.../front-office/utils/{transfer,upgrade,room}.helpers.ts` | mirrored eligibility statuses | LEG (room-level) |
| F-17 | Dead component with live calls | `apps/web/components/AvailabilitySales.tsx:55-56` (never imported) | `/availability/room-types`, `/availability/banquet-venues` | dead |

### 5.2 API clients and hooks (availability-bearing)

| File:line | Method | Endpoint | Backend exists? |
|---|---|---|---|
| `apps/web/features/reservations/api/reservation.api.ts:214` | `getRoomTypes` | `GET /availability/room-types` | yes (`availability-sales.controller.ts:41`) |
| `:232` | `getAvailabilityMatrix` | `GET /availability/matrix` | yes `:265` |
| `:247` | `bulkUpdateAvailability` | `POST /availability/bulk-update` | yes `:651` |
| `:341/:350/:363/:370` | market/source/rate-lookup/bed-features | `GET /availability/*` | yes `:152/:164/:176/:616` |
| `:397` | `getRestrictionRows` | `GET /availability/restriction-rows` | yes `:900` |
| **`:405`, `:420`** | `getMatrixLogs` / `createMatrixLog` | `GET/POST /availability/matrix/logs` | **NO — backend is `/availability/logs` `:719/:752` ⇒ 404** |
| **`:431`** | `intervalUpdate` | `POST /activities/availability/interval-update` | **NO — backend is `/availability/interval-update` `:773` ⇒ 404; payload also differs (`ratePlanCodes`/`startDate`/`endDate` vs `ratePlans`/`dateRange{start,end}`)** |
| `apps/web/features/reservations/hooks/use-reservation-queries.ts:68-83` | `useRoomTypes`, `useAvailabilityMatrix` | above (`staleTime 60000`, `enabled: arrivalDate`) | — |
| `apps/web/store/reservationStore.ts:187` | `fetchRoomTypes` | `GET /availability/room-types` | **zero callers (dead)** |
| `apps/web/app/(dashboard)/rates-inventory/page.tsx:143,155` | — | `GET /rates/availability`, `/rates/engine/rate` | yes |
| `apps/web/features/reservations/hooks/use-crs-book.ts:87,129` | quote + book | `GET /rates/engine/quote`, `POST /rates/engine/book` | yes (legacy write gate) |
| `apps/web/features/front-office/api/front-office.api.ts:179-194` | quote | `GET /rates/engine/quote` | yes |
| `apps/web/features/group-allotment/api/group-allotment.api.ts:88,131,142,320,343,374` | grid/pickups/release | `/group-bookings/*`, `/allotments/*` | yes (GBA) |
| `apps/web/store/analyticsStore.ts:51-56` | `fetchOccupancy` | `GET /analytics/occupancy` | response **never stored (dead)** |
| `apps/web/lib/inventory/api/endpoints.ts:309` | — | `/inventory/advanced-reservations/availability` | **different bounded domain (goods)** |

Callers of the broken routes: `AvailabilityPage.tsx:309` (`intervalUpdate`), `:545` (`getMatrixLogs`). Both silently fail today.

### 5.3 Independent client-side availability math (must be removed or re-pointed)

| File:line | Formula |
|---|---|
| `AvailabilityPage.tsx:851-876` | `totalAvailable += cell.available`; `occupancy = (physical − available)/physical` |
| `AvailabilityPage.tsx:1006-1009` | `totalAvail = Σ physicalInventory` (mislabelled) |
| `apps/web/app/(dashboard)/layout.tsx:108` | `round(max(0, occupied/total)*100)` |
| `KpiHeader.tsx:30-37` | `pct(occupied, total)` |
| `RoomGrid.tsx:130` | `round((occupied/total)*100)` |
| `AnimatedRoomRack.tsx:25-41,122-134` | status → counters (dead component) |
| `GroupBookingsListView.tsx:100-101` | `round((picked/blocked)*100)` |
| `GroupBookingDetailView.tsx:99-101,418,819` | `round(nights/n)`, pickup %, wash % |
| `AllotmentDetailView.tsx:757` | `quota − picked − released` |
| `apps/web/store/hotelStore.ts:123,148` | `totalRooms = h.total_rooms \|\| 200` (**hardcoded 200**) |
| `AvailableRatesMatrix.tsx:61-89` | cells hardwired `available: 0` |
| `apps/web/components/AvailabilitySales.tsx:73-74,211` | `totalRooms = Σ physicalRooms`; occupancy (dead) |

### 5.4 Shadow/duplicated state

| Store/state | File:line | Content | Verdict |
|---|---|---|---|
| `reservationStore.roomTypeList` | `:114-115,172,187-191` | room types + `availableRooms` from `/availability/room-types` | shadow, writer dead |
| `analyticsStore` | `:6,23,34,51-56` | `dashboard.occupancy.today` feeds layout fallback | shadow; `fetchOccupancy` never persists |
| `hotelStore` / `settingsStore` | `hotelStore.ts:36,123,148`; `settingsStore.ts:20` | `totalRooms` | shadow + fake default |
| `commandCenterStore` | widget registry | occupancy gauge | shadow |
| `reservation-ui-store.ts:13,28` | `'availability'` tab key | navigation only (OK) |

**Confirmed negatives:** no Zustand/React-Query store anywhere holds snapshot data; no `NEXT_PUBLIC_*` availability flag exists.

### 5.5 Dead / broken frontend artifacts

`apps/web/components/AvailabilitySales.tsx` (never imported), `apps/web/components/AnimatedRoomRack.tsx` (never imported), `apps/web/features/operasales/**` (orphan folder), `apps/web/features/reservations/booking/types.ts:16,27` (duplicate unused rate-grid types), `reservationStore.fetchRoomTypes`, `analyticsStore.fetchOccupancy`, `reservation.api.ts:362-407` (`getRateLookup`, `createMatrixLog`, `createIntervalUpdate` — zero callers), duplicate routes `/allotments` vs `/reservations/allotments` and `/group-blocks` vs `/reservations/group-blocks`.

### 5.6 Feature flags visible to the frontend

**None.** Frontend env surface is `NEXT_PUBLIC_API_URL` / `NEXT_PUBLIC_WS_URL` only (`apps/web/next.config.js:35-36`, `services/api.ts:1`, `lib/api/client.ts:24`). The only authority-related signal that reaches UI is `washMetadataAuthoritative` (`group-allotment/lib/wash-authority.ts:5-28`, fed from `get-allotments.handler.ts:34` / `get-group-booking-detail.handler.ts:47`). The `gba.a3.authoritative` state and the matrix `label` field are invisible on `AvailabilityPage` (`AvailableRatesMatrix.tsx:140-142` is the only place `label` is rendered).

### 5.7 Admin / mobile

`apps/admin`: zero availability API calls; reservations dashboard is mock data (`ReservationsDashboard.tsx:6,16`); properties page hardcodes rooms/occupancy; 4 dead inventory nav links (`app/(corporate)/inventory/page.tsx:50-55`). `apps/mobile`: zero availability surface (only auth client). Both are **greenfield**, not migration.

### 5.8 Frontend tests

5 test files total (`wash-authority.test.ts`, `group-allotment.api.test.ts`, `reservation-actions.test.ts`, `idempotency.test.ts`, `client-idempotency.test.ts`). **No test covers availability math, matrix mapping, snapshot consumption, or `/rates/availability`.** No tests exist in `apps/admin`, `apps/mobile`, or `packages/ui-*`.

---

## 6. Backend Consumer Inventory

### 6.1 Reservations write paths (authority wiring status)

| Operation | Location | Authority call | Legacy call | Verdict |
|---|---|---|---|---|
| create | `reservation.repository.ts:469` → `assertCreateAvailability :552` | `assertInTransaction` | — | AUTH |
| update (stay/type) | `:621-623` + `:625` | `replaceReservationAssertion` | **`crs.modifyReservation`** | **MIX — dual write** |
| status change | `:646-661` → `applyStatusAvailability :1000-1055` | release/assert per status | — | AUTH |
| cancel / cancelIsolated | `:732,:748-758` → `releaseLifecycleAssertion :800` | release | — | AUTH |
| no-show | `:826-848` | release `slice(1)` | — | AUTH |
| reinstate | `:851-869` | assert | — | AUTH |
| batch status | `:926-990` | `applyStatusAvailability` | — | AUTH |
| **delete** | `:889-924` | **none** (terminal-only guard, Decision 12) | — | see §12 DS-11 note |
| waitlist join/promote | `:1165-1259` | release / assert | — | AUTH |
| change-rate handler | `change-rate.handler.ts:30` (`repo.update` with no write identity) | runs with `randomUUID()` operation keys, `actorId='system'` | — | **identity gap (F-16)** |
| create identity | `reservations.service.ts:89-93`; `create-reservation.handler.ts:25,30` | `idempotencyKey` undefined; `hotelId \|\| 'default'` fallback (authority rejects `'default'`) | — | **identity gap (F-16)** |

### 6.2 Front Office

| Operation | Availability decision | Verdict |
|---|---|---|
| check-in | room-level only (`get-check-in-room-options.handler.ts:50-113`); inventory consumed at reserve | correct |
| check-out | overstay/early-departure `findCurrentInTransaction` + `replaceInTransaction` (`check-out.handler.ts:194-230`); **zero GBA writes** (`:402-405`) | AUTH, compliant |
| upgrade | **legacy `inventoryDomain.isAvailable` gate `:69` + authority `replaceRoomTypeAssertion` `:85`** | **MIX (F-05)** |
| transfer (same type) | no availability op (`transfer-room.handler.ts` has no availability reference) | correct |
| extend | routes to `repo.update` → dual write (§6.1) | MIX |
| walk-in / waitlist promote / no-show / reinstate | authority via repository | AUTH |

### 6.3 Group / Allotment

Pickup creation validates intra-ledger capacity (`group-block.aggregate.ts:234-242`, `allotment.aggregate.ts:242-252` incl. stop sale) and never consults snapshot/assertion unless `gba.pickup.twoLayerConsult` is ON (`pickup-availability.snapshot-consult.ts:25,66` — default OFF, reads `availability_assertion_balances` fail-closed `:148-151`). Pickup-created reservations are inserted by raw SQL (`prisma-reservation-association.adapter.ts:100-120`) **without** registering an assertion (Phase 4 A5 finding, re-verified).

### 6.4 Other backend consumers

- **Activities A3** — second complete availability engine (§4.3/4.4) + restriction writer.
- **Channels** — `pushAvailabilityToChannels(data.available)` (`channels.service.ts:162-183`) performs **no computation**; `channel_availability` is read but never written.
- **Events** — `events.consumer.ts:140-202` handles reservation cascade + GBA invalidation; the invalidation deletes `availability:${hotelId}:*` although nothing in the authority writes that key (**no-op, F-14**).
- **Analytics / command-center / corporate board** — independent occupancy KPIs.
- **Dead DI / orphan adapters** — `reservation.repository.ts:15,89`; `PrismaInventoryReservationAdapter`; `PrismaRatesAdapter`; `IInventoryCommitmentPort` (zero implementations); `AttritionCalculationService` (never injected).

---

## 7. API Contract / Integration Inventory

### 7.1 Authority surface

| Method | Path (prefix `api/v1`) | file:line | Response |
|---|---|---|---|
| GET | `/properties/:propertyId/availability/snapshot?roomType&arrivalDate&departureDate&channelCode&rateCode` | `availability.controller.ts:12` | `{hotelId,roomType,arrivalDate,departureDate,calculatedAt,dates[]}`; per date `physicalCount, oooCount, oosCount, unavailablePhysicalCount, reservationConsumption(+Outcome), gbaHeld/Picked/Released/Remaining, allotmentQuota/Picked/Released/Remaining, sellLimit, overbookingAllowance/Used, consumption, physicalAvailable, sellableCapacity, sellableAvailable, restrictionOutcome, unresolvedSources, bookingEligibility, sourceReferences, calculatedAt, freshness` |
| GET | `/properties/:propertyId/availability/reconciliation?roomType&arrivalDate&departureDate` | `:20` | authority vs `availability.available` legacy projection + deltas |

Auth: explicit 403 without actor (`:15,:23`); `snapshots.authorize` re-checks property membership + permission codes `['availability:read','reservations:read','reservations:occupancy:read','*','platform:admin:full']` and rejects `hotelId === 'default'`.

### 7.2 Classification of every availability-adjacent endpoint

| Endpoint | Class | Notes |
|---|---|---|
| `GET /properties/:pid/availability/{snapshot,reconciliation}` | **AUTHORITY** | 0 consumers |
| `GET /availability/matrix` | **SHADOW (dual-mode)** | flag `gba.a3.authoritative` (`:275`); OFF → raw SQL + `label` `:612`; ON → snapshot numbers `:461-514` **but restrictions stay raw `:384-405`** |
| `GET /availability/room-types` | LEGACY | `room_inventory` reader with no writer; `reservation_name` counts |
| `GET /availability/restriction-rows` | LEGACY | raw restriction reads |
| `POST /availability/bulk-update`, `POST /availability/interval-update` | LEGACY (writes) | raw SQL into restriction tables; **bypass authority** |
| `GET/POST /availability/logs` | LEGACY audit | frontend calls wrong path (404) |
| `GET /availability/{market-codes,source-codes,rate-lookup,bed-features,banquet-venues}` | reference data | benign |
| `GET /tax-rates` (declared twice) | **COLLISION** | `banquet-refs.controller.ts:45` registered **before** `availability-sales.controller.ts:131` (`activities.module.ts:24-25`) ⇒ availability implementation unreachable |
| `GET /rates/availability` | LEGACY, **unscoped** | `rates-inventory.service.ts:34-40` |
| `GET /rates/engine/{availability,quote,restrictions,rate}` | LEGACY | legacy counters / `rate_restrictions` |
| `POST /rates/engine/{book,modify,release}` | LEGACY writes | `assertAvailability`/`reserve`/`release` |
| `/allotments/*`, `/group-bookings/*` | GBA authority | staged canonical read/write flags |
| `POST /reservations*` availability mutations | AUTHORITY | via repository |
| `POST /front-office/{check-out,walk-in,transfer,:id/upgrade,:id/extend}` | AUTHORITY / MIX | see §6.2 |
| `POST /channels/availability/push` | LEGACY (caller-supplied number) | no in-repo caller |
| `GET /analytics/occupancy` | LEGACY | raw occupancy |
| `GET /front-office/{dashboard,room-grid}` | LEGACY ad-hoc | |
| `inventory/availability`, `warehouses/:id/capacity` | OUT OF SCOPE (goods) | |

### 7.3 Contract / DTO gaps

- **No shared TypeScript type exists** for the snapshot/reconciliation payload; authority consumers would re-declare it (frontend hand-rolls `MatrixCell`, `RestrictionData`, `RateQuote`, `DashboardKPIs` etc.).
- No `class-validator` DTO classes on the authority endpoints; A3 matrix/log/interval handlers accept `@Body() body: any`.
- **Payload mismatches:** `intervalUpdate` sends `ratePlanCodes`/`startDate`/`endDate`; backend expects `ratePlans`/`dateRange{start,end}` (`availability-sales.controller.ts:776-796`).
- **Query mismatches:** frontend `pageSize` vs backend logs paging params.
- **Parameter divergence:** authority controller authorizes `@Param('propertyId')` while `core/guards/property-scope.guard.ts:28-45` may scope from `x-property-id` or `user.tenantId/hotelId` (super-admin prefers header) — guard can validate property X while data is read for property Y (F-19).
- **Guard opt-out:** `AvailabilitySalesController` and `BanquetRefsController` are `@PropertyScope(false)` (`:30-31` / `:9-10`) → property-scope guard bypassed; they rely on `requireHotelId()` from request context instead.

### 7.4 Feature-flag register (all read via `ConfigService.getFeatureFlag`, default **OFF**; env `FEATURE_*`)

| Flag | Read sites | Gates | Rollout state (Phase 4 `17_PHASE4_READINESS_REVIEW.md` §6) |
|---|---|---|---|
| `gba.a3.authoritative` | `availability-sales.controller.ts:275` | matrix numbers from snapshot vs raw SQL | "per its own gate" |
| `gba.consumers.cascade` | `events.consumer.ts:150,171,188` | reservation→pickup cascade + GBA invalidation | **MUST be ON at deploy** — but **not present in `.env`, `.env.example`, or `compose.yaml`** |
| `gba.wash.schedulerEnabled` | `wash-scheduler.service.ts:33`; `get-allotments.handler.ts:34`; `get-group-booking-detail.handler.ts:47` | hourly wash + `washMetadataAuthoritative` | **BLOCKED on Deviations A+B** |
| `gba.reconciliation.enabled` | `gba-reconciliation.service.ts:133` | hourly detectors | ready for soak |
| `gba.pickup.twoLayerConsult` | `pickup-availability.snapshot-consult.ts:66` | read-time consult vs quota-only guard | default OFF |
| `gba.pickup.canonicalRead` | `allotment.controller.ts:313` | pickup read source | enable **first** |
| `gba.pickup.canonicalWrite` | `reservation-pickup-cascade.service.ts:102` | legacy ledger freeze | enable **second** |

**Second flag system exists:** `apps/api/src/platform/configuration/feature-flag.service.ts` (DB-backed, HTTP CRUD at `/api/v1/platform/feature-flags`, tenant/property/role/percentage conditions). **No availability code uses it** and no availability flags are seeded (F-18).

**Test gate:** `apps/api/src/modules/reservations/infrastructure/__tests__/availability-postgres.harness.ts:8-9` — `describePostgres = databaseUrl ? describe : describe.skip`; **48 of 178 API suites** are gated this way and silently skip when `AVAILABILITY_TEST_DATABASE_URL` is unset (no jest `setupFiles`; not in `.env.example`).

### 7.5 Gateway

`gateway/api-gateway/kong.yml` — single catch-all service (`path: /api/v1`), `routes/` directory **empty**; `gateway/ingress/nginx.conf:43-44` `/api/` → upstream; WAF generic. **No availability-specific routing, versioning, or rate tier.** Frontend rewrite chain: `services/api.ts:1` (`API_BASE='/api'`) → `next.config.js:57-64` → `http://localhost:4000/api/v1`.

---

## 8. Phase 1–4 Authority Compliance Matrix

Legend: ✅ compliant · ⚠️ partial/flag-gated · ❌ non-compliant · ➖ not applicable

| # | Consumer | Authority dependency | Compliance | Required migration implication | Risk |
|---|---|---|---|---|---|
| M-01 | Availability snapshot HTTP | owns it | ✅ | expose as the single read contract | — |
| M-02 | Assertion engine | owns it | ⚠️ | usable only after DS-01 (restriction evaluator) | critical |
| M-03 | Reservation create/cancel/no-show/reinstate/batch/waitlist | port | ✅ | none | low |
| M-04 | Reservation modify (`update`) | port **+ legacy CRS** | ❌ dual-write `:621/:625` | remove legacy leg or schedule to Phase 11 (DS-05) | high |
| M-05 | Reservation delete | none | ⚠️ | confirm terminal-state handling with Stage 2 | medium |
| M-06 | FO check-out | port | ✅ | none | low |
| M-07 | FO upgrade | port for write, legacy for gate | ❌ | replace `:69` with authority consult | high |
| M-08 | FO check-in / assignment / transfer | none (room-level) | ➖ | annotate as out-of-scope by design | low |
| M-09 | CRS engine `/rates/engine/*` | legacy counters | ❌ | retire or re-point (DS-03) | high |
| M-10 | `GET /rates/availability` | none | ❌ unscoped ad-hoc | retire/re-point + tenant scoping (DS-03) | critical |
| M-11 | A3 matrix (flag OFF) | none | ❌ second engine | DS-02 surface decision | high |
| M-12 | A3 matrix (flag ON) | snapshot numbers, raw restrictions | ⚠️ hybrid | cannot pass parity while DS-01 open; restrictions must move too (DS-04) | high |
| M-13 | A3 restriction writes | bypass | ❌ | authority write path (DS-04) | high |
| M-14 | GBA pickup guards | own ledger (correct owner) | ✅ | optional consult flag promotion | medium |
| M-15 | GBA pickup-created reservations | raw insert, no assertion | ⚠️ | phase-4 A5 carry-over; confirm in Stage 2 | medium |
| M-16 | Event cascade / invalidation | partial | ⚠️ | invalidation no-op (F-14) | medium |
| M-17 | Channels push | none | ❌ | DS-07 (outbound contract) | medium |
| M-18 | Analytics/FO/command-center KPIs | none | ❌ | DS-06 (canonical occupancy) | medium |
| M-19 | Web availability page | A3 payload | ❌ | migrate to snapshot per DS-02 | high |
| M-20 | Web rates tab / CRS book gate / quick-book matrix | legacy | ❌ | migrate or remove (DS-03) | high |
| M-21 | Web group/allotment views | client recompute | ⚠️ | consume ledger read models instead of recomputing | medium |
| M-22 | Admin / mobile | none | ➖ greenfield | build on snapshot (DS-02) | low |
| M-23 | Frontend authority consumers | **none exist** | ❌ | net-new `useAvailabilitySnapshot` | critical path |

---

## 9. Legacy vs Canonical Behavior Matrix

| Dimension | Canonical (Phase 1–4 authority) | Legacy / shadow today | Where |
|---|---|---|---|
| Sellable formula | `sellableAvailable = max(0, min(capacityWithOverbooking, sellLimit ?? ∞) − consumption)`; floors always `max(0,·)`; sellLimit can only tighten | A3: `max(0, physical − ooo − reserved − groupCommit − allotCommit)` with proportional unassigned distribution; legacy counters: `reserved ± 1`; ad-hoc: `rooms.count() − reservations.count()` | `snapshot-calculator.ts:13-22` vs `availability-sales.controller.ts:548-581`, `inventory.domain-service.ts:96-117`, `rates-inventory.service.ts:34-40` |
| Block eligibility | `deleted_at IS NULL` + booking `status IN (CONFIRMED,ACTIVE)` + block `status IN (DEFINITE,OPEN_FOR_PICKUP)` else UNRESOLVED | A3: `status NOT IN (CANCELLED,CLOSED)` (DRAFT/TENTATIVE count), no `deleted_at`, no booking-status filter | `availability-source.adapter.ts:49-67` vs `availability-sales.controller.ts:306-335` (Phase 4 `04_…§3`) |
| Allotment eligibility | `contract_type='HARD_COMMITMENT'` + `status='ACTIVE'` + validity window | A3: all contract types, `status IN ('ACTIVE','CONFIRMED')` (invalid value), no validity window | same pair |
| Restriction evaluation | 8 stores combined → **currently always `UNRESOLVED`** (fail-closed ⇒ `sellableAvailable: 0`, `bookingEligibility: 'UNKNOWN'`) | A3 reads `close_to_arrival/departure`, `zero_sell_limits` raw and **always applies them** (not flag-gated); CRS reads only `rate_restrictions`; a second table `restrictions (rate_code='CUTOFF')` is written but never read | `unresolved-restriction.adapter.ts:42-82`; `availability-sales.controller.ts:384-405,540-543`; `crs-engine.service.ts:152` |
| Reservation consumption | `normalizeStatus` canonical classification; quantity = candidate count; excludes `ASSERTION_MANAGED` (dual-count guard); `CHECKED_OUT` consumes | A3 counts raw reservation rows + proportional unassigned; rates/CRS counts `CHECKED_IN,CONFIRMED` only | `reservation-consumption.adapter.ts:6,22,28-53` vs `availability-sales.controller.ts:314-339`, `rates-inventory.service.ts:36-38` |
| Checkout effect | `CHECKED_OUT` **holds, does not release** (ratified D-1 / invariant 26) | legacy counters unaffected (they are only mutated by CRS engine) | `reservation.repository.ts:69-75,1053` — **compliant**, §13 |
| OOO/OOS | subtracted from physical capacity | A3 subtracts OOO; `/rates/availability` ignores OOO entirely | `snapshot-calculator.ts:15` vs `availability-sales.controller.ts:369-381` |
| Overbooking | measured (`overbookingUsed`), allowance from `overbooking_limits` | legacy `availability.oversell` flag math; A3 has no allowance | `availability-source.adapter.ts:116`; `inventory.domain-service.ts:106,115` |
| Fail-closed | unresolved ⇒ `physicalAvailable: 0`, `sellableAvailable: 0`, `bookingEligibility: 'UNKNOWN'`, assertion `REJECTED` | A3/legacy never fail closed — they trust whatever rows exist | §10 F-01, F-03 |
| Pickup ledger | `group_pickups` / `allotment_pickups` canonical; `picked/released` counters | legacy `block_pickup`/`orms_pickup` tables are schema-only (no writer) | Phase 4 `07_LEGACY_DEPENDENCY_MAP.md` §1 |
| Persistence of availability decision | assertion balance = Σ movements (append-only trigger) | `availability` counters updated in place | migration `20260927000000_…` vs `inventory.domain-service.ts:164,181` |
| Tenant scoping | `properties/:propertyId` + authorize + permission codes | `GET /rates/availability` **no `hotel_id`**; A3 `@PropertyScope(false)` + `requireHotelId()` | F-06, F-19 |

---

## 10. Findings / Gaps

Severity: **CRITICAL** (blocks Phase 5 core), **HIGH**, **MEDIUM**, **LOW**.

### CRITICAL

**F-01 — The authority cannot resolve a restriction, therefore cannot assert capacity, under production DI.**
*Evidence:* `availability.module.ts:26` binds `RESTRICTION_EVALUATOR → UnresolvedRestrictionAdapter`; `unresolved-restriction.adapter.ts:47-59` unconditionally pushes `zero_sell_limits, close_to_arrival, close_to_departure, minimum_length_of_stay, maximum_length_of_stay, room_type_sell_limits, allotment_stop_sales` into `unresolvedSources` and hardcodes `status: 'UNRESOLVED'` (the unit spec `restriction.adapter.spec.ts:14` locks this in as designed). Snapshot propagation: `restriction.status !== 'RESOLVED'` ⇒ `sellableAvailable: 0` and `bookingEligibility: 'UNKNOWN'` (`availability-snapshot.service.ts` results block). Assertion: `unassertableReason` (`availability-assertion.service.ts:881-893`) → `UNRESOLVED_CAPACITY` → `capacityRejection` never reached → result `REJECTED` → `assertReservationCreate` throws `AVAILABILITY_ASSERTION_REJECTED` and rolls the transaction back (`reservation-availability-wiring.ts`). *No* test wires the production binding end-to-end: `availability-snapshot.service.spec.ts:8-10` injects `{status:'RESOLVED', unresolvedSources:[]}`; `availability-postgres.harness.ts:33-53` `makeSnapshots()` hardcodes `bookingEligibility:'ELIGIBLE'`, `restrictionOutcome:{status:'RESOLVED'}`; `restriction.adapter.spec.ts` tests the stub in isolation. *Implication:* (a) frontend migration onto `/snapshot` would render `0 available / UNKNOWN`; (b) `gba.a3.authoritative=ON` would set **every** matrix `available` cell to `0` (`availability-sales.controller.ts:540` uses `fact.sellableAvailable`); (c) consuming reservation writes appear to fail closed in production wiring. **Not resolved here — DS-01.**

**F-02 — Zero consumers of the authority read surface.** Repo-wide grep for `availability/snapshot|availability/reconciliation` across `apps/**`, `gateway/**`, `packages/**` returns 0 hits (§15.3-E1). The frontend's entire availability experience sits on legacy/A3 endpoints.

**F-03 — A3 is a second complete availability + restriction engine.** `activities/availability-sales.controller.ts` (966 lines, root `@Controller()`, `@PropertyScope(false)`): parallel capacity computation (§4.4 #3/#4), raw restriction **reads** not flag-gated (`:384-405`), and raw restriction **writes** to 8 tables (`:676-707,825-885`) — including a second restrictions table (`restrictions`, `rate_code='CUTOFF'` `:878`) that nothing reads. It directly contradicts Phase 4 decision text: *"Availability … is the sole authority … No other component may compute or publish an independent 'available' number"* (`10_DECISION_RESOLUTION.md:114`).

**F-04 — Dual-write + one-sided legacy update on reservation modify.** `reservation.repository.ts:621-623` (authority `replaceReservationAssertion`) then `:625` (`crs.modifyReservation` → legacy `availability` counters) inside one transaction; meanwhile `applyStatusAvailability` (`:1000-1055`) updates only the authority, so cancel/no-show/reinstate never decrement legacy counters. The two stores therefore **drift by design** in opposite directions.

### HIGH

**F-05 — FO upgrade gates on legacy, writes to authority.** `upgrade-room.handler.ts:69` (`inventoryDomain.isAvailable`) throws/permits using the legacy table, then `:85 → :166/:172` commits to the assertion engine. Compounding: `isAvailable` returns **`true` when the row is absent** (`inventory.domain-service.ts:126`).

**F-06 — `GET /rates/availability` is tenant-unscoped and div-by-zero prone.** `rates-inventory.service.ts:34-40`: `prisma.rooms.count()` and `reservations.count()` carry **no `hotel_id`**; `occupancyPct = round((occupied/total)*100)`; consumed live by `apps/web/app/(dashboard)/rates-inventory/page.tsx:143` under the heading **"Availability Snapshot"** (`:314`).

**F-07 — A legacy source gates writes.** `use-crs-book.ts:87,129` and `front-office.api.ts:193` block booking on `GET /rates/engine/quote` → `crs-engine` → legacy counters.

**F-08 — Two availability features in the primary UI are silently broken (404) plus payload mismatch.** `reservation.api.ts:405,420` → `/availability/matrix/logs` (backend `/availability/logs`); `:431` → `/activities/availability/interval-update` (backend `/availability/interval-update`, and body shape `ratePlanCodes/startDate/endDate` vs `ratePlans/dateRange`). Callers: `AvailabilityPage.tsx:545` (audit log) and `:309` (interval update).

**F-09 — Frontend independently computes availability/occupancy in ≥12 places** (§5.3), including two defective ones: header row labelled "avail" that sums `physicalInventory` (`AvailabilityPage.tsx:1006-1015`) and occupancy from a client formula (`:870-871`).

**F-10 — Quick-book matrix always renders 0 available.** `AvailableRatesMatrix.tsx:61-89` builds cells with literal `available: 0` in both branches; `matrixResponse.inventory` is never read.

**F-11 — `room_inventory` has no writer.** Only readers: `availability-sales.controller.ts:62,249-252`. `availableRooms` on `GET /availability/room-types` is therefore `COALESCE(...,0)`/fallback — an availability figure sourced from a table nothing populates (population method UNVERIFIED, §15.4).

**F-12 — Many simultaneous, contradictory "available/occupancy" numbers** (§9), including hardcoded `78.4%` (`reporting-analytics/page.tsx:14`), admin mocks, `hotelStore.totalRooms || 200`.

**F-13 — No shared contract type for the authority payload** (§7.3).

### MEDIUM

**F-14 — Availability cache invalidation is a no-op.** `events.consumer.ts:195-202` deletes `availability:${hotelId}:*`; the authority is `LIVE_READ` with no cache writer (single grep hit is the delete itself) — and this is the *only* thing `gba.consumers.cascade` gates in `handleGbaMutation` (`:188`).

**F-15 — Outbound channel availability publication does not exist.** `channels.service.ts:162-183` inserts caller-supplied `available` into `channel_availability_log`; `channel_availability` is never written; `packages/analytics/.../channel-sync.job.ts` is a `console.log` stub. Documented as deliberately Phase 5+ (`13_FINAL_DOMAIN_SPECIFICATION.md:65`) — carried as DS-07, not treated as a regression.

**F-16 — Availability write-identity gaps.** `change-rate.handler.ts:30` passes no write identity ⇒ `randomUUID()` operation keys, `actorId='system'`; create path has `idempotencyKey: undefined` for walk-in/webhook (`reservations.service.ts:89-93`, `sub-resources.handlers.ts:477`, `webhook.service.ts:71`) ⇒ retried creates mint new assertion identities; `hotelId || 'default'` fallback (`create-reservation.handler.ts:25`) is a value the authority rejects (`availability-snapshot.service.ts` authorize).

**F-17 — 48 of 178 API suites silently skip without `AVAILABILITY_TEST_DATABASE_URL`.** `availability-postgres.harness.ts:8-9`; no jest `setupFiles`; the variable is absent from `.env`, `.env.example`, and `compose.yaml`. Green runs can mask an unexecuted authority battery.

**F-18 — Two disjoint feature-flag systems** (env `FEATURE_*` via `config.service.ts:63-65` vs DB `platform/configuration/feature-flag.service.ts`), and **no `FEATURE_*` entry exists anywhere in config**, including `gba.consumers.cascade` which Phase 4 marks "MUST be ON at deploy" (`17_PHASE4_READINESS_REVIEW.md:80`).

**F-19 — Property-scope divergence on the authority controller.** Controller authorizes `@Param('propertyId')` (`availability.controller.ts:13-17`) while `core/guards/property-scope.guard.ts:28-45` may set scope from `x-property-id`/`user.tenantId` (super-admin prefers header over route) — guard can validate property X while data is read for property Y. A3/Banquet controllers bypass the guard entirely (`@PropertyScope(false)`).

**F-20 — Duplicate route `GET /tax-rates`.** Declared in `banquet-refs.controller.ts:45` and `availability-sales.controller.ts:131`, both `@Controller()`; `activities.module.ts:24-25` registers Banquet first ⇒ Express first-match wins, availability variant unreachable (runtime winner UNVERIFIED).

**F-21 — Dead code cluster carrying conflicting formulas.** Frontend: `AvailabilitySales.tsx`, `AnimatedRoomRack.tsx`, `features/operasales/**`, `reservationStore.fetchRoomTypes`, `analyticsStore.fetchOccupancy`, duplicate rate-grid types, duplicate routes. Backend: `inventory.domain-service` dead members (`dateList,reserveRooms,releaseRooms,blockAvailability,consumePickup,releaseUnsold`), dead DI (`reservation.repository.ts:15,89`), `PrismaInventoryReservationAdapter`, `PrismaRatesAdapter`, `IInventoryCommitmentPort`.

**F-22 — Test baselines not reproducible statically.** Documented baselines: API 2 suites/6 tests, web 1 suite/10 tests (`17_PHASE4_READINESS_REVIEW.md:57-60`). Static analysis: `reservation-integration.spec.ts` = 3 (confirmed `:70,:81,:86`); check-in `queries.handler.spec.ts` = 3 confirmed + a 4th candidate (`:177-182` vs `room.aggregate.ts:116-125`); web `reservation-actions.test.ts` = 1 identifiable (`:44-47`). Requires an executed run to reconcile (DS-09). *No tests were executed in Stage 1.*

**F-23 — Phase 4 documentation drift.** `04_AVAILABILITY_INTEGRATION_MATRIX.md:84-86` still lists `GET /group-bookings/available-rooms` (`group-booking.controller.ts:95-116`, `group-allotment.api.ts:29`) as live — retired and enforced absent by `t39-available-rooms-absence.spec.ts`; multiple cited line numbers in that document are stale (A3 `:237`→`:265`, GBA reads, formula `:459`→`:566`).

**F-24 — Phase 4 exit actions remain open** (`17_PHASE4_READINESS_REVIEW.md:92-99`): 766-file Stage-6 delta uncommitted (776 dirty paths observed this session), canonical cutover (`canonicalRead` → soak → `canonicalWrite`) not started, `gba.consumers.cascade` not configured ON, BLK-1/BLK-2 still gate wash.

### LOW

**F-25 — Duplicate web routes** for allotments and group-blocks (`/allotments` vs `/reservations/allotments`, `/group-blocks` vs `/reservations/group-blocks`) — duplicate cache keys/nav.

**F-26 — Raw-SQL interpolation surface in A3.** `availability-sales.controller.ts:279-281,343-366` interpolate `roomType`/`rateCode` into `$queryRawUnsafe` with quote-escaping only.

**F-27 — OTA overbooking branch likely dead (UNVERIFIED).** `webhook.service.ts:104` detects overbooking via `err.message.includes('inventory')`, but the authority throws `AVAILABILITY_ASSERTION_REJECTED` (`reservation-availability-wiring.ts`) — no `"inventory"` substring.

---

## 11. Risks and Dependencies

| ID | Risk | Sev | Trigger | Dependency |
|---|---|---|---|---|
| R-01 | Frontend migrated onto `/snapshot` shows 0/UNKNOWN for every date | critical | starting DS-02 migration before DS-01 | DS-01 |
| R-02 | Enabling `gba.a3.authoritative` zeroes the availability matrix | critical | flag ON during parity soak | DS-01, then DS-02 |
| R-03 | Consuming reservation writes rejected/failing closed in production wiring | critical | any CONFIRMED/GUARANTEED/CHECKED_IN create/modify | DS-01 (+ empirical confirmation, §15.4) |
| R-04 | Oversell via the legacy write gate (`/rates/engine/book`) while authority is authoritative | high | any booking through quick-book/CRS | DS-03 |
| R-05 | Cross-tenant data exposure on `GET /rates/availability` | high | concurrent properties, live UI | DS-03 (or immediate scoping — but Stage 1 proposes no code) |
| R-06 | Two disagreeing availability stores after every reservation modify (F-04) | high | any modify | DS-05 ordering vs Phase 11 |
| R-07 | Restriction edits bypass the authority and are invisible to snapshots | high | any `bulk-update`/`interval-update` | DS-04 |
| R-08 | User-visible contradictory availability across screens | high | any multi-screen workflow | DS-02, DS-06 |
| R-09 | Silent feature breakage (404s) already live in the primary availability screen | high | opening availability audit-log/interval UI | DS-11 |
| R-10 | False-green CI (48 skipped suites; unverified baselines) | medium | default test run | DS-09 |
| R-11 | Flag drift: documented "MUST be ON" flag absent from all config | medium | first release | DS-10 |
| R-12 | Wash/attrition activation before Deviations A+B resolved | medium | enabling `gba.wash.schedulerEnabled` | Phase 4 carry-over (not Phase 5 scope) |
| R-13 | Pickup-created reservations invisible to assertion balances (A5 carry) | medium | GBA pickup volume grows | DS-01/Stage 2 confirmation |
| R-14 | No shared contract type ⇒ frontend/backend drift during migration | medium | migration begins | F-13 remediation (Stage 3/4) |
| R-15 | Phase 4 delta uncommitted (776 dirty paths) — audit evidence could be lost | medium | any git operation | explicit authorization outside Stage 1 |

**External dependencies:** Phase 4 Deviations A/B (wash gating), Phase 2b integrity hardening, Phase 11 legacy retirement, compose Postgres availability for postgres suites.

---

## 12. Items Requiring Stage-2 Business Decisions

*Recorded, not resolved. No rule is decided in this document.*

| ID | Question | Why Stage 2 | Blocking? |
|---|---|---|---|
| **DS-01** | **Restriction authority:** build a real `RestrictionEvaluator` covering all 8 stores (and reconcile `restrictions` CUTOFF vs `rate_restrictions`), **or** keep fail-closed and gate assertion enforcement / matrix cutover behind a documented interim rule? | Decides whether the authority can ever return `ELIGIBLE`, whether assertions are enforceable today, and whether `gba.a3.authoritative` can be turned on. Changes no Phase 1–4 rule — it *implements or defers* TR-1.1's "selling permission" fact kind (Phase 3 never ruled on restrictions: only 2 hits in the Phase 3 audit, none in the Phase 3 spec/decision sheet). | **YES — hard blocker** |
| **DS-02** | **Frontend read contract:** is `/properties/:pid/availability/snapshot` the contract the UI migrates to, or does `/availability/matrix` remain the contract (authority-backed) with the snapshot hidden behind it? Shape, roomType×bedType grouping, and `label`/`bookingEligibility` surfacing must be chosen. | Determines every §5 migration task and whether F-13's shared type is snapshot- or matrix-shaped. | YES (conditional on DS-01) |
| **DS-03** | **Disposition of legacy rates/CRS availability:** retire `/rates/engine/*` + `GET /rates/availability`, or re-point them at the authority? Includes the quick-book write gate and tenant-scoping remediation. | Removes F-06/F-07; touches booking flow behaviour. | no |
| **DS-04** | **Restriction write ownership:** keep raw-SQL restriction writes in Activities (Phase 3 L-12 said "KEEP — Phase 1 authority"), or move them onto an authority-owned write path? | Changes who owns "selling permission" writes (TR-1.1). | no |
| **DS-05** | **Dual-write removal ordering:** is `reservation.repository.ts:625` removed in Phase 5 or deferred to Phase 11 legacy retirement? | Phase 4 explicitly deferred legacy deletion to Phase 11 (contract §18). | no |
| **DS-06** | **Canonical occupancy/KPI definition:** which number do dashboards, analytics, FO KPIs, and command-center widgets publish — snapshot-derived occupancy, an analytics contract, or a documented "reporting-only" exemption? | Phase 4 rule says no independent available number; occupancy is adjacent but not identical. | no |
| **DS-07** | **Outbound channel/CRS push contract** (spec `13_…:65` defers it to Phase 5+): in or out of Phase 5 scope? | Scope question for this phase. | no |
| **DS-08** | **Multi-property contracts** (TR-14.3 deferral): in or out of Phase 5? | Scope question. | no |
| **DS-09** | **Test-execution policy:** run the 48 postgres suites + full 178-suite battery to reconcile documented baselines (F-22), and require `AVAILABILITY_TEST_DATABASE_URL` in CI? | Affects evidence quality for later stage gates; Stage 1 executed nothing. | no |
| **DS-10** | **Flag governance for Phase 5:** which of the 7 flags Phase 5 may touch, which config surface documents them (`.env.example`/`compose.yaml`), and what happens to the second (platform DB) flag system? | Phase 5 will need at least one rollout gate of its own. | no |
| **DS-11** | **Broken UI features:** repair `matrix/logs` + `interval-update` (route + payload) or remove the dead UI affordances? Also: disposition of reservation **delete** with respect to a still-live assertion for terminal `CHECKED_OUT` rows (no availability op at `repository.ts:889-924`). | Behaviour change, not audit. | no |
| **DS-12** | **Source of truth for physical/OOO inputs:** who writes `room_inventory`, `out_of_order`/`out_of_service`, and the restriction tables the authority must read? | F-11 + DS-01 input completeness. | no |

---

## 13. Items Already Correct

Verified as compliant — **do not reopen**:

1. **`CHECKED_OUT` does not release its assertion** — matches ratified Phase 3 D-1 / invariant 26 (`availability-phase3-domain-specification.md:180,190`); `RELEASING_STATUSES` (`reservation.repository.ts:69-75`) and `applyStatusAvailability` default (`:1053-1054`) implement it. Recorded as **compliant**, contradicting any "missing release" reading.
2. **`reservation-consumption.adapter.ts` implements D-0/D-1**: canonical `normalizeStatus` (`:47`), quantity = candidate count, `CHECKED_OUT` consumes (`:6`), unclassifiable ⇒ `UNRESOLVED` (`:49-50`), and the `ASSERTION_MANAGED` dual-count guard (`:22`).
3. **FO checkout performs zero GBA writes** (T-31/L-13 closed) — `check-out.handler.ts:402-405`; guarded by `check-out.handler.spec.ts`.
4. **Assertion wiring exists on 15 command handlers** (create, confirm, guarantee, cancel, change-room-type, extend, batch, mass-update, no-show, reinstate, auto-cancel, waitlist join/promote, scheduled room move).
5. **Legacy `availability` table has exactly one writer file**, with static grep gates (`t65-legacy-write-absence.spec.ts:40,46,118,141`) and a non-authority test (`t69-…`).
6. **F-18 `GET /group-bookings/available-rooms` retired** and enforced absent (`t39-available-rooms-absence.spec.ts`).
7. **GBA pickup canonical flags default OFF with proven rollback** (`t68-canonical-write-switch.postgres.spec.ts`), and `allotment_pickups` freezes only when `canonicalWrite` ON.
8. **Hard-delete is blocked for non-terminal statuses** (Decision 12) — `reservation.repository.ts:896-901`.
9. **Room-level Front Office decisions (check-in room options, room assignment, same-type transfer) correctly sit outside rate-type inventory authority** and are documented as such (`get-check-in-room-options.handler.ts:15-24`).
10. **GBA owns its own ledger correctly** (`quota − picked − released` / `contracted − picked − released`) and is the right authority for pickup capacity.
11. **`mass-update-execute.handler.ts:77` explicitly rejects `overrideAvailableInventory` bypass** — "the Availability engine is authoritative for every item".
12. **Outbound stock/warehouse inventory (`inventory/availability`, `warehouses/:id/capacity`), banquet/venue capacity, and `command-center/widgets/available`** are separate bounded contexts — correctly out of scope.
13. **Gateway has no availability-specific misconfiguration** (catch-all only).
14. **No unexpected writer to any authority-owned table** (`availability_assertions`, `availability_assertion_balances`, `reservation_availability_state/operations`, `group_pickups`, `group_block_daily_allocations`, `allotment_daily_quotas`) outside intended paths — verified by dedicated specs (`t65`, `t69`, `reservation-gba-invariance`).

---

## 14. Recommended Migration Boundaries

Proposed for Stage 4 (Implementation Plan) to consider — **not decisions**:

| Boundary | Contents | Rationale |
|---|---|---|
| **B-1 — Authority enablement (prerequisite)** | DS-01 outcome + restriction inputs (DS-12) + empirical confirmation of assertion behaviour (§15.4) | Nothing downstream is safe until the authority can produce a real number |
| **B-2 — Read-contract cutover (frontend)** | new `useAvailabilitySnapshot`; `AvailabilityPage` re-point; remove client occupancy formulas (§5.3); surface `bookingEligibility`/`label`; fix or remove broken routes (DS-11) | Largest user-visible surface; isolated by DS-02 |
| **B-3 — Secondary read surfaces** | rates-inventory tab, dashboard/FO/command-center/reporting occupancy (DS-06), quick-book matrix (F-10), group/allotment view math | independent of each other once B-1 lands |
| **B-4 — Write-path convergence** | upgrade gate (`:69`), legacy modify leg (`:625`), CRS engine retirement (DS-03/DS-05) | highest behavioural risk; must follow B-1 and respect Phase 11 boundaries |
| **B-5 — Restriction ownership** | A3 restriction reads/writes, second `restrictions` table, authority evaluator coverage (DS-04) | separable but entangled with B-1 |
| **B-6 — Outbound publication** | channel push built on snapshot (DS-07) | explicitly deferred by spec; optional for Phase 5 |
| **B-7 — Greenfield** | admin + mobile availability surfaces | no migration content |
| **B-8 — Hygiene (gated)** | dead-code removal (F-21), duplicate routes (F-25), doc drift (F-23), flag config (F-18/DS-10), test gate (F-17/DS-09) | must not precede behaviour-preserving work |

Sequencing constraint carried forward: **B-1 before B-2…B-6**; Phase 11 still owns legacy-table deletion (contract §18).

---

## 15. Complete Evidence Index

### 15.1 Authority documentation (Phase 1–4, read-only)

- `docs/availability/phase-4/01_FORENSIC_AUDIT.md` … `17_PHASE4_READINESS_REVIEW.md` (all 17 files listed in §15.1 inventory) — especially `04_AVAILABILITY_INTEGRATION_MATRIX.md` (A1–A9 consumers, §3 divergence, §5 frontend map), `07_LEGACY_DEPENDENCY_MAP.md` (§1 legacy GBA tables, §3 counter writer, §6 dead code), `11_TARGET_BUSINESS_RULES.md` (TR-1.1, TR-7.x, TR-10.3), `13_FINAL_DOMAIN_SPECIFICATION.md:65` (Phase 5+ exclusions), `14_IMPLEMENTATION_PLAN.md:239,559,563,923-939` (T-37/T-38, §11 flags), `15_READINESS_REVIEW.md:503-527,617-641,687-724,767-777` (legacy boundaries, DEF/NB/ID/FP registers, Stage-6 entry), `16_RR_DECISIONS.md`, `17_PHASE4_READINESS_REVIEW.md` (§5 deviations A–D, §6 flags, §8 actions).
- `docs/enterprise/availability-phase1-2-contract-extract.md` (§4 snapshot contract, §5 assertion contract, §7 Phase 3 divergences).
- `docs/enterprise/availability-phase3-domain-specification.md:180,190,215-216` (`CHECKED_OUT` hold rule), `…-business-rules-decision-sheet.md`, `…-implementation-plan.md`, `…-readiness-review.md`, `…-reservation-lifecycle-forensic-audit.md:761-773,763,842` (legacy register L-items, L-12).
- `docs/availability/phase-4/10_DECISION_RESOLUTION.md:114` (sole-authority decision text).

### 15.2 Backend files read in full or in material part

`availability.module.ts`; `api/controllers/availability.controller.ts`; `application/services/availability-snapshot.service.ts` (results block); `application/services/availability-assertion.service.ts` (`:865-905`, rejection chain); `domain/policies/snapshot-calculator.ts` (via contract extract); `infrastructure/adapters/unresolved-restriction.adapter.ts` (**full**); `infrastructure/adapters/reservation-consumption.adapter.ts` (**full**); `infrastructure/adapters/__tests__/restriction.adapter.spec.ts` (**full**); `reservation-availability-wiring.ts` (**full**); `reservation.repository.ts` (`:69-75,:430-500,:610-670,:889-925,:999-1055`); `inventory.domain-service.ts` (member/caller map); `rates-inventory.controller.ts`; `rates-inventory.service.ts` (`:20-41`); `activities/availability-sales.controller.ts` (`:1-34,:645-715,:765-805,:455-555`); `activities.module.ts` (controller order); `banquet-refs.controller.ts`; `channels/channels.service.ts` (`:158-204`); `upgrade-room.handler.ts`; `check-out.handler.ts` (availability calls); `core/guards/property-scope.guard.ts`; `common/config/config.service.ts:63-65`; `availability-postgres.harness.ts` (`:1-70`).

### 15.3 Search evidence (grep/ripgrep, read-only)

- **E1** `rg -n "availability/snapshot|availability/reconciliation" apps` → **0 hits** (frontend/other consumers of authority).
- **E2** counts: 286 files (`availability|sellable|occupancy` across `apps`, `packages`, `gateway`); 149 `apps/web`; 275 `apps/api/src`; 7 admin/mobile/ui.
- **E3** `rg -n "getFeatureFlag\(" apps/api/src` → 15 call sites / 7 distinct flags (§7.4); `rg FEATURE_` in `.env`, `.env.example`, `compose.yaml` → **0**.
- **E4** test counts: 178 `*.spec.ts` (api), 48 `*.postgres.spec.ts`, 5 `*.test.ts*` (web); sole `it.skip` = `t45-execute-cut-off-wash.spec.ts:293`.
- **E5** web availability endpoint calls (`reservation.api.ts:214-431`, `reservationStore.ts:187`, `AvailabilitySales.tsx:55-56`) vs backend routes (`availability-sales.controller.ts:41,265,651,719,752,773,900`).
- **E6** `rg "room_inventory" apps/api packages/db` → schema + 2 readers, **no writer**.
- **E7** `rg "tax-rates" apps/api/src` → 2 declarations.
- **E8** `rg "RESTRICTION_EVALUATOR|implements RestrictionEvaluator" apps/api/src` → single binding, single implementation.
- **E9** `rg -i "availab" apps/admin apps/mobile packages/ui-*` → 7 incidental files.
- **E10** git: 4 commits, HEAD `c854f79`, `git status --short` = 776 paths (pre-existing, untouched).

### 15.4 Items that remain UNVERIFIED (declared, not hidden)

1. **Runtime confirmation of F-01** (assertion rejection under production DI) — static chain is complete and deterministic, but no execution was permitted/attempted in Stage 1. Recommend an executed check in Stage 2 (DS-09).
2. Runtime winner of the `/tax-rates` collision (F-20) — Express registration order implies Banquet, unconfirmed.
3. Population source for `room_inventory` and the restriction tables (external job/DBA script?) — no in-repo writer.
4. Documented test-baseline counts (6 API / 10 web) vs static analysis (§10 F-22).
5. External (out-of-repo) consumers of `/rates/engine/*`.
6. Live behaviour of the OTA overbooking branch (F-27).

---

## 16. Phase 5 Audit Conclusion

1. **The Phase 1–4 authority is complete as a write-side mechanism and orphaned as a read-side contract.** Reservation/FO write paths are wired (§6.1/6.2) and several legacy behaviours are correctly preserved (§13); the two HTTP authority endpoints have **no consumers at all** (F-02).
2. **Migration cannot begin before DS-01.** Because the production restriction binding is permanently `UNRESOLVED`, the authority currently returns `sellableAvailable: 0` / `bookingEligibility: 'UNKNOWN'` and rejects assertions with `UNRESOLVED_CAPACITY` (F-01). A frontend migrated onto it today would display zeros, and `gba.a3.authoritative=ON` would zero the matrix. This is recorded as the single **hard blocker** for Phase 5 and is explicitly deferred to Stage 2.
3. **The legacy field is broader than Phase 4's map implied.** Beyond the known A3/legacy-counter pair there are ad-hoc KPI computations (§4.4), client-side formulas (§5.3), mock numbers, two broken routes, an unscoped tenant query, and a legacy write gate that blocks bookings (F-04…F-12).
4. **Nothing in Phase 1–4 needs reopening.** Every candidate "bug" checked against ratified decisions resolved as compliant (§13.1 is the key example).
5. **Scope questions inherited by Phase 5** are external-channel push and multi-property contracts (`13_FINAL_DOMAIN_SPECIFICATION.md:65`) — surfaced as DS-07/DS-08, not decided.
6. **Verdict: Stage 1 COMPLETE.** 27 findings, 12 Stage-2 decision items, 1 hard blocker, 0 files modified besides this document.

**STOP.** Stage 1 ends here. Stages 2–6 (Business Rules/Decisions → Final Domain Specification → Implementation Plan → Readiness Review → Implementation) must not begin until explicitly instructed.

---

### Stage-1 closing report

| Item | Result |
|---|---|
| **Files inspected (read/grepped)** | 17 Phase-4 documents + 8 Phase-1–4/Phase-3 authority documents; **~60 backend source files** read in material part; **~40 frontend files** read; **286** distinct source files matched by availability/sellable/occupancy grep; 178 API + 5 web test files inventoried (selected suites read in full) |
| **Files modified** | **ZERO** — except this artifact (`docs/availability/phase-5/01_FORENSIC_AUDIT.md`). No source, schema, migration, config, env, or test file was touched; no git command that mutates state was run |
| **Tests / builds executed** | **NONE** (Stage 1 is read-only; test execution is proposed as DS-09) |
| **Findings count** | **27** (F-01…F-27): 4 critical, 9 high, 10 medium, 4 low |
| **Decision-required items** | **12** (DS-01…DS-12) |
| **Blockers** | **1 hard:** DS-01 (restriction authority ⇒ assertion/read unusable). **1 conditional:** DS-02 (frontend read contract) once DS-01 resolves |
| **Non-blocking findings** | 26 findings + Phase-4 carry-overs (Deviations A/B, NB-1…NB-6, open Phase-4 exit actions, uncommitted 776-path delta) |
| **Recommended next step** | Convene **Stage 2 — Business Rules / Decisions** on `DS-01` first, then `DS-02`, `DS-03`, `DS-04`; produce `docs/availability/phase-5/02_BUSINESS_RULES_DECISIONS.md` as artifact **02**. Do not start implementation, migrations, or frontend changes before then |

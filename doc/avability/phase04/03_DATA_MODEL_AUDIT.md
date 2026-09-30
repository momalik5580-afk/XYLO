# Phase 4 — Data Model Audit (GBA / Allotment)

**Rule applied:** tables are described from `packages/db/schema.prisma` + migrations + raw SQL actually executed. Where the three disagree, the disagreement is the finding.

---

## 1. Three Parallel GBA Data Worlds

| World | Tables | Created by | Written by |
|---|---|---|---|
| **NEW (GBA module)** | `group_bookings`, `group_blocks`, `group_block_daily_allocations`, `group_pickups`, `allotment_contracts`, `allotment_daily_quotas`, `allotment_vouchers`, `allotment_stop_sales`, `*_analytics` | **no migration** (schema drift — see §2) | `group-allotment` repositories + raw SQL handlers |
| **LEGACY (Int-key)** | `allotment` (`:105`), `allotment_pickup` (`:137`), `allotment_room_types` (`:152`), `block_pickup` (`:991`), `block_rates` (`:1012`), `block_room_type` (`:1030`), `reservation_block` (`:9127`), `reservation_groups` (`:9100`), `datamart_block_stats` (`:3102`), `allotment_ledger` (`:16068`), `orms_pickup` (`:7292`) | migrations incl. `20260920_groups_blocks_allotments` | **no TS writer found** for `block_pickup` / `orms_pickup` / `reservation_block` |
| **SHADOW (referenced, never declared)** | `allotment_pickups` (plural), `group_blocks.shoulder_days_before/after` | **none — no migration, not a Prisma model** | raw SQL at runtime |

---

## 2. Schema Drift — Evidence

```
rg -l -F 'group_blocks'         packages/db/migrations  →  (no files)
rg -l -F 'allotment_contracts'  packages/db/migrations  →  (no files)
rg -l -F 'allotment_pickups'    packages/db/migrations  →  (no files)
rg -l -F 'shoulder_days'        packages/db/migrations  →  (no files)
rg -F     'allotment_pickups'   packages/db/schema.prisma → (no match; model is 'allotment_pickup' at :137)
```

49 migration directories exist; `20260920_groups_blocks_allotments/migration.sql` (the only GBA-titled migration) contains:
- `CREATE TABLE reservation_groups …` (new legacy-style table)
- `ALTER TABLE reservation_block ADD COLUMN release_date/updated_at/billing_routing_config`
- `ALTER TABLE allotment ADD COLUMN contract_type/call_in_days/billing_routing_config`

It contains **no** `CREATE TABLE group_blocks`, `group_bookings`, `allotment_contracts`, `group_pickups`, `allotment_daily_quotas`, `allotment_vouchers`, or `allotment_stop_sales`.

**Implication (stated as fact about the repo, not about the live DB):** a database built solely from `prisma migrate deploy` would not contain the GBA tables that `group-allotment` requires; the working state therefore depends on `prisma db push` or hand-applied DDL. Whether those tables/columns exist in the live database was **not** verified (out of scope — no DB access during this audit) and is logged as an open verification item in `08_OPEN_DECISIONS.md`.

**Consequence if the shadow columns do not exist:** `add-shoulder-days.handler.ts:97-116` and `remove-shoulder-days.handler.ts:31-36,100-112` throw on every call. **Consequence if the shadow table does not exist:** allotment pickup create/cancel, `GET /allotments/:id/pickups`, and FO checkout pickup write-back all throw (the latter two inside `try/catch`, so they degrade silently — `check-out.handler.ts:390-477`).

---

## 3. NEW model reference (as declared)

### 3.1 `group_bookings` (`schema.prisma:17046-17077`)
`id` (text UUID), `hotel_id`, `booking_code` (unique with hotel: `uq_group_bookings_hotel_code`), `group_name`, `status` default **`PROSPECT`**, contact fields, `company_id`, `agent_id`, `arrival_date`/`departure_date`, `rate_plan_code`, `billing_method`, `billing_routing Json`, `notes`, `tags[]`, `stay_type`, `advance_deposit`, `version` default 1, `deleted_at`.
Relation: `blocks group_blocks[]`.

### 3.2 `group_blocks` (`:17079-17111`)
`id`, `hotel_id`, `group_booking_id` (**FK → `group_bookings.id`, `onDelete: Cascade`** `:17103`), `block_code` (unique with hotel), `name`, `status` default **`DRAFT`**, `arrival_date`, `departure_date`, `cutoff_date`, `rate_plan_code`, `negotiated_rate Decimal(12,2)`, `inventory_policy` default **`DEDUCT_INVENTORY`**, `contracted_nights`, `picked_nights`, `released_nights`, `attrition_threshold` default 80, `nightly_rooms`, `stay_type`, `version`, `deleted_at`.
**Missing from the model but read/written by raw SQL:** `shoulder_days_before`, `shoulder_days_after`.

### 3.3 `group_block_daily_allocations` (`:17113-17134`)
Natural key `@@unique([hotel_id, group_block_id, stay_date, room_type])` (`uq_group_alloc_hotel_block_date_rt`).
Columns: `contracted_qty`, `current_held_qty`, `picked_qty`, `released_qty`, `rate_per_night`, `version`.
FK: `group_blocks` with `onDelete: Cascade`.

### 3.4 `group_pickups` (`:17136-17159`)
`reservation_id String?`, `guest_name`, `room_type`, `arrival_date`, `departure_date`, `rooms_picked`, `status` default **`ACTIVE`**, `billing_routing`, `notes`. FK to `group_blocks` (Cascade). **No FK/unique constraint binding `reservation_id` to `reservations.id`** (plain indexed column `:17157`).

### 3.5 `allotment_contracts` (`:17161-17197`)
`allotment_code` (unique with hotel `uq_allotment_contracts_hotel_code`), `contract_ref_code`, `partner_name`, `partner_type`, `company_id`, `agent_id`, `contract_type` default **`SOFT_QUOTA`**, `status` default **`DRAFT`**, `validity_start`, `validity_end`, `release_days_before` default 14, `is_rolling_release` default true, `meal_plan`, `payment_terms`, `total_daily_quota`, `total_picked`, `total_released`, `version`, `deleted_at`.

### 3.6 `allotment_daily_quotas` (`:17198-17220`)
`@@unique([hotel_id, allotment_id, stay_date, room_type])` (`uq_allotment_quota_hotel_allot_date_rt`), `quota`, `picked`, `released`, `stop_sale_active`, `net_rate`, `version`.

### 3.7 `allotment_vouchers` (`:17221-17251`)
`@@unique([hotel_id, allotment_id, voucher_code])` (`uq_allotment_voucher_hotel_allot_code`), `reservation_id`, `rooms_count`, `status`, `issued_at/consumed_at/cancelled_at/cancellation_reason`.

### 3.8 `allotment_stop_sales` (`:17252-…`), analytics tables (`:17272+`, `:17306` unique `[hotel_id, allotment_id, snapshot_date]`).

---

## 4. LEGACY model reference (still declared, mostly writerless)

| Table / model | Line | Notes |
|---|---|---|
| `allotment` | `:105` | `@@unique([hotel_id, allotment_code])` `:129`; Int PK; extended by `20260920` |
| `allotment_pickup` | `:137` | singular; **not** the table the module writes |
| `allotment_room_types` | `:152` | legacy quota-by-room-type |
| `availability` | `:433` | counters `alloted`, `reserved`, `available`, `oversell`, `waitlist` — written by `inventory.domain-service.ts:164,181,227,256,275` |
| `block_pickup` | `:991` | no TS writer |
| `block_rates` | `:1012` / `block_room_type` `:1030` | no TS writer |
| `datamart_block_stats` | `:3102` | datamart |
| `orms_pickup` | `:7292` | no TS writer |
| `reservation_groups` | `:9100` | created by `20260920` |
| `reservation_block` | `:9127` | Int PK; extended by `20260920`; no TS writer |
| `reservations` GBA columns | `:9579+` | `block_code`, `group_block_id`, `allotment_id (Int)`, `allotment_voucher_id`, `pickup_type` |
| `room_blocks` | `:10288` | physical room blocks — different domain, often confused with group blocks |
| `allotment_ledger` | `:16068` | legacy ledger |

---

## 5. Cross-World Mismatches (structural)

| # | Mismatch | Evidence | Effect |
|---|---|---|---|
| D-1 | `reservations.allotment_id` is **Int** and points at legacy `allotment`; `allotment_contracts.id` is **String UUID** | `schema.prisma:9579` (`allotment_id Int?`) vs `:17162` (`id String`) | a reservation can never FK-link to a new-world contract; `PrismaReservationAssociationAdapter` consequently writes `allotment_id = NULL` and stores the voucher **code** in `allotment_voucher_id` |
| D-2 | Authoritative group link is `reservations.group_block_id → group_blocks.id`, but `reservations.block_code` also exists and is populated by a different writer | adapter `:105,112` vs `reservation-persistence.domain-service.ts:72,81` vs DTO `create-reservation.dto.ts:26` | two coexisting link representations; readers can disagree |
| D-3 | Two allotment pickup tables: `allotment_pickup` (declared) vs `allotment_pickups` (executed) | `schema.prisma:137` vs `create-allotment-pickup.handler.ts:130` | declared model is stale; runtime table undeclared |
| D-4 | `shoulder_days_before/after` used in raw SQL, absent from model and migrations | `remove-shoulder-days.handler.ts:100-112`; `schema.prisma:17079-17111` | column drift inside a table that itself has no migration |
| D-5 | New tables have no migration; legacy tables do | §2 | reproducibility gap: `migrate deploy` ≠ running schema |
| D-6 | Two block concepts: `group_blocks` (rooms commitment) vs `room_blocks` (physical) vs `reservation_block` (legacy) | `schema.prisma:17079 / 10288 / 9127` | naming collision in queries and in UI language |
| D-7 | `reservations.allotment_voucher_id` typed to accept legacy voucher reference while new vouchers live in `allotment_vouchers` with String ids | `schema.prisma:9579` vs `:17221` | association is by code, not by FK |
| D-8 | Status vocabularies differ across readers | activities SQL `ac.status IN ('ACTIVE','CONFIRMED')` (`availability-sales.controller.ts:332`) vs value object `AllotmentStatus = DRAFT/ACTIVE/SUSPENDED/CLOSED/EXPIRED` (`allocation-status.value-object.ts`) | `'CONFIRMED'` can never match → filter silently narrower than authored |

---

## 6. ID Generation Inconsistency (same table, two strategies)

| Site | Strategy |
|---|---|
| `allotment.repository.ts:224` (`save()` quota upsert) | `gen_random_uuid()` |
| `allotment.repository.ts:276` (`upsertQuota`) | deterministic `` `${allotmentId}-${dateStr}-${roomType}` `` |
| `allotment.repository.ts:309` (`batchSetQuotas`) | deterministic `` `${allotmentId}-${dateStr}-${roomType}` `` |
| `schema.prisma:17199` (model default) | `dbgenerated("(gen_random_uuid())::text")` |

Rows are protected from duplication by the natural unique key `@@unique([hotel_id, allotment_id, stay_date, room_type])`, so the strategies do not produce duplicate rows — but they do produce **non-deterministic primary keys** depending on which code path inserted first, which matters for any future FK from analytics/ledger tables and for `ON CONFLICT` behavior when a caller supplies an explicit `id`.

---

## 7. Counter Model (what each quantity means today)

### Group block (`group_block_daily_allocations`)
```
contracted_qty    – rooms committed for that night/room type
current_held_qty  – rooms still held (minus released)
picked_qty        – rooms consumed by pickups
released_qty      – rooms washed back
remaining (domain) = current_held_qty - picked_qty      // DailyRoomAllocation.remainingQty
canPickup(n)      → remainingQty >= n                   // allocation-status/daily-room-allocation VOs
```
Block-level rollups `contracted_nights`/`picked_nights`/`released_nights` are recomputed by the aggregate (`group-block.aggregate.ts` `recalculateNights()` at `:248`) and persisted at `group-block.repository.ts:119-121,131-133`. **Raw-SQL paths that bypass the aggregate (`remove-shoulder-days`, `remove-room-category`) delete allocation rows without recomputing those rollups** → stale block-level counters.

### Allotment (`allotment_daily_quotas`)
```
quota, picked, released, stop_sale_active, net_rate
remaining (domain) = quota - picked                     // issueVoucher checks quota - picked (:248)
contract rollup     = total_daily_quota / total_picked / total_released
```
`AllotmentRepository.save()` upsert **does not write the `quota` column** (`:226`: `DO UPDATE SET picked, released, stop_sale_active, net_rate`) — quota is written only by `upsertQuota` (`:282`) and `batchSetQuotas` (`:327`). This is why `SetAllotmentQuotaHandler` performs the double write (`set-allotment-quota.handler.ts:32` save → `:35` `upsertQuota`), outside any transaction.

---

## 8. Constraints / Integrity Gaps Observed in the Model

| Gap | Evidence |
|---|---|
| `group_pickups.reservation_id` has no FK to `reservations` | `schema.prisma:17140` (plain `String?`) + index `:17157` |
| `allotment_vouchers.reservation_id` has no FK | `schema.prisma:17221+` |
| `ConsumeAllotmentVoucherHandler` can store a **fabricated** reservation id | `consume-allotment-voucher.handler.ts:18` |
| No CHECK constraint preventing `picked_qty > contracted_qty` at DB level | `schema.prisma:17113-17133` (only `@@unique`/`@@index`) |
| No DB-level guard on `group_blocks` status vocabulary (free `String`) | `schema.prisma:17085` `status String @default("DRAFT")` — same for all GBA status columns (no enums) |
| `allotment_daily_quotas` unique key exists; `group_block_daily_allocations` unique key exists — both rely on `hotel_id` correctness, not on RLS | `:17130`, `:17215` |
| `version` columns incremented unconditionally (`{ increment: 1 }`) with no read-side predicate → optimistic locking is declared but never enforced | `group-block.repository.ts:134,168`; `allotment.repository.ts:153` |

---

## 9. Data-Model Verdict

1. **Two complete GBA worlds coexist** (`allotment`/`block_*` legacy vs `allotment_contracts`/`group_blocks` new) with a **type-level impossibility** of linking reservations to new contracts (D-1).
2. **The new world has no migration history** (D-5) and a **third, undeclared layer** (shadow table + shadow columns, §2).
3. **Counters are enforced only in application memory** — no DB constraint prevents over-pickup; enforcement depends on read-modify-write paths that carry no locks (see `06_CONCURRENCY_AND_TRANSACTION_AUDIT.md`).
4. **`version` columns exist everywhere but are never used as a predicate**, so the schema advertises optimistic concurrency the code does not implement.

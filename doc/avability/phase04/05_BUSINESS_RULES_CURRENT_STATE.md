# Phase 4 — Business Rules: Current State vs Target

**Convention:** §1 = CURRENT behavior evidenced from code. §2 = TARGET behavior from `docs/design/gba-domain-spec.md` (design reference, not ratified code). §3 = the delta. No rule below is asserted unless a file:line is given.

---

## 1. CURRENT Rules (implemented)

### 1.1 Group block lifecycle

| Rule as implemented | Evidence |
|---|---|
| New block starts at `DRAFT` (schema default) | `schema.prisma:17085` `status String @default("DRAFT")` |
| Aggregate status vocabulary: `DRAFT / TENTATIVE / DEFINITE / OPEN_FOR_PICKUP / CLOSED / CANCELLED` | `allocation-status.value-object.ts` (`GroupBlockStatus`) |
| `DeleteGroupBlockHandler` **cancels first, then soft-deletes**: `block.cancel()` → `repo.save()` → `repo.delete()` | `delete-group-block.handler.ts:26-30` |
| Soft delete = `deleted_at` timestamp; readers filter it (`findById` `:24`, `findAll`) | `group-block.repository.ts:24,70` |
| `cancel()` is subject to aggregate transition rules — a block already `CLOSED`/`CANCELLED` throws inside the aggregate, so delete of such a block fails *before* `delete()` runs | `delete-group-block.handler.ts:26-30` ordering |
| Booking confirm: `booking.confirm()` then save | `confirm-group-booking.handler.ts:26-28` |
| Booking default status `PROSPECT`; only `CONFIRMED`/`ACTIVE` bookings contribute to Availability (A1) | `schema.prisma:17051`; `availability-source.adapter.ts:49` |
| Aggregate **defines** full block transitions — `confirm()` (`:131`), `openForPickup()` (`:138`), `close()` (`:144`), `cancel()` (`:151`) — each guarded by `assertTransition` (`:119`) and emitting `GroupBlockConfirmedEvent`/`GroupBlockClosedEvent` | `group-block.aggregate.ts:119-156` |
| **but no command ever invokes `confirm()` / `openForPickup()` / `close()`** — the only aggregate transition executed by a handler is `block.cancel()` inside `DeleteGroupBlockHandler` (`:26`); `booking.confirm()` at `confirm-group-booking.handler.ts:26` is the *booking*, not the block | grep `\.confirm\(\)|\.openForPickup\(\)|\.close\(\)` across the module → only `booking.confirm()` |
| **`washAllocation` (`group-block.aggregate.ts:332`) has zero call sites** — wash exists as an aggregate method + `GroupBlockWashedEvent`, never executed | grep `washAllocation` → 1 hit (its own definition) |
| **A block therefore cannot reach `DEFINITE` or `OPEN_FOR_PICKUP` through any existing command**; because Availability counts only those two statuses (`availability-source.adapter.ts:66`), a block created today (status `DRAFT`) contributes **zero** consumption to the snapshot until its status is changed by some path outside the command set | `schema.prisma:17085` default `DRAFT` + `availability-source.adapter.ts:66,67` |

**Not implemented:** automatic release of inventory on cancel (no code path decrements a *different* system; release happens implicitly because Availability stops counting cancelled blocks — §4 of `04_AVAILABILITY_INTEGRATION_MATRIX.md`).

### 1.2 Daily allocation & pickup

| Rule as implemented | Evidence |
|---|---|
| Allocation exists per `(hotel_id, group_block_id, stay_date, room_type)` (DB unique key) | `schema.prisma:17130` |
| Pickup validates **per night**: allocation row must exist → `DailyAllocationNotFound` thrown otherwise | `group-block.aggregate.ts:234-238` |
| Pickup validates capacity: `allocation.canPickup(roomsCount)` → `remainingQty = currentHeldQty − pickedQty ≥ n`, else `PickupBeyondContractedQuantity` | `group-block.aggregate.ts:239-241`; `daily-room-allocation.value-object.ts` |
| Validation loop runs for **all nights first**, mutation loop runs after | `group-block.aggregate.ts:234-247` |
| Stay dates = half-open `[arrival, departure)` | `group-block.aggregate.ts:227-232` |
| **No cutoff-date check on pickup** — `_cutoffDate` is a stored field with a getter only (`:97`) and is never referenced by `createPickup` (`:214-279`) | grep `cutoffDate` in `group-block.aggregate.ts` → `:37,49,66,83,97,366,391,404,435,457` (all storage/serialization) |
| **No date-containment check** against `arrival_date`/`departure_date` of the block in `createPickup` | `group-block.aggregate.ts:214-279` — only allocation lookup |
| **No status gate** on pickup (a `DRAFT`/`CANCELLED` block still executes `createPickup` if rows exist) | `create-group-pickup.handler.ts:29-42` — loads block, no status check |
| Pickup auto-posts a ROOM charge to the master folio when `nights > 0 && negotiatedRate > 0` | `create-group-pickup.handler.ts:60-73` |
| Reservation creation failure aborts the whole command (throws → `Result.failure`) | `create-group-pickup.handler.ts:74-78` |
| Reservation created for pickup is `CONFIRMED`, `source_code='GROUP'`, `pickup_type='group_pickup'`, linked via `group_block_id` | `prisma-reservation-association.adapter.ts:100-120` |
| Guest resolution for pickup = `SELECT id … WHERE hotel_id=$1 AND full_name ILIKE $2 LIMIT 1`, else insert | `prisma-reservation-association.adapter.ts:82-97` |
| `remove-room-category` refuses if active pickups exist (`status != 'CANCELLED'`) | `remove-room-category.handler.ts:41-48` |
| `remove-room-category` hard-`DELETE`s allocations (no `deleted_at`) | `remove-room-category.handler.ts:51-55` |
| `remove-shoulder-days` hard-`DELETE`s allocations **without any pickup check** | `remove-shoulder-days.handler.ts:79-94` (contrast with §1.2 above) |

### 1.3 Release / wash / cut-off

| Rule as implemented | Evidence |
|---|---|
| Release is **manual, caller-driven**: `{stayDate, roomType, quantity}` supplied by the client | `release-block-allocation.handler.ts:26-33`; controller `group-booking.controller.ts:247` |
| Allotment release likewise manual | `release-allotment-allocation.handler.ts:26-33`; `allotment.controller.ts:239` |
| **No scheduled job** performs wash/release (module has no cron; only the generic outbox processor has `@Cron`) | `outbox-processor.ts:12,22,32` |
| **`ReleaseWindow.isWithinReleaseWindow` has zero call sites** | defined `release-window.value-object.ts:26`; grep shows only construction `allotment.aggregate.ts:517,586` |
| `release_days_before` (default 14) and `is_rolling_release` (default true) are persisted but never evaluated by a rule | `schema.prisma:17174-17175`; `allotment.repository.ts:127-128,145-146` |
| **No cutoff-date enforcement anywhere** (stored only) | `group-block.repository.ts:115,127` |

### 1.4 Allotment voucher & stop-sale

| Rule as implemented | Evidence |
|---|---|
| `issueVoucher` requires `assertActive()` + `assertNotExpired()` | `allotment.aggregate.ts:231-232` |
| Voucher code uniqueness: rejects only if a **same-code, status `ISSUED`** voucher exists in memory | `allotment.aggregate.ts:234-237` (a `USED`/`CANCELLED` duplicate passes; DB unique key `[hotel_id, allotment_id, voucher_code]` would then reject at `schema.prisma:17245`) |
| Per-night quota checks: quota exists → stop-sale not active → `quota − picked ≥ roomsCount` | `allotment.aggregate.ts:242-252` |
| Stop-sale check consults **in-memory** stop-sale entities, not the `stop_sale_active` column | `allotment.aggregate.ts:324-328` (state hydrated from DB at load: `allotment.repository.ts:37-41,54`) |
| Consuming a voucher **does not** re-check quota or stop-sale; it flips status to `USED` | `allotment.aggregate.ts:284-292` |
| `ConsumeAllotmentVoucherHandler` fabricates `reservationId` when not supplied | `consume-allotment-voucher.handler.ts:18` |
| Cancelling a **USED** voucher does **not** restore quota (explicit code comment: "For now, mark as cancelled without restoring quota") | `allotment.aggregate.ts:298-303` |
| Cancelling an **ISSUED** voucher restores `picked` (floor 0) | `allotment.aggregate.ts:304-312` |
| Applying a stop sale flips `stopSaleActive` on matching quota rows **and** writes them twice (aggregate save + `updateQuotaStopSaleFlags`) | `allotment.aggregate.ts:355-367`; `allotment.repository.ts:240-264` |
| Stop-sale rows carry `room_type` nullable (null = all room types) | `coversRoomType` used at `:326`; entity `allotment-stop-sale.entity.ts` |
| Contract type vocabulary: `SOFT_QUOTA / HARD_COMMITMENT / FREE_SALE / GUARANTEED` | `allocation-status.value-object.ts` (`ContractType`) — only `HARD_COMMITMENT` reaches Availability |

### 1.5 Attrition

| Rule as implemented | Evidence |
|---|---|
| `AttritionCalculationService` is **provided and exported** by the module but **never injected into any handler/service** | `group-allotment.module.ts:13,72,121`; grep for `attritionCalculationService` outside module file → 0 hits |
| `attrition_threshold` (default 80) stored on `group_blocks` | `schema.prisma:17095` |
| `create-group-booking.command.ts:25` accepts `attritionThreshold?: number` — persisted, not evaluated | command DTO |
| No attrition report/penalty computation executes | no call sites; `AttritionPolicy` value object exists under `domain/value-objects/` |

### 1.6 Billing / master folio

| Rule as implemented | Evidence |
|---|---|
| Pickup auto-posts ROOM charge = `negotiatedRate × nights` to master folio | `create-group-pickup.handler.ts:64-73` |
| If no master folio exists, charge is skipped with a `warn` (no creation, no failure) | `create-group-pickup.handler.ts:70-72` |
| Commands exist for `post-master-charge`, `record-payment`, `reconcile-room-tax` | `application/commands/{post-master-charge,record-payment,reconcile-room-tax}` |
| Folio windows created per pickup with `folio_type='GRP_MASTER'`, `window_no=1` | `prisma-reservation-association.adapter.ts:127-131` |

### 1.7 Front Office interaction

| Rule as implemented | Evidence |
|---|---|
| Checkout sets `allotment_pickups.status`/`group_pickups.status` = `CHECKED_OUT` via raw SQL in `try/catch` | `check-out.handler.ts:390-392,479-481` |
| Checkout does **not** modify `picked`/`picked_qty`/`contracted_nights`/quota counters | same region — status columns only |
| Failures are logged and swallowed → status can remain `ACTIVE` after checkout | try/catch structure `:389-477` |
| No other FO command writes GBA state (no pickup creation, no block status change) | module grep: only `check-out.handler.ts` |

---

## 2. TARGET Rules (from `docs/design/gba-domain-spec.md` — reference only)

| # | TARGET rule | Spec location |
|---|---|---|
| T-1 | Cut-off date: unpicked rooms after cutoff return to general inventory (single wash event per block) | glossary `:36`; "Group Block Release" `:587-591` |
| T-2 | Rolling release: allotment rooms wash back N days before arrival if not picked; **GUARANTEED_BLOCK cannot wash** | `:37`, `:248-256`, invariant "Guaranteed No Wash" `:1146` |
| T-3 | DEDUCT blocks physically remove rooms from CRS; **CRS is single source of truth; GBA creates no second inventory engine; GBA must notify CRS** on create/cancel/wash and voucher intake, and query CRS availability during pickup | `:930-952`, integration table `:1580-1583` |
| T-4 | Pickup invariants: date containment (check-in ≥ block start, check-out ≤ block end), category containment, rate consistency | `:1128-1136` |
| T-5 | Optimistic concurrency: read version → write with `version = expectedVersion`; reject `CONFLICT`; optionally `SELECT … FOR UPDATE` under contention | `:1215-1227`, `:1541-1562` |
| T-6 | Lifecycle: `→ TENTATIVE → DEFINITE → CLOSED`, `TENTATIVE/DEFINITE → CANCELLED`; allotment `→ ACTIVE → ON_HOLD → ACTIVE → EXPIRED/TERMINATED` | `:1081-1099` |
| T-7 | Attrition: percentage threshold (default 80%), shortfall calculation, penalty on master folio | `:564-566`, `:1522-1530` |
| T-8 | Stop sales apply to **allotment contracts only**; block pickups are not stop-sale restricted | `:628-632`, open question `:1747` |
| T-9 | Association via `block_pickup` / `allotment_voucher_pickup` records rather than bare columns on `reservations` | `:871-900`, `:996-997`, open decision `:1734` |
| T-10 | Events: `CutOffWashExecuted`, `AllotmentWashExecuted`, `DelegatePickedUp`, etc., consumed by CRS/FO/Finance/Channel | `:1315-1345`, `:1580-1611` |
| T-11 | Master folio ownership + BEO as child entity (open questions, unresolved in spec) | `:1716-1747` |

---

## 3. Delta (CURRENT vs TARGET)

| ID | Rule | CURRENT | TARGET | Gap class |
|---|---|---|---|---|
| **B-1** | Cut-off wash (T-1) | `cutoff_date` stored; never read by any rule; no scheduler | automatic wash after cutoff | **unimplemented** |
| **B-2** | Rolling release (T-2) | `release_days_before`/`is_rolling_release` stored; `isWithinReleaseWindow` never called; release is manual | scheduled rolling wash; GUARANTEED blocked from wash | **unimplemented** (dead value object) |
| **B-3** | CRS/availability notification (T-3) | no port implementation; no event consumer; no availability call during pickup | mandatory notify + query | **unimplemented** (dead port) |
| **B-4** | Pickup containment invariants (T-4) | allocation existence + capacity only; no date/status/rate containment | full invariants | **partial** |
| **B-5** | Optimistic concurrency (T-5) | `version` columns exist; written with `{ increment: 1 }`; **never used as a predicate** | conditional update / `FOR UPDATE` | **declared but not enforced** |
| **B-6** | Lifecycle transitions (T-6) | enums + `assertTransition` exist, but `GroupBlock.confirm()/openForPickup()/close()` and `washAllocation()` are **never invoked by any command**; allotment `activate()` runs only at creation (`create-allotment.handler.ts:49`), `suspend()`/`close()` never | explicit transitions incl. ON_HOLD/EXPIRED/TERMINATED + block DEFINITE/OPEN_FOR_PICKUP | **dead code / unreachable** (see §1.1, F-6b, D-15) |
| **B-7** | Attrition (T-7) | service registered but unused; threshold stored; no computation | threshold + shortfall + penalty | **dead code** |
| **B-8** | Stop-sale scope (T-8) | implemented for allotments; blocks unaffected (consistent with target) | allotment-only | **aligned** |
| **B-9** | Association model (T-9) | reservations carry `block_code`, `group_block_id`, `allotment_id`, `allotment_voucher_id`, `pickup_type` **and** pickup tables carry `reservation_id` — both patterns coexist | association tables only | **conflicting (both patterns live)** |
| **B-10** | Event consumption (T-10) | 22 events published; consumer drops them | consumed by CRS/FO/Finance/Channel | **unimplemented** |
| **B-11** | Voucher consumption (T-3/T-9) | `consumeVoucher` does not verify a reservation exists; handler fabricates an id | linked to a real reservation | **violated** |
| **B-12** | Used-voucher cancel (spec silent) | quota not restored (`:298-303`) | undefined in spec | **open** (see `08_OPEN_DECISIONS.md`) |
| **B-13** | Guest resolution (spec silent) | `full_name ILIKE` match — merges guests across same-name records | undefined | **open** |

---

## 4. Summary

- **Implemented and coherent:** intra-block capacity checks, intra-allotment quota + stop-sale checks on *issue*, soft-delete/read scoping, pickup↔reservation linkage via `group_block_id`, folio auto-posting.
- **Implemented but weakened:** concurrency (B-5), lifecycle commands (B-6), containment invariants (B-4), FO write-back (status-only, best-effort).
- **Not implemented (rules exist only as stored fields / dead code):** cutoff wash (B-1), rolling release (B-2), CRS notification (B-3), attrition (B-7), event consumption (B-10).
- **Actively violated:** fabricated voucher reservation id (B-11).

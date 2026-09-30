# Phase 4 — Availability Integration Matrix (GBA)

**Purpose:** a single table of *who reads or writes what inventory state*, with eligibility rules and divergence flagged. CURRENT behavior only.

---

## 1. Direction of Integration

| Direction | Exists? | Evidence |
|---|---|---|
| Availability → GBA (read) | **YES** | `availability-source.adapter.ts:49` (blocks), allotment counterpart; provenance tags `:93` |
| GBA → Availability (write/notify) | **NO** | `IInventoryCommitmentPort` (`group-allotment/domain/ports/inventory-commitment.port.ts`) has zero implementations or DI bindings; grep across `apps/api/src` returns only the interface file |
| GBA events → any consumer | **NO** | `events.consumer.ts:46-57` handles `reservation.*` only; all other event types log `no handler` |
| GBA → assertion engine | **NO** | `availability-assertion.service.ts:64` implements `ReservationAvailabilityPort` for reservation flows; GBA pickup inserts `reservations` rows by raw SQL (`prisma-reservation-association.adapter.ts:100-120`) with no port call |
| GBA → legacy `availability` counters | **NO** | no GBA file references `inventory.domain-service` or the `availability` table |

**Net:** Availability is a *passive reader* of GBA state. Nothing in GBA triggers recalculation, cache invalidation, assertion balance updates, or channel pushes.

---

## 2. Matrix — Inventory State Consumers

| # | Consumer | Entry point | GBA tables read | Block eligibility filter | Allotment eligibility filter | Formula | Notes / divergence |
|---|---|---|---|---|---|---|---|
| A1 | **Availability snapshot** (Phase 1/2 authority) | `availability-source.adapter.ts:49-50` | `group_block_daily_allocations` (+join `group_blocks`, `group_bookings`) | WHERE `inventory_policy='DEDUCT_INVENTORY'` **AND** `deleted_at IS NULL` **AND** `group_bookings.status IN ('CONFIRMED','ACTIVE')` (`:49`); **then** `:66` keeps only `group_blocks.status IN ('DEFINITE','OPEN_FOR_PICKUP')` (other statuses → `:67` "no established eligibility rule" → UNRESOLVED) | WHERE `allotment_contracts.contract_type='HARD_COMMITMENT'`, `status='ACTIVE'`, `deleted_at IS NULL`, `validity_start ≤ day ≤ validity_end` (`:50`) | `:76` `gbaRemaining = contracted − picked − released`; `:82` `allotmentRemaining = quota − picked − released` | `snapshot-calculator.ts`: `consumption = reservationConsumption + gbaRemaining + allotmentRemaining`; integrity checks at `:68` and `:83-84` mark rows UNRESOLVED instead of trusting them |
| A2 | **Snapshot unresolved-source report** | `availability-snapshot.service.ts:83` | same as A1 | same | same | emits `group_block_daily_allocations` as unresolved fact source when `gbaStatus==='UNRESOLVED'` | fail-soft behavior |
| A3 | **Activities availability matrix** | `availability-sales.controller.ts:237` (`GET availability/matrix`) | `:306-322` `group_block_daily_allocations` JOIN `group_blocks`; `:325-335` `allotment_daily_quotas` JOIN `allotment_contracts` | `gb.status NOT IN ('CANCELLED','CLOSED')` + `inventory_policy='DEDUCT_INVENTORY'` — **no `deleted_at`, no booking status, block status not whitelisted** | `ac.status IN ('ACTIVE','CONFIRMED')` — **no `contract_type` filter**, `'CONFIRMED'` not a valid `AllotmentStatus` value | `:459` `available = max(0, physical − ooo − reserved − groupCommit − allotCommit)` where `groupCommit/allotCommit = SUM(qty − picked − released)` | **Diverges from A1 on every filter dimension** (see §3) |
| A4 | **Legacy counters** | `inventory.domain-service.ts` via `crs-engine.service.ts:345,391,531-542,589,595` and `reservation.repository.ts:625` | legacy `availability` table (`schema.prisma:433`) | n/a — operates on `availability.alloted/reserved/available` counters | n/a | `INSERT/UPDATE availability … reserved ± 1` (`:164,181,227,256,275`) | GBA never participates; block-related functions `blockAvailability` `:208`, `consumePickup` `:237`, `releaseUnsold` `:266` are **never called** |
| A5 | **Assertion engine** (Phase 2 authority) | `availability-assertion.service.ts:64` (`implements ReservationAvailabilityPort`) | `availability_assertion_balances` (`schema.prisma:17310`), journal/state | reservation-driven only | reservation-driven only | assert / release / replace / journal | **Pickup-created reservations bypass it** (raw INSERT, no port call) → balances never see GBA-origin rows |
| A6 | **Reservation consumption** | `reservation-consumption.adapter.ts` (Phase 1) | `reservations` per stay date | counts reservations irrespective of `group_block_id`/`pickup_type` | same | `reservationConsumption` term of A1 | picked rooms are counted here while *unpicked* rooms are counted as `gbaRemaining` — no double count **for A1** |
| A7 | **FO checkout write-back** | `check-out.handler.ts:390-392,479-481` | `allotment_pickups`, `group_pickups` | sets `status='CHECKED_OUT'` | same | status only | does not touch `picked`/`picked_qty`/`contracted_nights`; wrapped in try/catch (best-effort) |
| A8 | **GBA write path (pickup)** | `create-group-pickup.handler.ts` / `create-allotment-pickup.handler.ts` | `group_block_daily_allocations` via aggregate `canPickup`; `allotment_daily_quotas` via `quota − picked` | aggregate-level capacity check only (`group-block.aggregate.ts:234-242`) | `allotment.aggregate.ts:242-252` incl. stop-sale check | in-memory mutation, then `repo.save` | never consults A1/A4/A5 — no transient-room availability check |
| A9 | **GBA release (manual)** | `release-block-allocation.handler.ts:26-33`, `release-allotment-allocation.handler.ts:26-33` | same allocations/quotas | caller-supplied date+quantity | caller-supplied date+quantity | aggregate `releaseAllocation` | no cutoff/rolling-window gate (§4) |

---

## 3. Divergence Detail — A1 (Availability) vs A3 (Activities matrix)

Both read `group_block_daily_allocations` and `allotment_daily_quotas` for the same hotel/date/room-type and produce a *sellable* number, with these rule differences:

| Dimension | A1 `availability-source.adapter.ts` (`:49-50` WHERE, `:66` eligibility) | A3 `availability-sales.controller.ts:306-335` | Net effect on A3 |
|---|---|---|---|
| Block deleted | `deleted_at: null` (`:49`) | **not filtered** | soft-deleted blocks still consume |
| Booking status | `group_bookings.status IN ('CONFIRMED','ACTIVE')` (`:49`) | **not filtered** | PROSPECT bookings consume |
| Block status | `IN ('DEFINITE','OPEN_FOR_PICKUP')` (`:66`) | `NOT IN ('CANCELLED','CLOSED')` → **DRAFT, TENTATIVE count** | unconfirmed blocks consume |
| Allotment contract type | `HARD_COMMITMENT` only (`:50`) | **all types** → SOFT_QUOTA, FREE_SALE, GUARANTEED all consume | soft/free allotments suppress transient sell |
| Allotment status | `ACTIVE` only (`:50`) | `('ACTIVE','CONFIRMED')` — `CONFIRMED` not in value object (`allocation-status.value-object.ts`) | expression is wrong, though practically ≈ ACTIVE |
| Allotment validity window | `validity_start ≤ day ≤ validity_end` (`:50`) | **not filtered** | expired contracts keep consuming |
| Counter integrity | inconsistent rows → `UNRESOLVED`, excluded from trusted totals (`:68-71`) | no integrity check | A3 trusts corrupt rows |
| Reservation term | `reservationConsumption` from A6 | raw reservation counts `:377-385` + proportional distribution of unassigned `:446-451` | different reservation counting |
| OOO rooms | not part of `consumption` | subtracted (`:444,459`) | A3 stricter |
| Restrictions (CTA/CTD/ZeroSell) | separate restriction pipeline | applied in-line (`:461-468`) | A3 conflates |

**Result:** the same hotel/date can legitimately show two different "available" numbers depending on which endpoint the UI calls. Which endpoint each page uses is tracked in `04_AVAILABILITY_INTEGRATION_MATRIX.md` §5 (frontend consumers).

---

## 4. GBA → Availability Notification Gap (spec vs current)

`docs/design/gba-domain-spec.md` §"CRS Integration" states GBA must notify CRS on block created/cancelled/washed and voucher intake, and must query CRS availability during pickup (`gba-domain-spec.md:930-952, 1580-1583`).

| Spec expectation | CURRENT implementation | Evidence |
|---|---|---|
| Notify CRS on block create/cancel/wash | event emitted, **no consumer** | `GroupBlockCreatedEvent` etc. (`group-allotment.events.ts:57,155,173`) → `events.consumer.ts:57` default warn |
| Query CRS availability during pickup | **not done** — only intra-block capacity check | `group-block.aggregate.ts:234-242`; no Availability port import in `create-group-pickup.handler.ts` |
| Availability decremented for DEDUCT blocks | **indirect only** — Availability subtracts `gbaRemaining` at read time (A1), so there is no write-time decrement | `snapshot-calculator.ts` |
| Release/wash returns rooms | **manual release only**; no scheduled wash; `ReleaseWindow.isWithinReleaseWindow` never called | `release-window.value-object.ts:26` (0 call sites) |
| Cut-off wash after `cutoff_date` | `cutoff_date` persisted (`group-block.repository.ts:115,127`) but **never read by any rule** | grep: `cutoffDate` appears only in aggregate field/getter/toJSON and create handlers |
| Rolling release N days before arrival | `release_days_before` persisted (`allotment.repository.ts:127,145`) but **never evaluated** | §5 |

**Read-time subtraction (A1) means a block's held rooms disappear from sellable inventory only while the block is in an eligible status.** A block that transitions to `CANCELLED`/`CLOSED` (or is soft-deleted) immediately releases its consumption from the snapshot — with no event, no journal entry, and no assertion-balance adjustment.

**Asymmetry (blocks never eligible today):** A1 counts only `DEFINITE`/`OPEN_FOR_PICKUP` blocks (`:66`), yet **no command can set those statuses** — `GroupBlock.confirm()/openForPickup()/close()` have zero call sites (`group-block.aggregate.ts:131,138,144`), and new blocks are created `DRAFT` (`schema.prisma:17085`). Blocks created through the API therefore contribute **0** consumption to A1 until status changes outside the command set. Allotments behave differently: `CreateAllotmentHandler` calls `allotment.activate()` (`create-allotment.handler.ts:49`), so contracts are `ACTIVE` at creation and count immediately.

---

## 5. Frontend Consumer Map (which screen hits which computation)

| Screen / route | API called | Backed by |
|---|---|---|
| `apps/web/app/(dashboard)/reservations/group-blocks/**`, `…/allotments/**`, `features/group-allotment/**` | `group-allotment.api.ts` → `/group-bookings/*`, `/allotments/*` | GBA module (A8/A9) |
| Availability page (`reservations/availability`) | availability endpoints → snapshot | A1/A5/A6 |
| Activities availability matrix / restriction rows | `GET availability/matrix`, `GET availability/restriction-rows` | A3 (matrix), restrictions (rows) |
| Group booking "available rooms" picker | `GET /group-bookings/available-rooms` (`group-booking.controller.ts:95-116`) | **neither** — filters `rooms WHERE room_status='AVAILABLE'` with no date/reservation logic; `hotelId || 'default'` (`:103`) |

The last row is a third, undocumented "availability" computation: it reports currently-vacant rooms regardless of stay dates, and is exposed to the group-booking form (`features/group-allotment/api/group-allotment.api.ts:29`).

---

## 6. Integration Verdict

1. **Integration is read-only and asynchronous-free:** A1 reads GBA state at query time; there is no write-path coupling in either direction (§1).
2. **Two authoritative "available" numbers exist** (A1 and A3) with incompatible eligibility rules (§3).
3. **The GBA→CRS notification contract from the design spec is entirely unimplemented** (§4), and the cut-off/rolling-release rules that would justify time-based release have no execution path.
4. **Pickup-created reservations are invisible to the assertion engine** (A5), so Phase 2 balance tooling will treat GBA-origin reservations as unbalanced input.

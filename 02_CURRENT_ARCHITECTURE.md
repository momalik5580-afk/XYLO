# Phase 4 — Current Architecture (GBA / Allotment)

**Rule applied:** every flow below is derived from executed code with `file:line` evidence. TARGET behavior from `docs/design/gba-domain-spec.md` is not mixed in (it appears only in `05_BUSINESS_RULES_CURRENT_STATE.md`).

---

## 1. Module Topology

```
HTTP  group-booking.controller.ts (@Controller('group-bookings'), @PropertyScope(true), @Permission)
HTTP  allotment.controller.ts     (@Controller('allotments'))
        │  CommandBus / QueryBus (manual handler registration in group-allotment.module.ts)
        ▼
25 CommandHandlers ──► domain aggregates (GroupBlock, Allotment, GroupBooking)
        │                    │  in-memory state machine + 22 IntegrationEvents
        │                    ▼
        │             repository.save()  ──► raw Prisma upserts / $executeRawUnsafe
        │
        ├──► IReservationAssociationPort ──► prisma-reservation-association.adapter.ts (raw INSERT INTO reservations)
        ├──► FolioPort                    ──► master-folio raw SQL
        └──► EventBus.publishFromAggregate ──► outbox_messages ─(10s cron)─► BullMQ 'events'
                                                              ──► events.consumer.ts → "no handler" (warn)

Availability (read-only direction) ──► availability-source.adapter.ts ──► group_block_daily_allocations
                                                                    └──► allotment_daily_quotas
```

**Direction of coupling:** Availability reads GBA tables; **GBA never calls Availability.** The only reference to availability inside the module is `domain/ports/inventory-commitment.port.ts:19` — an `IInventoryCommitmentPort` interface with **no implementation and no binding anywhere in the repo** (grep for the symbol returns only its own declaration).

---

## 2. Command Flows (CURRENT)

### 2.1 Create group booking → create block

`POST /group-bookings` → `CreateGroupBookingHandler` → `bookingRepo.save()`.
`POST /group-bookings/:id/blocks` → `CreateGroupBlockHandler` (`create-group-block.handler.ts`):

1. `:26` `hotelId = getHotelId() || command.hotelId || 'default'`
2. `:27` load booking (`bookingRepo.findById`)
3. `:30` `blockCode = await blockRepo.nextBlockCode(hotelId, booking.bookingCode)`
4. `:32` `GroupBlock.create({…})` (aggregate) + optional `setDailyAllocation()` loop `:45-54`
5. `:56` `blockRepo.save(block)` → `$transaction` (block + allocations + pickups upserts, `group-block.repository.ts:103-202`)
6. `:57` `eventBus.publishFromAggregate(block.domainEvents)`

**Code generation:** `GroupBlockRepository.nextBlockCode` counts rows `WHERE group_booking_id = <bookingCode>` (`group-block.repository.ts:212-216`) then returns `` `${bookingCode}-${String.fromCharCode(65 + (count % 26))}` `` (`:215-216`). The column `group_blocks.group_booking_id` is a **UUID FK** (`schema.prisma:17103`) while the caller passes `booking.bookingCode` (a human code) — so the count is against a non-matching value; the suffix therefore derives from a query that does not match its own rows.

### 2.2 Pickup (the critical path)

`POST /group-bookings/:bookingId/blocks/:blockId/pickups` → `CreateGroupPickupHandler` (`create-group-pickup.handler.ts`):

| Step | Code | Transactional? |
|---|---|---|
| 1. Load block (3 separate queries: block, allocations, pickups) | `group-block.repository.ts:22-46` | read |
| 2. `block.createPickup(…)` — in-memory capacity check `allocation.canPickup()` then mutate counters | `group-block.aggregate.ts:214-279` (checks at `:234-242`, mutate at `:244-247`) | in memory |
| 3. `reservationAssoc.createPickupReservation(…)` — raw SQL guest lookup/create, `INSERT INTO reservations`, folio + routing inserts | `prisma-reservation-association.adapter.ts:61-131+` | **no** (`$queryRawUnsafe`/`$executeRawUnsafe` sequence) |
| 4. On reservation success: `pickup.linkReservation(reservationId)` | handler `:58` | — |
| 5. Auto-post room charge to master folio (`folioPort.postFolioCharge`) | handler `:64-73` | **separate write** |
| 6. `repo.save(block)` (state → DB) | handler `:80` → `group-block.repository.ts:100-203` | own `$transaction` |
| 7. `eventBus.publishFromAggregate` | handler `:81` | — |

Reservation columns written (`prisma-reservation-association.adapter.ts:100-120`): `group_block_id = <block UUID>`, `pickup_type = 'group_pickup'`, `reservation_status = 'CONFIRMED'`, `source_code/market_code = 'GROUP'`, `rate_code = block.ratePlanCode || 'BAR'`, `room_rate = block.negotiatedRate || 0`.

**Authoritative pickup link = `reservations.group_block_id` → `group_blocks.id`.** `reservations.block_code` is *not* written by this path (the DTO accepts it — `create-reservation.dto.ts:26` — but `create-reservation.handler` never uses it; the value is persisted only by `reservation-persistence.domain-service.ts:72,81` when supplied).

**Failure semantics:** if step 6 fails after step 3, a reservation exists pointing at a block whose counters were never incremented (reservation created, `picked_qty` unchanged). If step 3 fails, the handler throws at `:74-78` and no state is written — clean. Steps 3, 5, 6 are three independent write units with no compensating action.

### 2.3 Allotment voucher vs allotment pickup (two parallel mechanisms)

**Voucher path (aggregate-driven):** `POST /allotments/:id/vouchers` → `CreateAllotmentVoucherHandler` → `allotment.issueVoucher()` (`allotment.aggregate.ts:220-282`):
- `:231` `assertActive()`, `:232` `assertNotExpired()`
- `:242-252` per-night: quota exists → `isStopSaleActive(date, roomType)` → `remaining = quota.quota - quota.picked >= roomsCount`
- `:254-257` mutate `quota.picked += roomsCount`
- `repo.save(allotment)` → `allotment.repository.ts:109-231` (no `$transaction`)

`POST /allotments/:id/vouchers/:vid/consume` → `ConsumeAllotmentVoucherHandler`:
```ts
const reservationId = command.data.reservationId || `RES-${Date.now()}`;   // :18
allotment.consumeVoucher(command.data.voucherId, reservationId);            // :19
```
If the caller omits `reservationId`, a **fabricated id** (`RES-<epoch ms>`) is stored on the voucher — no reservation is created, no link verified.

**Pickup path (raw-SQL-driven):** `POST /allotments/:id/pickups` → `CreateAllotmentPickupCommand` → `create-allotment-pickup.handler.ts`:
- `:47` SELECT from **`allotment_pickups`** (table absent from schema.prisma + migrations)
- `:130` `INSERT INTO allotment_pickups`
- separate reservation insert (same association adapter), separate quota update
- `cancel-allotment-pickup.handler.ts:66,89-94` — final `UPDATE allotment_pickups` and `UPDATE reservations` carry **no `hotel_id` predicate**

**Both paths are reachable from the UI** (`apps/web/features/group-allotment/api/group-allotment.api.ts:288` vouchers, `:345` pickups).

### 2.4 Stop sale (apply / lift)

`apply-allotment-stop-sale.handler.ts` and `lift-allotment-stop-sale.handler.ts` each perform:
1. aggregate mutation (`allotment.applyStopSale` / `.liftStopSale`, `allotment.aggregate.ts:330-400`) which also flips `quota.stopSaleActive` in memory;
2. `repo.save(allotment)` — quota rows updated via raw upsert that **does** write `stop_sale_active` (`allotment.repository.ts:226`);
3. **plus** a second, separate raw `UPDATE allotment_daily_quotas … version = version + 1` (`allotment.repository.ts:240-264`, `updateQuotaStopSaleFlags`) with **string-interpolated SQL built from escaped fragments** (`:246-261`) rather than bound parameters.

Two writes to the same rows, no transaction wrapping them together, no idempotency key.

### 2.5 Release / wash (manual only)

- `POST /group-bookings/:bookingId/blocks/:blockId/release` → `ReleaseBlockAllocationHandler` (`release-block-allocation.handler.ts:26-33`) → `block.releaseAllocation(stayDate, roomType, quantity)` — **caller supplies date and quantity; no cutoff-date comparison, no status gate in the handler.**
- `POST /allotments/:id/release` → `ReleaseAllotmentAllocationHandler` (`release-allotment-allocation.handler.ts:26-33`) → `allotment.releaseAllocation(...)` — same shape.

No scheduler invokes these. Repo-wide: `ReleaseWindow.isWithinReleaseWindow` (`release-window.value-object.ts:26`) has **zero call sites** (only the class's own construction inside `allotment.aggregate.ts:517,586`). There is **no cron/job** in the module (the only cron in scope is the generic outbox processor, `outbox-processor.ts:12,22,32`).

### 2.6 Front Office write-back (the only GBA write outside the module)

`check-out.handler.ts`:
- `:390-392` `UPDATE allotment_pickups SET status='CHECKED_OUT' … WHERE …` inside `try/catch` (best-effort)
- `:479-481` `UPDATE group_pickups SET status='CHECKED_OUT' … WHERE …` likewise

Neither statement changes quota/counter columns (`picked`, `picked_qty`, `contracted_nights`) — checkout changes status only. Failures are swallowed (logged), so pickup rows can remain `ACTIVE` after checkout.

No Front Office command calls `block.createPickup`, `allotment.issueVoucher`, or any GBA command handler.

---

## 3. Read Flows

### 3.1 Availability (Phase 1/2 authority)

`availability-source.adapter.ts`:
- `:49` `group_block_daily_allocations.findMany` where `hotel_id`, `room_type`, `stay_date`, `group_blocks: { inventory_policy: 'DEDUCT_INVENTORY', deleted_at: null, group_bookings: { status: { in: ['CONFIRMED','ACTIVE'] } } }`
- counterpart query for `allotment_daily_quotas` filtered by `contract_type = 'HARD_COMMITMENT'`, `status = 'ACTIVE'`, within validity window
- `:93` provenance tagging: `source: 'group_block_daily_allocations' | 'allotment_daily_quotas'`
- exposes `gbaHeld/gbaPicked/gbaReleased/gbaRemaining`, `allotment*`, `allotmentStopSaleActive`

`snapshot-calculator.ts`:
```
consumption = reservationConsumption + gbaRemaining + allotmentRemaining
gbaRemaining      = contracted - picked - released   (held semantics per DailyRoomAllocation)
allotmentRemaining = quota - picked - released        (HARD_COMMITMENT only)
```

Unresolved sources are surfaced by `availability-snapshot.service.ts:83` (`facts.gbaStatus === 'UNRESOLVED' → ['group_block_daily_allocations']`).

### 3.2 Activities availability matrix (second, divergent reader)

`availability-sales.controller.ts` `GET availability/matrix` (`:237`):

- `:306-322` group commitment: `FROM group_block_daily_allocations gba JOIN group_blocks gb … WHERE … AND gb.status NOT IN ('CANCELLED','CLOSED') AND gba.inventory_policy = 'DEDUCT_INVENTORY'` — **no booking-status filter, no `deleted_at` filter**
- `:325-335` allotment commitment: `SUM(adq.quota - adq.picked - adq.released) … JOIN allotment_contracts ac … WHERE ac.status IN ('ACTIVE','CONFIRMED')` — **no `contract_type` filter** (SOFT/FREE_SALE count), and `CONFIRMED` is not a value in the `AllotmentStatus` value object
- `:453-459` `available = max(0, physical - ooo - reserved - groupCommit - allotCommit)`

`GET availability/restriction-rows` (`:798`) reads only restriction/sell-limit tables — no GBA data.

### 3.3 Frontend

`apps/web/features/group-allotment/api/group-allotment.api.ts` mirrors both controllers (list at `:23-376`); `use-group-allotment.ts` wraps React Query (`:32` `queryKey: ['group-bookings', params]`); views under `apps/web/features/group-allotment/views/` and routes under `apps/web/app/(dashboard)/reservations/{group-blocks,allotments}/`.

**Contract defect:** `releaseAllocation` issues `api.put('/group-bookings/…/release')` (`group-allotment.api.ts:146`) while the controller declares `@Post(':bookingId/blocks/:blockId/release')` (`group-booking.controller.ts:247`) → HTTP verb mismatch (405) for the block-release action. (The allotment release call at `:322` uses `POST` and matches.)

---

## 4. Event Flow

1. Aggregate `addDomainEvent(...)` — 22 `IntegrationEvent`s (`domain/events/group-allotment.events.ts:3-396`).
2. `EventBus.publishFromAggregate` → outbox row (`outbox_messages`).
3. `OutboxProcessor` cron every 10s (`outbox-processor.ts:12`) → `OutboxPublisher.processPendingMessages` → BullMQ queue `events` with `jobId = message.id` (`outbox-publisher.ts:36-40`), idempotency mark at `:42`.
4. `EventsConsumer` (`@Processor('events')`, `events.consumer.ts:21`) switch at `:46-57`:
   - `reservation.created` → `handleReservationCreated`
   - `reservation.updated|cancelled|no_show|deleted` → `invalidateAvailability`
   - **`default` → `logger.warn('no handler for …')`**

**Consequence:** all 22 GBA events are durably persisted, published, consumed, marked processed, and then discarded with a warning. No subsystem (Availability, Channels, Analytics, Finance) reacts to GBA state changes.

---

## 5. Tenancy & Authorization Context

| Mechanism | Behavior | Evidence |
|---|---|---|
| Request hotel id | `getHotelId()` from request context; **every** GBA handler falls back to `command.hotelId` then `'default'` | e.g. `create-group-block.handler.ts:26`, `set-allotment-quota.handler.ts:22`, `consume-allotment-voucher.handler.ts:15` |
| Controller scope | `@PropertyScope(true)` on `group-booking.controller.ts:32` | — |
| Header override | mismatched `x-property-id` **overrides** the authenticated user's hotel id | `tenant.interceptor.ts:19-20`, `property-scope.guard.ts:35` |
| Development bypass | `PropertyAccessGuard` — "Property check disabled for development" | `resource-access.guard.ts:29` |
| Permissions | `@Permission(GROUP_BOOKING_PERMISSIONS.*)` on all controller routes | `group-booking.controller.ts:57-297`, `allotment.controller.ts` |
| Query scoping | repositories always filter `hotel_id` (`group-block.repository.ts:23,30,36`; `allotment.repository.ts:24,30,36`) | — |
| Query scoping exceptions | `cancel-allotment-pickup.handler.ts:66,89-94` and `linkReservation` UPDATE omit `hotel_id` | see `06_CONCURRENCY_AND_TRANSACTION_AUDIT.md` |

---

## 6. What Is NOT Present (architecture-level gaps, evidence only)

| Expected by spec / by Availability | Current state |
|---|---|
| GBA → Availability commitment port (`IInventoryCommitmentPort`) | interface only; zero implementations/bindings (`domain/ports/inventory-commitment.port.ts`) |
| Scheduled cut-off wash / rolling release | none; `ReleaseWindow.isWithinReleaseWindow` uncalled |
| Consumer for GBA events | `events.consumer.ts:46-57` drops them |
| Transactional pickup (reservation + counters) | 3+ independent writes (`create-group-pickup.handler.ts:44,64,80`) |
| Tests for 25 commands | 0 spec files in module |
| `allotment_pickups` schema artifact | raw SQL only; absent from schema.prisma and all 49 migrations |
